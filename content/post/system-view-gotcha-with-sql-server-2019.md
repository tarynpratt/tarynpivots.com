---
title: A gotcha when upgrading to SQL Server 2019
date: 2020-02-21 06:00:38 +0000
draft: false
tags: [sql, sql-server, stack-overflow, sql-server-2019, server-upgrade, maintenance, projects]
excerpt: After upgrading to SQL Server 2019, job dependent on a system view started failing. 
---

In my [last post](https://www.tarynpivots.com/post/recovering-lost-linked-servers/), I mentioned that I started the process of upgrading Stack Overflow to <a href="https://docs.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-ver15?view=sql-server-ver15" target="_blank">SQL Server 2019</a>. This week I tackled our first production servers and after upgrading, we hit a small issue aka a gotcha because we were using an old system view. Below is a recap of what I encountered. 

## A Little Background

The servers I upgraded were the three SQL Servers that run <a href="https://stackoverflow.com/teams" target="_blank">Stack Overflow for Teams</a>. For those not familiar with our setup, we use <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver15" target="_blank">availability groups</a> (AGs) across all of our main SQL Servers. This allows us to use the primary server for read/writes, and use the secondaries for a lot of the read-only traffic. Since we utilize both the primary and secondaries for read purposes, logins need to be the same across all of the servers in the AGs. In order to keep the logins in sync, we have a job that periodically runs on each server and creates new logins using dynamic SQL. 

## Upgrade Day

Since I previously upgraded our servers to [SQL Server 2017](https://www.tarynpivots.com/post/how-we-upgraded-stackoverflow-to-sql-server-2017/), I knew what to expect as I moved through the cluster. Once I started with the secondary servers the databases would <a href="https://twitter.com/tarynpivots/status/1227676918083772416" target="_blank">not be accessible</a> until the failover was complete. After upgrading a secondary, I would see this if I tried to access a database:

![Database not accessible](/image/2020/database_unavailable.jpg)

This also meant that any of the jobs touching the databases would fail, so I disabled SQL Agent jobs to minimize noisy alerts to my email. After the upgrade and failover, I'd enable all the jobs and be on my merry way.

The upgrade of both secondary servers went totally as planned, without any issues. The evening rolled around and despite <a href="https://twitter.com/Nick_Craver/status/1229933221565038592" target="_blank">Nick Craver's popcorn</a>, the failover was successful, and we <a href="https://twitter.com/tarynpivots/status/1229936342471036928" target="_blank">finally had SQL Server 2019</a> running in production.

I upgraded the last server in the cluster (the former primary), and did some other clean-up tasks, including enabling all the SQL Agent jobs, before calling it a night. Once the jobs were enabled I started getting emails that one was failing &mdash; the Login Replication job. 

Something's broken - time to investigate. 

### The Failing Job

I pulled up the history on the job and saw the error:

>Message

>Executed as user: &lt;username&gt;. Invalid value given for parameter PASSWORD. Specify a valid parameter value. \[SQLSTATE 42000\] (Error 15021).  The step failed.

_Sigh_ ok, something is really broken because this was working before we failed over. 

The code for the login replication basically does the following via a cursor (yeah, I know, but it works...normally):

1. Select from the primary via `OPENQUERY` to query the logins and passwords
2. Using <a href="https://support.microsoft.com/en-us/help/918992/how-to-transfer-logins-and-passwords-between-instances-of-sql-server" target="_blank">`sp_hexadecimal`</a> convert the `varbinary` password to a `string` value
3. Create a string to be executed, i.e. dynamic SQL that runs a `CREATE LOGIN`

I have trimmed the code because it's long, but the key parts query the login and password.

{{< highlight sql>}}
SELECT *
FROM OPENQUERY([SQL-AG], '
    SELECT p.sid sid_varbinary, p.name, p.default_database_name,
        CAST(l.password AS varbinary (256)) pwd_varbinary,
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

The value of `pwd_varbinary` and `sid` are then used with `sp_hexadecimal`:

{{< highlight sql>}}
EXEC sp_hexadecimal @PWD_varbinary, @PWD_string OUT
EXEC sp_hexadecimal @SID_varbinary, @SID_string OUT
{{< / highlight >}}

Finally, we concatenate them into a SQL string that we execute:

{{< highlight sql>}}
SET @sqlString = ''CREATE LOGIN '' + QUOTENAME(@name) 
    + '' WITH PASSWORD = '' + @PWD_string + '' HASHED, SID = '' + @SID_string 
    + '', DEFAULT_DATABASE = ['' + @defaultdb + '']''
    + '', CHECK_POLICY = '' + @is_policy_checked
    + '', CHECK_EXPIRATION = '' + @is_expiration_checked
{{< / highlight >}}

When I ran the `SELECT` statement, I noticed the issue &mdash; the value of `pwd_varbinary` was `null` which obviously was wrong, and since we were passing a `null` as the password to `sp_hexadecimal` the job was failing. 

Great, so now what? 

### The Fix

We couldn't just stop replicating the logins across all servers, due to our usage of the secondaries, and we needed to do this automatically on a regular interval. We had to figure out a solution. Thankfully the fix was easy. 

After seeing that the `null` value of the password was coming from `sys.syslogins`, and since we were already querying <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-sql-logins-transact-sql?view=sql-server-ver15" target="_blank">`sys.sql_logins`</a> we could easily replace:

{{< highlight sql>}}
CAST(l.password AS varbinary (256)) pwd_varbinary,
{{< / highlight >}}

from `sys.syslogins` with 

{{< highlight sql>}}
sl.password_hash pwd_varbinary,
{{< / highlight >}}

from `sys.sql_logins` (h/t to <a href="https://twitter.com/Nick_Craver" target="_blank">Nick Craver</a>). With this one change to the query, the new `SELECT` statement was:

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

Once it was updated, the job successfully ran, and our logins were replicating again without issue. Now, I really called it a night. 

## Ok, but what was the gotcha?

Remember I said that this was working fine on SQL Server 2017? 

When we started looking for solutions, I looked up the Microsoft Docs for <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-compatibility-views/sys-syslogins-transact-sql?view=sql-server-ver15" target="_blank">`sys.syslogins`</a> and right at the top of the doc it says:

> This SQL Server 2000 system table is included as a view for backward compatibility. We recommend that you use the current SQL Server system views instead. To find the equivalent system view or views, see <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-tables/mapping-system-tables-to-system-views-transact-sql?view=sql-server-ver15" target="_blank">Mapping System Tables to System Views (Transact-SQL)</a>. This feature will be removed in a future version of Microsoft SQL Server. Avoid using this feature in new development work, and plan to modify applications that currently use this feature.

The doc shows the description and value of `password`:

| Column   | Data type     | Description   |
|----------|---------------|---------------|
| password | nvarchar(128) | Returns NULL. |

Hmm, that's weird because the `password` column contains an actual value in SQL Server 2017. Something obviously changed. Yes, we were risky because we were using an old view, but the important thing is the value in the `password` column changed, and it bit us. 

In SQL Server 2017, there is still a value for `password` in `sys.syslogins`, but in SQL Server 2019 it is now `null`. 

If this was mentioned somewhere, I missed it. Thankfully, this wasn't a critical job otherwise it could have been more problematic. We don't have the Login Replication job running in development, so we didn't hit the issue until we moved to production.

Technically it's on us because we were using an old system view and didn't check for changes, but keep in mind if you're using the `sys.syslogins` view anywhere, and rely on the `password`, you'll need to make a code update before moving to SQL Server 2019.

If you've upgraded to SQL Server 2019, have you hit any of these issues yet? 

