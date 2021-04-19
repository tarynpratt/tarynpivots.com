---
title: Testing Backups and Offloading CheckDB
date: 2021-04-19 06:00:00 +0000
draft: false
tags: [sql, sql-server, stack-overflow, backup, restore, restore-testing, maintenance, checkdb, availability-group]
excerpt: Performing backup testing and running CHECKDB on an alternative server
---

While every DBA knows they need to backup all their databases, not all may realize the importance of testing those backups. Performing backups is pointless, if you're unable to restore them.
 
I wanted to restore our backups on a regular basis, so I set up a process to test them automatically, on a nightly basis.

## Background

Early on when I started working on the SQL Servers at Stack Overflow, we were taking daily backups. We had a handful of databases that were being restored for other processes, but the majority weren't actively tested to ensure the backups were good. Since you never want to be in a situation where you need to restore a database and find it doesn't work, my goal was to create a process to automatically restore our backups to a separate server, and then run `DBCC CHECKDB` on it.
 
Now, you might be wondering why I'd run `CHECKDB` on the backup after I restore it and asking yourself "aren't you supposed to run `CHECKDB` on the SQL Server you take a backup on?"
 
Yes, technically many people suggest you do that, but we don't,  because while we take full backups on the primary replica in our <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server?view=sql-server-ver15" target="_blank">Always On Availability Groups</a>, we don't have maintenance windows, and running `CHECKDB` on a very busy production SQL Server like we have for Stack Overflow can cause issues (ask me how I know). We can't easily run a full `CHECKDB` in production, so since I was going to be restoring the backups to make sure they worked, I might as well run `CHECKDB `on it to verify everything.

## Automating the Testing

Before I jumped into creating a new process there were some questions I needed to answer for this project:

1. What databases do we want or need to test the backups for? 
2. Where can we perform this testing? Do we have a server available for use?
3. How often do we want to test the backups? Nightly? Weekly? What would the cadence be for this process?

The answer to the first question was pretty clear, since we can't easily run `DBCC CHECKDB` in production basically, any database that was in an Always On Availability Group was on the list to have the backup restored and tested. 

Next, I needed a place to perform the testing. Thankfully, we had a separate server where we were already restoring some backups, so it was the perfect candidate for this. The only issue was lack of disk space for some of the large databases. We resolved the lack of space issue by purchasing a couple of new NVMEs which gave us a 14TB drive to use for the restores.

Lastly, it was figuring out how often we wanted to test our restores. Since we take full backups nightly, we decided we would restore all databases daily, then run `DBCC CHECKDB` against it. Is that a little much? Maybe, but it also gives us confidence that our backups are working.

Now that I had the answer to the questions above, it was time to create a process that would do the following:

1. Automatically add or remove databases that need testing 
2. Restore the our backups on a separate server
3. Run <a href="https://docs.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql?redirectedfrom=MSDN&view=sql-server-ver15" target="_blank">`DBCC CHECKDB`</a> on the backup after being restored


### Maintaining a List of Databases to Test

I knew the initial list of databases we wanted to test, but what if that list changed? What would happen if we added new databases, or removed databases, and no longer had backups? I didn't want the nightly process to fail, and even though I typically know when we add or remove databases, I didn't want to hand-hold and edit the list manually, I wanted it to be dynamic.
 
To do this, I had to save the list of databases to restore. I did this by first creating a table in the `master` database on the server we were using for backup and restore testing. The table structure is:

{{< highlight sql>}}
CREATE TABLE [dbo].DatabasesToRestore(
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Name] [sysname] NOT NULL,
	[AvailabilityGroup] nvarchar(100) NULL,
	IsActive bit NOT NULL DEFAULT(1)
	CONSTRAINT [PK_DatabasesToRestore] PRIMARY KEY CLUSTERED ([Id] ASC)
) ON [PRIMARY]
GO
{{< / highlight >}}

This table stores basic information about each database, including name, but also the name of the `AvailabilityGroup` that the database is in. I purposely stored the AG name because of the way we save our backups. Our backup directory structure uses the Availability Group name instead of the name of the SQL Server in the file path because it allows the backup locations to stay consistent no matter what server is the primary. The file share for backups uses this directory structure:

{{< highlight text>}}
|--- server
    |--- Backups
        |--- SQL
            |--- AG-name1
                |--- Database1
                    --- database1_20210101.bak
                    --- database1_20210102.bak
                |--- Database2
                    --- database2_20210101.bak
            |--- AG-name2
                |--- Database1
                |--- Database2
{{< / highlight >}}

By storing the name of the Availability Group for each database, it allowed me to dynamically create the path to the backup for testing. It also made the process to find additions and deletions for each availability group more efficient, so I didn't have to manually adjust the list of databases that would need nightly testing. All that was needed was the initial population of the databases to restore, and then the stored procedure would keep the list up to date. The stored procedure that I wrote is:

{{< highlight sql>}}
CREATE OR ALTER Procedure [dbo].[spUpdateDatabasesToRestore] 
AS
BEGIN
  DECLARE @fileSearch nvarchar(500) = '',
        @backupPath NVARCHAR(500) = '',
        @error nvarchar(600);

  DECLARE @ag_name nvarchar(100)

  DECLARE ag_cursor CURSOR FOR
    SELECT DISTINCT AvailabilityGroup
    FROM dbo.DatabasesToRestore
  OPEN ag_cursor
  FETCH NEXT FROM ag_cursor INTO @ag_name
  WHILE @@FETCH_STATUS = 0
  BEGIN

    PRINT 'Getting databases for AG: '+ @ag_name

	-- go through the list of availability groups and their backup path
	-- update the list of databases to restore 
	-- adding new ones and deleting old ones that are no longer active databases

	SET @backupPath = concat('\\fileserver\Backups\SQL\', @ag_name, '\')
	SET @fileSearch = 'DIR /S /b/a-d/od/t:c ' + @backupPath;

	DECLARE @files table (ID int IDENTITY, FileName varchar(max), DBName sysname null, FileDate date null)

	INSERT INTO @files (FileName)
	EXEC master.sys.xp_cmdshell @fileSearch

	UPDATE @files 
	SET DBName = Left(replace(FileName, @backupPath, ''), charindex('\', replace(FileName, @backupPath, ''))-1),
	  FileDate = 
                cast(substring(right(replace(FileName, @backuppath, ''), len(replace(FileName, @backuppath, ''))
                -charindex('\', replace(FileName, @backuppath, ''))), 
                charindex('_FULL_', right(replace(FileName, @backuppath, ''), 
                len(replace(FileName, @backuppath, ''))-charindex('\', replace(FileName, @backuppath, '')))) + 6, 8) as date)
	WHERE FileName like '%.bak%'

	PRINT 'inserting new databases for AG: '+ @ag_name

	-- Insert new databases into the Databases table 
	INSERT INTO DatabasesToRestore (Name, AvailabilityGroup)
	SELECT DBName, AGName = @ag_name
	FROM
	(
		SELECT DBName, 
			rn = row_number() over(partition by DBName order by FileDate desc)
		FROM @files
		WHERE FileDate >= GETDATE()-7
			AND FileName LIKE '%'+@ag_name+'%'
	) f
	WHERE rn = 1
		AND NOT EXISTS (SELECT 1
				FROM dbo.DatabasesToRestore d
				WHERE f.DBName = d.Name
				AND d.AvailabilityGroup = @ag_name
    				AND d.IsActive = 1)

	PRINT 'Removing any old databases for AG: '+ @ag_name

	-- mark any Databases that are no longer producing backups as IsActive = 0
	UPDATE d
	SET d.IsActive = 0
	FROM dbo.DatabasesToRestore d
	LEFT JOIN 
	(
		SELECT DISTINCT DBName, AGName = @ag_name
		FROM @files 
		WHERE FileDate >= GETDATE()-7
			AND DBName IS NOT NULL
			AND FileName LIKE '%'+@ag_name+'%'
	) f
	    ON d.Name = f.DBName
	    AND d.AvailabilityGroup = f.AGName
	WHERE d.IsActive = 1
		AND d.AvailabilityGroup = @ag_name
		AND f.DBName IS NULL

	FETCH NEXT FROM ag_cursor INTO @ag_name
  END
  CLOSE ag_cursor  
  DEALLOCATE ag_cursor 
END
{{< / highlight >}}

This procedure uses <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql?view=sql-server-ver15" target="_blank">xp_cmdshell</a> to get the list of files in the backup directory, then adds new databases that are missing from the current list, and removes databases that haven't had a new .bak file in the last seven days. Finally, we have a scheduled job that executes this stored procedure nightly to keep the table of `DatabasesToRestore` up to date. 

### Restoring Database Backups

Once I had a list of the databases to restore, and a way to maintain that list on its own, it was time to get the process in place to restore the backups for testing nightly.
 
In addition to restoring the backups, I also wanted to keep a record of what was being restored, as a reference, in the event there were issues or failures with the nightly job. To do this, I created another table which I put in the `master` database.

{{< highlight sql>}}
CREATE TABLE dbo.DatabaseRestoreLog
(
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[DatabaseId] [int] NOT NULL,
	[RestoredName] [sysname] NULL,
	[RestoredDate] [date] NOT NULL,
	[RestoreStartTime] datetime NULL,
	[RestoreEndTime] datetime NULL,
	[DBCCStartTime] datetime NULL,
	[DBCCEndTime] datetime NULL,
	[PassedDBCCChecks] [bit] NOT NULL DEFAULT (0)
	CONSTRAINT [PK_DatabaseRestoreLog] PRIMARY KEY CLUSTERED ([ID] ASC),
	CONSTRAINT [FK_DatabaseRestoreLog_DatabasesToRestore] 
		FOREIGN KEY ([DatabaseId]) REFERENCES dbo.DatabasesToRestore ([Id])
);
{{< / highlight >}}

This table allows the job to capture critical details, like how long the restore took, it also tracked whether or not the backup passed the DBCC checks (more on that later).
 
As I mentioned earlier, we were restoring some databases in another process, so instead of inventing something from scratch, I decided to modify the existing stored procedure and expand it for the restore testing. Again, this procedure utilizes `xp_cmdshell` and loops through all of the databases in the `DatabasesToRestore` table and looks for the oldest databases that haven't been restored recently, creates the path to the backup, and then grabs the most recent .bak file to restore for testing.

{{< highlight sql>}}
CREATE OR ALTER Procedure dbo.spBackupRestoreTesting 
	@TimeLimit int = 43200,		-- the amount of time we want to limit the restores - the default is 12 hours
    @exec BIT = 0
AS
BEGIN
	DECLARE @fileSearch nvarchar(500) = '',
		@backupFile nvarchar(500),
		@data_dir nvarchar(100) = 'F:\DBRestore\', 
		@cmd nvarchar(max),
		@dropCmd nvarchar(500),
		@RestoredDBName sysname,
		@backupPath NVARCHAR(500) = '',
		@error nvarchar(600);

	-- Set the time we started the restore process so we know when we want to exit 
	-- if it hits the TimeLimit, we'll stop for the day
	DECLARE @StartTime datetime
	SET @StartTime = GETDATE()

	DECLARE @fileList Table (rowNumber int identity(1,1), backupFile nvarchar(255), backupPath nvarchar(500));

	DECLARE @FileListTable Table (
		[LogicalName] nvarchar(128),
		[PhysicalName] nvarchar(260), 
		[Type] char(1), 
		[FileGroupName] nvarchar(128), 
		[Size] numeric(20,0),
		[MaxSize] numeric(20,0), 
		[FileId] bigint, 
		[CreateLSN] numeric(25,0), 
		[DropLSN] numeric(25,0), 
		[UniqueId] uniqueidentifier, 
		[ReadOnlyLSN] numeric(25,0), 
		[ReadWriteLSN] numeric(25,0),
		[BackupSizeInBytes] bigint, 
		[SourceBlockSize] int, 
		[FileGroupId] int, 
		[LogGroupGUID] uniqueidentifier, 
		[DifferentialBaseLSN] numeric(25,0), 
		[DifferentialBaseGUID] uniqueidentifier, 
		[IsReadOnly] bit, 
		[IsPresent] bit, 
		[TDEThumbprint] varbinary(32), 
		[SnapshotUrl] nvarchar(360));

	DECLARE @database_id int
	DECLARE @database_name sysname
	DECLARE @ag_name nvarchar(100)
	DECLARE @last_restore_date date

	-- loop through the list of databases
	-- restore them, then run DBCC checks on them
	WHILE GETDATE() < DATEADD(ss,@TimeLimit,@StartTime)
	BEGIN
		-- grab the oldest databases that haven't been restored recently
		SELECT TOP 1 @database_id = d.Id, 
            @database_name = d.Name, 
            @ag_name = d.AvailabilityGroup, 
            @last_restore_date = drl.LastRestoredDate
		FROM DatabasesToRestore d
		LEFT JOIN
		(
			-- get the last date each database was restored
			SELECT 
				DatabaseId,
				LastRestoredDate = MAX(RestoredDate)
			FROM DatabaseRestoreLog
			GROUP BY DatabaseId
		) drl
			ON d.Id = drl.DatabaseId
		WHERE d.IsActive = 1
			AND (drl.LastRestoredDate <> CAST(GETDATE() AS DATE) 
			    OR drl.LastRestoredDate IS NULL)
		ORDER BY drl.LastRestoredDate 

		IF @@ROWCOUNT = 0
          BEGIN
            BREAK
          END

		PRINT CONCAT('Restoring : ', @database_name, ' in AG ', @ag_name, ' last restored date: ', @last_restore_date)

		SET @backupPath = CONCAT('\\fileserver\Backups\SQL\', @ag_name, '\', @database_name, '\')
		PRINT @backupPath

		SET @fileSearch = 'DIR *.bak /b /O:D ' + @backupPath;

		INSERT INTO @fileList(backupFile)
		EXEC master.sys.xp_cmdshell @fileSearch;
		UPDATE @fileList Set backupPath = @backupPath;

		SELECT Top 1 @backupFile = backupFile, @backupPath = backupPath
		FROM @fileList
		WHERE backupFile Like @database_name + '%'
		ORDER BY backupFile DESC

		IF @backupFile Is Not Null
			BEGIN
				DECLARE @fullPath nvarchar(500) = @backupPath + @backupFile;
				PRINT 'Backup File Found: ' + @fullPath;

				INSERT INTO @FileListTable
				EXEC('Restore FileListOnly From Disk = ''' + @fullPath + '''');

				SET @RestoredDBName = REPLACE(@backupFile, '.bak', '')

				-- insert into the log the Database we're restoring
				INSERT INTO dbo.[DatabaseRestoreLog] (DatabaseId, RestoredName, RestoredDate, RestoreStartTime)
				VALUES (@database_id, @RestoredDBName, GETDATE(), GETDATE());

				SET @cmd = 'Restore Database [' + @RestoredDBName + '] ' + char(13) + '  From Disk = ''' + @fullPath + ''' With File = 1, ' + char(13);
				SELECT @cmd = @cmd + '  Move N''' + LogicalName + ''' To N''' + @data_dir + LogicalName + '_' + @RestoredDBName + '.' +
					REVERSE(SUBSTRING(REVERSE(PhysicalName), 0, CHARINDEX('.', REVERSE(PhysicalName),0))) + ''', ' + char(13)
				FROM @FileListTable

				SET @cmd = @cmd + '  NoUnload, Stats = 5;' + char(13);
				SET @cmd = @cmd + 'Alter Database [' + @RestoredDBName + '] Set Recovery Simple With No_Wait;'

				-- if for some reason the database with the restored name exists, drop it first
				IF EXISTS (SELECT 1 
							FROM sys.databases 
							WHERE Name = @RestoredDBName)
					BEGIN
						PRINT 'Dropping existing [' + @RestoredDBName +']';
						SET @dropCmd = 'Alter Database [' + @RestoredDBName + '] Set Single_User With Rollback Immediate; Drop Database [' + @RestoredDBName + '];';
						PRINT(@dropCmd);

						IF @exec = 1 
							EXEC(@dropCmd);
					END

				-- if it doesn't exist, then restore the database
				PRINT 'Restoring [' + @database_name + '] from ' + @backupFile;
				PRINT(@cmd);
				IF @exec = 1 
					EXEC(@cmd);

				-- make sure the DB exists before trying to run DBCC CHECKDB on it
				IF EXISTS (SELECT 1 FROM sys.databases WHERE Name = @RestoredDBName) 
					BEGIN
						-- When the restore is complete, update the Log to reflect the end time
						UPDATE dbo.DatabaseRestoreLog
						SET RestoreEndTime = GETDATE(),
							DBCCStartTime = GETDATE()
						WHERE DatabaseId = @database_id
							AND RestoredName = @RestoredDBName

						EXEC dbo.spRunDBCCChecks @dbName = @RestoredDBName, @exec = @exec;

						-- When the DBCC Check is complete, update the Log to reflect the end time
						UPDATE dbo.DatabaseRestoreLog
						SET DBCCEndTime = GETDATE()
						WHERE DatabaseId = @database_id
							AND RestoredName = @RestoredDBName
					END
				ELSE 
					-- if the restore failed, then raise an error that it failed and send an emails
					BEGIN
						SET @error = 'Restore of database ''' + @RestoredDBName + ''' failed. Check the log on the server.' ;
						-- send an email alert to that the DBCC CHECKDB failed for the restored item
						EXEC msdb.dbo.sp_send_dbmail @profile_name = 'Mail', @recipients = 'useyourown@email.com', @subject = @error, @body = @error;
						RAISERROR(@error, 18, 1);
					END

			END
		ELSE
			BEGIN
				SET @error = 'No Recent backup was found in ''' + @backupPath + '''';
				RAISERROR(@error, 18, 1);
			END

		-- delete the info on the one we just restored
		DELETE FROM @fileList
		DELETE FROM @FileListTable
	END

	-- once we're done, let's clear out the DBCC_History_Log table a bit, cause it grows fast
	DELETE
	FROM dbo.DBCC_History_Log
	WHERE DBCCCheckDate < GETDATE() - 30;

	-- send an email to sql-alerts with a count of what was done and what passed
	DECLARE @TotalDatabasesRestored int 
	DECLARE @TotalPassedCheck int 

	SELECT 
		@TotalDatabasesRestored = count(*),
		@TotalPassedCheck = count(case when PassedDBCCChecks = 1 then DatabaseId end)
	FROM dbo.DatabaseRestoreLog
	WHERE RestoredDate = CAST(GETDATE() as date)

	DECLARE @subject nvarchar(100) = CONCAT('Database restore results for ', cast(getdate() as date))
	DECLARE @body nvarchar(500) = CONCAT('Total Databases Restored: ', @TotalDatabasesRestored, 'Total Databases Passing DBCC Check:',  @TotalPassedCheck)

	EXEC msdb.dbo.sp_send_dbmail @profile_name = 'Mail', 
		@recipients = 'useyourown@email.com', 
		@subject = @subject, 
		@body = @body;

	
END
{{< / highlight >}}

Once the database is restored we perform a `DBCC CHECKDB` on it to see if there are any issues. 

Since we restore hundreds of databases daily, I set a limit on the amount of time we perform restore testing each day. We currently limit testing to 12 hours a day, but if  need be, we could easily adjust this to 18 hours or some other value. I attempted to make this process flexible to account for adding or removing  databases.

### Running DBCC CheckDB

Outside of restoring our backups, the last piece is the most important...running a `CHECKDB` against the database since we do not do so in production. 

If you look closely at the previous stored procedure, you won't see a direct call to `DBCC CHECKDB`. That's because I created a separate stored procedure to make that call. But before we get to that stored procedure, there is one more table that I created for this process. Unless you are actively watching the `CHECKDB` run, you're going to want to capture the results so you can see any issues that were found. In order to do this, I created one more table to capture the lengthy results:


{{< highlight sql>}}
CREATE TABLE dbo.DBCC_History_Log
(
	[ID] [int] IDENTITY(1,1) NOT NULL CONSTRAINT PK_DBCC_History_Log PRIMARY KEY,
	[DatabaseName] [sysname] NULL,
	[DBCCCheckDate] [date] NOT NULL,
	[Error] [int] NULL,
	[Level] [int] NULL,
	[State] [int] NULL,
	[MessageText] [varchar](7000) NULL,
	[RepairLevel] nvarchar(500) NULL,
	[Status] [int] NULL,
	[DbId] [int] NULL,
	[DbFragId] [int] NULL,
	[ObjectId] [int] NULL,
	[IndexId] [int] NULL,
	[PartitionID] [bigint] NULL,
	[AllocUnitID] [bigint] NULL,
	[RidDbId] [int] NULL,
	[RidPruId] [int] NULL,
	[File] [int] NULL,
	[Page] [int] NULL,
	[Slot] [int] NULL,
	[RefDbId] [int] NULL,
	[RefPruId] [int] NULL,
	[RefFile] [int] NULL,
	[RefPage] [int] NULL,
	[RefSlot] [int] NULL,
	[Allocation] [int] NULL,
	[TimeStamp] [datetime] NULL CONSTRAINT [DF_dbcc_history_TimeStamp] DEFAULT (GETDATE())
);
{{< / highlight >}}

You might be wondering why I'd want to save this info? Since this is a nightly process, I just want it to run in the background and only notify me if something is wrong in the `CHECKDB`. I also don't want to have to rerun `CHECKDB` if there’s an issue. Essentially, I only need to review the data in the table to see the error, and investigate from there.
 
This table contains the standard output from a `CHECKDB`, but also includes a few additional columns including the name of the database being tested and the date that the check was performed. The table is populated via the final stored procedure in this process:

{{< highlight sql>}}
CREATE OR ALTER Procedure dbo.spRunDBCCChecks
    @dbName sysname,
    @exec BIT = 0 
AS
BEGIN
	DECLARE @dropCmd nvarchar(500),
			@error nvarchar(600),
			@MailMsg nvarchar(500);

	-- DBCC CheckDB will output results that we want to capture to verify if there are errors
	IF OBJECT_ID('tempdb..#tempdbcccheck') IS NOT NULL
		DROP TABLE #tempdbcccheck

	CREATE TABLE #tempdbcccheck
	(
		[Error] [int] NULL,
		[Level] [int] NULL,
		[State] [int] NULL,
		[MessageText] [varchar](7000) NULL,
		[RepairLevel] nvarchar(500) NULL,
		[Status] [int] NULL,
		[DbId] [int] NULL,
		[DbFragId] [int] NULL,
		[ObjectId] [int] NULL,
		[IndexId] [int] NULL,
		[PartitionID] [bigint] NULL,
		[AllocUnitID] [bigint] NULL,
		[RidDbId] [int] NULL,
		[RidPruId] [int] NULL,
		[File] [int] NULL,
		[Page] [int] NULL,
		[Slot] [int] NULL,
		[RefDbId] [int] NULL,
		[RefPruId] [int] NULL,
		[RefFile] [int] NULL,
		[RefPage] [int] NULL,
		[RefSlot] [int] NULL,
		[Allocation] [int] NULL
	);

	PRINT 'Starting DBCC CHECKDB on ' + @dbName

	INSERT INTO #tempdbcccheck
	EXEC ('DBCC CHECKDB(''' + @dbName + ''') with tableresults');

	-- insert the results in the history log table
	INSERT INTO [dbo].[DBCC_History_Log] (DatabaseName, DBCCCheckDate, Error, [Level], [State], MessageText, RepairLevel, [Status], DbId, DbFragId, 
		ObjectId, IndexId, PartitionID, AllocUnitID, RidDbId, RidPruId, [File], Page, Slot, RefDbId, RefPruId, RefFile, RefPage, RefSlot, Allocation)
	SELECT @dbName, GETDATE(), t.*
	FROM #tempdbcccheck t

	-- if the there were no errors in the database check, then we can safely drop the DB
	IF NOT EXISTS (SELECT 1 
					FROM dbo.DBCC_History_Log
					WHERE DatabaseName = @dbName
						AND DBCCCheckDate = cast(getdate() as date)
						AND [Level] > 10 )
		BEGIN
			UPDATE [dbo].[DatabaseRestoreLog]
			SET [PassedDBCCChecks] = 1
			WHERE [RestoredName] = @dbName
				AND [RestoredDate] = cast(getdate() as date)

			PRINT 'Clean DBCCCheck - dropping database [' + @dbName +']';
			SET @dropCmd = 'Alter Database [' + @dbName + '] Set Single_User With Rollback Immediate; Drop Database [' + @dbName + '];';
			PRINT(@dropCmd);

			IF @exec = 1 
			BEGIN
				EXEC(@dropCmd);
			END
		END
	ELSE
		BEGIN
			SET @error = 'DBCC CHECKDB generated errors for database ''' + @dbName + ''' not dropping for further review.' ;
			-- send an email alert to sql-alerts that the DBCC CHECKDB failed for the restored item
			EXEC msdb.dbo.sp_send_dbmail @profile_name = 'Mail', @recipients = 'useyourown@email.com', @subject = @error, @body = @error;
			RAISERROR(@error, 18, 1);
		END

END
{{< / highlight >}}

In this stored procedure I do a couple of things, run `CHECKDB` and then validate that the `CHECKDB` was successful. If it wasn't successful, then we don't drop the database, and we send an email alert stating that something needs to be investigated. If everything is ok with the `CHECKDB`, then we drop the database and it will either move to the next database to restore, or it will exit the job if we have hit the 12 hour time limit. 

To get this entire process done, I have two SQL Agent jobs running - one to update the list of the `DatabasesToRestore` via `spUpdateDatabasesToRestore` and one that executes the Backup and Restore Testing procedure (`spBackupRestoreTesting`). 

I've added all these scripts to <a href="https://github.com/tarynpratt/misc_sql_scripts/tree/master/BackupRestoreTesting" target="_blank">GitHub</a>, feel free to peruse them that way or tell me how terrible they are. 

## Final Thoughts

Currently, we're restoring 398 databases daily and running the `DBCC CHECKDB` on each one of them. Does that seem like a lot? Probably, but this process has, in fact, caught issues with some of our databases.
 
If the `CHECKDB` finds an issue, it doesn't drop the database (leaving it for investigation), it sends an email alert. This gives me ample time to figure out what has gone wrong and fix the issue on the production database.
 
Will this capture issues on the production server that don't exist in the backup? No, but it does give us confidence in our backups, and has found issues with databases that needed to be fixed - like data corruption. Ideally, you'd be able to run `CHECKDB` in production, but when you can't, you need to get creative. This process does two things - tests our backups daily to ensure they’re working, and runs the `CHECKDB` to make sure there is no corruption in the backup (which would bite us if we ever needed a backup).
 
How do you run `CHECKDB`? Do you run it on your production servers or do you run it on a separate server?

