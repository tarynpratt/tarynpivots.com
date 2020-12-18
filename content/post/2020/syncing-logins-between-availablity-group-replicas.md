---
title: Syncing Logins Between Availability Group Replicas
date: 2020-12-18 04:00:00 +0000
draft: false
tags: [sql, sql-server, stack-overflow, sql-server-2019, maintenance, distributed-availability-group, availability-group, readable-secondary]
excerpt: Keeping logins in sync when using availablity groups and readable secondary replicas.
---

I mentioned this in a few previous posts, but for for those who may have missed it or forgotten, here’s a quick refresher -  we use [Always On Availability Groups](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server?view=sql-server-ver15) at [Stack Overflow](https://stackoverflow.com) on all of our main production servers running the network of public Q&amp;A sites, Jobs, and [Stack Overflow for Teams](https://stackoverflow.com/teams). It's a great way to implement disaster recovery for a SQL Server environment. 

Always On Availability Groups can support up to nine availability replicas, and while we don’t use anywhere near that many replicas in each of our clusters, we do have 2 replicas per cluster (3 servers total), with the replicas being used as a [readable secondary](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/active-secondaries-readable-secondary-replicas-always-on-availability-groups?view=sql-server-ver15). 

Since we use readable secondaries in our environments, the application needs to connect to both the primary and the secondary servers with the same login. The catch is, logins don’t automatically sync across replicas.  If the logins don’t sync, the application won’t connect to a secondary, which results in login failures.

## Copying Logins with a T-SQL Script 

Before you jump the gun and start criticizing my method, know that I’m aware there are other ways to do this, including using <a href="https://dbatools.io/keeping-availability-group-logins-in-sync-automatically/" target="_blank">dbatools</a>, but this method has been around since the early days of Stack Overflow and the Stack Exchange network (with some <a href="https://www.tarynpivots.com/post/system-view-gotcha-with-sql-server-2019/" target="_blank">modifications</a>) and it works for our needs for now.

We don’t create new databases very often, but on [Stack Overflow for Teams](https://stackoverflow.com/teams) we use schemas to keep customers separated from each other - i.e. every team has a login and user to allow it to query their data. While we pre-provision a set number of schemas, tables, and other database objects for teams, we want the logins to sync on a frequent basis to avoid login failures on the secondaries.  We have a job in place which syncs the logins via a SQL Server Agent job every 10 minutes for Stack Overflow for Teams. (We have a similar job for the public Q&A sites that runs nightly.)

The job needs to do several things: 

1. Get a list of the existing logins on the primary 
2. Create the logins on replica
3. Drop any logins no longer being used i.e. teams that are deleted 

### Gather Existing Logins

The first thing the job does is, create a temp table called `@logins` to store all the details needed from the primary. The table is pretty basic:

{{< highlight sql>}}
DECLARE @logins TABLE 
(
    SID varbinary(256), 
    Login sysname, 
    SQL varchar (1024), 
    DefaultDB sysname
);
{{< / highlight >}}

The table is populated by connecting to the primary server using <a href="https://docs.microsoft.com/en-us/sql/t-sql/functions/openquery-transact-sql?view=sql-server-ver15" target="_blank">`OPENQUERY`</a> (Eeek a linked server) 

{{< highlight sql>}}
SELECT *
FROM OPENQUERY([SQL-AG], '
    SELECT p.sid sid_varbinary, p.name, p.default_database_name,
        sl.password_hash pwd_varbinary,
        CASE sl.is_policy_checked WHEN 1 THEN ''ON'' WHEN 0 THEN ''OFF'' ELSE NULL END is_policy_checked,
        CASE sl.is_expiration_checked WHEN 1 THEN ''ON'' WHEN 0 THEN ''OFF'' ELSE NULL END is_expiration_checked
    FROM sys.server_principals p
    JOIN sys.syslogins l ON l.name = p.name
    JOIN sys.sql_logins sl ON l.name = sl.name
    WHERE p.type = ''S''
      AND p.name <> ''sa''
      AND l.denylogin = 0
      AND l.hasaccess = 1
      AND p.is_disabled = 0
    ORDER BY p.name');
{{< / highlight >}}

and executing <a href="https://docs.microsoft.com/en-us/troubleshoot/sql/security/transfer-logins-passwords-between-instances" target="_blank">sp_hexadecimal</a> to get the SID and password which are then used to create a dynamic SQL string which is stored in the temp table.

{{< highlight sql>}}
EXEC sp_hexadecimal @PWD_varbinary, @PWD_string OUT
EXEC sp_hexadecimal @SID_varbinary, @SID_string OUT

SET @sqlString = 'CREATE LOGIN ' + QUOTENAME(@name) + ' WITH PASSWORD = ' + @PWD_string 
                    + ' HASHED, SID = ' + @SID_string + ', DEFAULT_DATABASE = [' + @defaultdb + ']'
                    + ', CHECK_POLICY = ' + @is_policy_checked
                    + ', CHECK_EXPIRATION = ' + @is_expiration_checked

INSERT INTO @logins (SID, Login, SQL, DefaultDB) 
VALUES (@SID_varbinary, @name, @sqlString, @defaultdb);
{{< / highlight >}}

### Create Logins on Replicas

Now that the table has all the logins from the primary, it’s time to create them locally. While it’s possible that we have only one login to sync, it’s more likely we have multiple.  Keeping this in mind, the script loops through the temp table and executes the dynamic SQL for each login that doesn’t already exist on the replica.

{{< highlight sql>}}
DECLARE @sql varchar (1024), @db sysname, @loginname sysname;
DECLARE login_cursor CURSOR FOR
    SELECT SQL, DefaultDB, Login
    FROM @logins
    WHERE SID NOT IN (SELECT sid 
                        FROM sys.server_principals)
        AND EXISTS (SELECT 1 
                    FROM sys.databases 
                    WHERE name = DefaultDB)
        AND Login NOT IN (SELECT name 
                            FROM sys.server_principals 
                            WHERE name = Login 
                                And type = 'S')
OPEN login_cursor

-- add new logins
FETCH NEXT FROM login_cursor INTO @sql, @db, @loginname
WHILE @@fetch_status = 0
BEGIN
    EXEC(@sql);
    FETCH NEXT FROM login_cursor 
        INTO @sql, @db, @loginname
END
CLOSE login_cursor
DEALLOCATE login_cursor
{{< / highlight >}}

### Drop Unused Logins on Replicas

On the public Q&A sites, like Stack Overflow, we don’t need to drop logins, but after running Stack Overflow for Teams for several months, we realized very quickly that we need to purge logins no longer being used. The last step of the job goes through the existing logins on the secondaries and deletes any that don’t exist on the primary. Using the original `@logins` temp table I can get a list of the logins to drop:

{{< highlight sql>}}
SELECT p.sid, p.name, p.default_database_name
FROM sys.server_principals p
JOIN sys.syslogins l ON l.name = p.name
JOIN sys.sql_logins sl ON l.name = sl.name
WHERE p.type = 'S'
  AND p.name <> 'sa'
  AND l.denylogin = 0
  AND l.hasaccess = 1
  AND p.is_disabled = 0
  AND NOT EXISTS (SELECT 1
                  FROM @logins cl
                  WHERE p.sid = cl.sid
                    AND p.name = cl.Login
                    AND p.default_database_name = cl.DefaultDB)
ORDER BY p.name
{{< / highlight >}}

then it creates a dynamic SQL string which gets executed cleaning up the unused logins. 

{{< highlight sql>}}
SET @drop_sqlString = 'DROP LOGIN ' + QUOTENAME(@name) + '; '
--print @drop_sqlString
EXEC(@drop_sqlString);
{{< / highlight >}}


### Putting the Pieces Together

Both the create and delete of the logins are done in various loops, in other words cursors. The full script is below:

{{< highlight sql>}}
DECLARE @name sysname,
    @PWD_varbinary varbinary (256),
    @PWD_string varchar (514),
    @SID_varbinary varbinary (85),
    @SID_string varchar (514),
    @sqlString varchar (1024),
    @is_policy_checked varchar (3),
    @is_expiration_checked varchar (3),
    @defaultdb sysname;

DECLARE @logins TABLE 
(
    SID varbinary(256), 
    Login sysname, 
    SQL varchar (1024), 
    DefaultDB sysname
);

DECLARE login_cursor CURSOR FOR
    SELECT *
    FROM OPENQUERY([SQL-AG], '
        SELECT p.sid, p.name, p.default_database_name,
            sl.password_hash pwd_varbinary,
            CASE sl.is_policy_checked WHEN 1 THEN ''ON'' WHEN 0 THEN ''OFF'' ELSE NULL END is_policy_checked,
            CASE sl.is_expiration_checked WHEN 1 THEN ''ON'' WHEN 0 THEN ''OFF'' ELSE NULL END is_expiration_checked
        FROM sys.server_principals p
            JOIN sys.syslogins l ON l.name = p.name
            JOIN sys.sql_logins sl ON l.name = sl.name
        WHERE p.type = ''S''
        AND p.name <> ''sa''
        AND l.denylogin = 0
        AND l.hasaccess = 1
        AND p.is_disabled = 0
        ORDER BY p.name')
OPEN login_cursor

FETCH NEXT FROM login_cursor 
    INTO @SID_varbinary, @name, @defaultdb, @PWD_varbinary, @is_policy_checked, @is_expiration_checked
WHILE @@fetch_status = 0
BEGIN
    EXEC sp_hexadecimal @PWD_varbinary, @PWD_string OUT
    EXEC sp_hexadecimal @SID_varbinary, @SID_string OUT

    SET @sqlString = 'CREATE LOGIN ' + QUOTENAME(@name) + ' WITH PASSWORD = ' + @PWD_string 
                        + ' HASHED, SID = ' + @SID_string + ', DEFAULT_DATABASE = [' + @defaultdb + ']'
                        + ', CHECK_POLICY = ' + @is_policy_checked
                        + ', CHECK_EXPIRATION = ' + @is_expiration_checked

    INSERT INTO @logins (SID, Login, SQL, DefaultDB) 
    VALUES (@SID_varbinary, @name, @sqlString, @defaultdb);

    FETCH NEXT FROM login_cursor 
        INTO @SID_varbinary, @name, @defaultdb, @PWD_varbinary, @is_policy_checked, @is_expiration_checked
END
CLOSE login_cursor
DEALLOCATE login_cursor

DECLARE @sql varchar (1024), @db sysname, @loginname sysname;
DECLARE login_cursor CURSOR FOR
    SELECT SQL, DefaultDB, Login
    FROM @logins
    WHERE SID NOT IN (SELECT sid 
                        FROM sys.server_principals)
        AND EXISTS (SELECT 1 
                    FROM sys.databases 
                    WHERE name = DefaultDB)
        AND Login NOT IN (SELECT name 
                            FROM sys.server_principals 
                            WHERE name = Login 
                                And type = 'S')
OPEN login_cursor

-- add new logins
FETCH NEXT FROM login_cursor INTO @sql, @db, @loginname
WHILE @@fetch_status = 0
BEGIN
    EXEC(@sql);
    --print @sql
    FETCH NEXT FROM login_cursor 
        INTO @sql, @db, @loginname
END
CLOSE login_cursor
DEALLOCATE login_cursor

-- drop logins that are no longer being used
DECLARE @drop_sqlString nvarchar(max) = '';
DECLARE drop_login_cursor CURSOR FOR
    -- logins on current server that don't exist on the primary
    SELECT p.sid, p.name, p.default_database_name
    FROM sys.server_principals p
    JOIN sys.syslogins l ON l.name = p.name
    JOIN sys.sql_logins sl ON l.name = sl.name
    WHERE p.type = 'S'
       AND p.name <> 'sa'
       AND l.denylogin = 0
       AND l.hasaccess = 1
       AND p.is_disabled = 0
       AND NOT EXISTS (SELECT 1
                        FROM @logins cl
                        WHERE p.sid = cl.sid
                          AND p.name = cl.Login
                          AND p.default_database_name = cl.DefaultDB)
	ORDER BY p.name
OPEN drop_login_cursor
FETCH NEXT FROM drop_login_cursor INTO @SID_varbinary, @name, @defaultdb
WHILE @@fetch_status = 0
BEGIN

    SET @drop_sqlString = 'DROP LOGIN ' + QUOTENAME(@name) + '; '
	--print @drop_sqlString
	EXEC(@drop_sqlString);
	
    FETCH NEXT FROM drop_login_cursor INTO @SID_varbinary, @name, @defaultdb
END
CLOSE drop_login_cursor
DEALLOCATE drop_login_cursor
{{< / highlight >}}


As I said, this might not be the way you would do this, but this works for us and keeps the logins synced across all of our replicas which minimizes login failures. I’ve uploaded the <a href="https://github.com/tarynpratt/misc_sql_scripts/tree/master/SyncLogins" target="_blank">full script</a> to GitHub, if you're interested in it. 
