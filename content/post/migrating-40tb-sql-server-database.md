---
title: Migrating a 40TB SQL Server Database
date: 2020-07-28 06:00:38 +0000
draft: false
tags: [sql, sql-server, stack-overflow, availability-group, data-migration, powershell, maintenance, projects]
excerpt: Moving 40TB of data - table by table to a new database, then using an availability group to migrate it to a new server.
---

Initially, I wasn’t sure whether to write about this migration project, but when I randomly asked <a href="https://twitter.com/tarynpivots/status/1234966141526671360" target="_blank">if people would be interested</a>, the response was overwhelming. This was a long, kind of boring, very repetitive, and at times incredibly frustrating project, but I learned a lot, and maybe someone else will learn from this too. There may be far better ways to move this amount of data. In the path I went down, there was a huge amount of juggling that had to take place  (I’ll explain that later). Thankfully, I’m a decent juggler.

While most people are probably interested in what we do with our primary SQL Servers (the servers that run Stack Overflow and the Stack Exchange network of sites), this post isn’t about those servers. This post is about what I called a miscellaneous server in a [previous post](https://www.tarynpivots.com/post/how-we-upgraded-stackoverflow-to-sql-server-2017/) - more specifically, it’s about the migration of our traffic log data.

## Background

Our HAProxy<sup>1</sup> logs aka traffic logs, are currently stored on two SQL Servers, one in New York, and one in Colorado. While we store a minimal summary of the traffic data, we have a lot of it. At the beginning of 2019, we had about 4.5 years of data, totaling about 38TB. The database was initially designed to have a single table for each day. This meant that at the start of 2019 we had approximately 1600 tables in a single database with multiple database files (due to the <a href="https://docs.microsoft.com/en-us/sql/sql-server/maximum-capacity-specifications-for-sql-server?view=sql-server-ver15" target="_blank">16TB size limit of data files</a>). Each table had a <a href="https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview?view=sql-server-ver15" target="_blank">clustered columnstore index</a>, which held anywhere from 100-400 million rows in it.
 
I had to move the data from their existing daily tables into a new structure — one table for each month. This needed to be completed on both the NY and CO servers, and the data was stored on spinny drives (Eeek). No matter what, it was going to be painful and slow.

Oh, and to make things even more complicated, this had to be done with minimal free disk space on the existing servers. The servers had a 44TB spinny drive partition, and we were using 36-38TB of it, depending on the server. This meant I’d be following these steps throughout the process:
 
- migrate data to the new format
- delete old data
- shrink old database files...yeah, I know, but I had no choice
- repeat over, and over, and over again
 
I fully expected the process to take months, and it did &mdash; after hitting delays in getting new servers the entire project took about 11 months.

<hr />
 
<sup>1 </sup>Note: If you're interested, my colleague, Nick Craver (<a href="https://nickcraver.com/" target="_blank">b</a> | <a href="https://twitter.com/Nick_Craver/" target="_blank">t</a>), wrote extensively about our monitoring on his blog and gave an <a href="https://nickcraver.com/blog/2018/11/29/stack-overflow-how-we-do-monitoring/#logs-haproxy" target="_blank">overview of our HAProxy usage</a>.


### Things to Remember

Before I get started there are a few things I need to mention which impacted the project:

- I work remote 100% of the time, so I would not be able to run any of the processes from my local computer. Everything needed to be executed on a machine that was constantly connected to the network, so there wouldn’t be any VPN failures.
- I needed to run this on two separate machines. I was doing this on SQL servers in two different datacenters, I needed a machine in the same location, so I wouldn’t have to deal with slowness over the network.
- We have jump boxes that we use for various things at Stack Overflow, and we have one in NY and in CO, which were perfect to run the migration
- The old database was still being used in a live production environment, meaning that as I was moving data, we were constantly adding a new table daily. In other words, I had a moving target
- The databases are in `simple recovery` so we don't have to deal with transaction logs. Our backup is the original source log files and the pair of SQL Servers - NY and CO - are copies of each other. 

The bullets above were pain points throughout the project. Being disconnected from the VPN meant I’d have to reconnect to monitor the migration progress. Getting booted from a jumpbox, meant that whatever process was running would need to be kicked off again, after some clean-up. Since we were still inserting new data I was chasing the target, the longer it took, the more data I'd be moving in the end.

### Why, oh why would you do this?

I like to be miserable. Kidding.
 
There were lots of reasons this needed to be done, one being tech debt. We realized that the original daily table structure wasn't ideal. If we needed to query something over several days or months, it was terrible &mdash; lots of `UNION ALL`s or loops to crawl through days or even months at a time was slow. 

### Why not delete or purge some of the data?

As I mentioned, we were dealing with very little free disk space on both servers. This data is primarily used by the developers for investigating issues and by the data team for analysis. We already had purged some data, but for the data team - more data is better. 

Instead of purging, the plan was to eventually get new hardware to replace them, but we weren’t exactly sure when that would happen. The goal was to migrate the data to the new format, then when we got new servers it would be as simple as moving the drives to get the new database in place (or so we thought).

Each server had the following drives and approximate free space:
 
- a 230GB C: drive for Windows - obviously we wouldn't put data files here
- a 3.64GB NVMe D: drive - containing tempdb, one data file, and the log file for the existing HAProxyLogs database - it was approximately 85% full
- a 44TB E: drive filled with spinny disks - with the remaining 3 data files for the HAProxyLogs database - this was anywhere from 85-90% full
 
There was not a lot of room to move terabytes of data to a new database on the same server.

Since new servers were on down the road, I asked about the possibility of getting more disk space. Having more space would give me some breathing room to not be so crunched during the process. After some research, it was decided that we could fit a couple of additional NVMe SSDs into both servers. We only had PCIe slots available in the server, so we ended up using U.2 NVMe drives on PCIe adapters to get us some space. The SSDs gave us a brand spanking new F: drive that was **14TB** of free space. While I couldn’t move everything to the new drive, it gave me plenty of space to work with, at least, for a little bit.

Now that we had free space to use, it was time to set up the new database and begin the migration. I wrote the script to create the new database:

 
{{< highlight sql>}}
CREATE DATABASE [TrafficLogs] CONTAINMENT = NONE
ON PRIMARY
(   NAME = N'TrafficLogs_Current',
   FILENAME = N'F:\Data\TrafficLogs_Current.mdf',
   SIZE = 102400000KB,
   FILEGROWTH = 5120000KB
),
FILEGROUP [TrafficLogs_Archive]
(
   NAME = N'TrafficLogs_Archive1',
   FILENAME = N'E:\Data\TrafficLogs_Archive1.ndf' ,
   SIZE = 102400000KB,
   FILEGROWTH = 5120000KB
),
(
   NAME = N'TrafficLogs_Archive2',
   FILENAME = N'E:\Data\TrafficLogs_Archive2.ndf',
   SIZE = 102400000KB , FILEGROWTH = 5120000KB
),
(
   NAME = N'TrafficLogs_Archive3',
   FILENAME = N'E:\Data\TrafficLogs_Archive3.ndf',
   SIZE = 102400000KB,
   FILEGROWTH = 5120000KB
)
LOG ON
(
   NAME = N'TrafficLogs_log',
   FILENAME = N'F:\Data\TrafficLogs_log.ldf',
   SIZE = 5120000KB,
   MAXSIZE = 2048GB,
   FILEGROWTH = 102400000KB
);
{{< / highlight >}}
 
The database was going to have all of the old historical tables (i.e. the 40TBs) stored on the `TrafficLogs_Archive` filegroup, and we'd use the `PRIMARY` and `TrafficLogs_Current` for newer data being added.
 
You'll notice that the `TrafficLogs_Archive` filegroup went on our already very full E: drive, instead of the new F: drive - more on that mistake later.

## Moving All The Data

We had a database, so it was time to start the migration. To be clear, I was actually taking over a process that was started, stopped, and then punted from several years earlier. The project was on the backlog of things to be done long before I became the DBA at Stack Overflow. Back then, everyone realized what a time consuming project this would be, and since we didn’t have the resources to devote to it, the can continuously got kicked down the road. As a result, more and more tables were added making the overall task a lot larger.

Since this had been a previously abandoned project, there were already some scripts written, which meant I wasn’t totally starting from scratch. I was handed a couple of scripts to:
 
1. Create all of the new monthly tables

	{{< highlight sql>}}
-- written by Nick Craver
Declare @month datetime = '2015-08-01';
Declare @endmonth datetime = '2021-01-01'

WHILE @month < @endmonth
BEGIN

Set NoCount On;
Declare @prevMonth datetime = DateAdd(Month, -1, @month);
Declare @nextMonth datetime = DateAdd(Month, 1, @month);
Declare @monthTable sysname 
    = 'Logs_' + Cast(DatePart(Year, @month) as varchar) 
	  + '_' + Right('0' + Cast(DatePart(Month, @month) as varchar), 2);

Begin Try
  If Object_Id(@monthTable, 'U') Is Not Null
  Begin
    Declare @error nvarchar(400) 
	  = 'Month ' + Convert(varchar(10), @month, 120) 
	    + ' has already been moved to ' + @monthTable + ', aborting.';
	Throw 501337, @error, 1;
	Return;
  End

  -- Table Creation
  Declare @tableTemplate nvarchar(4000) = '
    Create Table {Name} (
	  [CreationDate] datetime Not Null,
	  <insert all the columns>,
	  Constraint CK_{Name}_Low Check (CreationDate >= ''{LowerDate}''),
	  Constraint CK_{Name}_High Check (CreationDate < ''{UpperDate}'')
  ) On {Filegroup};
		
    Create Clustered Columnstore Index CCI_{Name} 
	  On {Name} With (Data_Compression = {Compression}) On {Filegroup};';

  -- Constraints exist for metadata swap
  Declare @table nvarchar(4000) = @tableTemplate;
  Set @table = Replace(@table, '{Name}', @monthTable);
  Set @table = Replace(@table, '{Filegroup}', 'Logs_Archive');
  Set @table = Replace(@table, '{LowerDate}', Convert(varchar(20), @month, 120));
  Set @table = Replace(@table, '{UpperDate}', Convert(varchar(20), @nextMonth, 120));
  Set @table = Replace(@table, '{Compression}', 'ColumnStore_Archive');
  Print @table;
  Exec sp_executesql @table;
		
  Declare @moveSql nvarchar(4000) 
    = 'Create Clustered Columnstore Index CCI_{Name} 
	  On {Name} With (Drop_Existing = On, Data_Compression = Columnstore_Archive) On Logs_Archive;';
  Set @moveSql = Replace(@moveSql, '{Name}', @monthTable);
  Print @moveSql;
  Exec sp_executesql @moveSql;

End Try
Begin Catch
	Select Error_Number() ErrorNumber,
	    Error_Severity() ErrorSeverity,
	    Error_State() ErrorState,
	    Error_Procedure() ErrorProcedure,
	    Error_Line() ErrorLine,
	    Error_Message() ErrorMessage;
	Throw;
End Catch

set @month = dateadd(month, 1, @month)

END
GO
	{{< / highlight >}}
 
2. A LINQPad script to loop through each day, starting with the earliest, and insert the data into the new table

	{{< highlight cs>}}
-- written by Nick Craver
<Query Kind="Program">
<NuGetReference>Dapper</NuGetReference>
<Namespace>Dapper</Namespace>
</Query>

void Main()
{
  MoveDate(new DateTime(2015, 08, 1));

  DateTime date = new DateTime(2015, 08, 1);
  while (date < DateTime.UtcNow)
  {
    MoveDate(date);
    date = date.AddDays(1);
  }
}

static readonly List<string> cols = new List<string> { "<col list>" };

public void MoveDate(DateTime date)
{
  var tableName = GetTableName(date);
  var destTable = GetDestTableName(date);
  $"Attempting to migrate {date:yyyy-MM-dd} f
    from {tableName} to {destTable}".Dump($"{date:yyyy-MM-dd}");

  using (var conn = GetConn())
  {
    int rowCount;
	try
	{
	  rowCount = conn.QuerySingle<int>($"Select Count(*) 
						From HAProxyLogs.dbo.{tableName};");
	}
	catch (SqlException e)
	{
	  ("  Error migrating: " + e.Message).Dump();
	  return;
	}
	$"  Summary for {date:yyyy-MM-dd}".Dump();
	$"    {rowCount:n0} row(s) in {tableName}".Dump();
	var pb = new Util.ProgressBar($"{tableName} (0/{rowCount})");
	pb.Dump(tableName + " copy");

	Func<int> GetDestRowCount = () 
	  => conn.QuerySingle<int>($"Select Count(*) 
					From {destTable} 
					Where CreationDate >= @date 
					  And CreationDate < @date + 1;", new { date });
	  Action<long, int> UpdatePB = (copied, total) =>
	  {
		pb.Fraction = (double)copied / total;
		pb.Caption = $"{tableName} ({copied}/{total})";
	  };

	  var destRowCount = GetDestRowCount();
	  $"  {destRowCount:n0} row(s) in {destTable}".Dump();

	  if (destRowCount > 0)
	  {
		$"Rows found in destiation table - aborting!".Dump();
		return;
	  }

	var reader = conn.ExecuteReader($"Select {string.Join(",", cols)} 
					From HAProxyLogs.dbo.{tableName};");

	using (SqlConnection dest = GetConn())
	{
	  dest.Open();
			
	  using (SqlBulkCopy bc = new SqlBulkCopy(dest))
	  {
	    bc.BulkCopyTimeout = 5*60;
		bc.BatchSize = 1048576;
		bc.NotifyAfter = 100000;
		bc.DestinationTableName = GetDestTableName(date);
		bc.SqlRowsCopied += (e, s) => UpdatePB(s.RowsCopied, rowCount);

		foreach (var c in cols)
		{
		  bc.ColumnMappings.Add(c, c);
		}

		try
		{
 		  bc.WriteToServer(reader);
		}
		catch (Exception ex)
		{
		  Console.WriteLine(ex.Message);
		}
		finally
		{
		  reader.Close();
		}
	  }

	  var destFinalCount = GetDestRowCount();
	}
  }
}

public SqlConnection GetConn() =>
  new SqlConnection(new SqlConnectionStringBuilder
  {
	DataSource = "servername",
	InitialCatalog = "TrafficLogs",
	IntegratedSecurity = true
}.ToString());

public string GetTableName(DateTime dt) => $"Log_{dt:yyyy_MM_dd}";
public string GetDestTableName(DateTime dt) => $"HAProxyLogs_{dt:yyyy_MM}";
	{{< / highlight >}}
 
Ideally, this was going to be as simple as kicking off the LINQPad script, and letting it run in the background looping through all the tables. I thought I wouldn’t have to worry about it, but things are never really that easy.

I started the migration in Colorado, on January 14, 2019 (yes, I know the exact date). Pulling millions of rows of data from spinny drives, and then inserting it back into new tables on those same spinny drives was slow. By day 2, 
 <a href="https://twitter.com/tarynpivots/status/1085225095168065536" target="_blank">knew this was going to be painful</a>. My original plan was to run the migration in Colorado first, then in New York. However, once I saw how slow each table was taking, I decided to run them simultaneously. About a week later, I kicked off the New York data migration. 


## Data Migration Issues

Early on, I hit various issues that I needed to work through. Some of these were easy fixes, others not so much.
 
### Script Timeouts
 
The first issue was with the C# script. The script periodically timed-out when querying the database. After adjusting the settings, I crossed my fingers and hoped it would continue to work, which it did, for the most part.
 
### Random Script Errors
 
Next, I ran into an issue, where the script would continue to the next table, aka day, if there was an error. That was terrible because we wanted the data to be <a href="https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-query-performance?view=sql-server-ver15" target="_blank">inserted in date order</a>, for the clustered columnstore index. If the script errored and moved on to the next day, cleanup would need to be done for the day that didn’t finish. At that point, I’d have to start the process over for that particular day. 

You might be wondering what I mean by "cleanup". 

Let me take a quick step back and explain the old process. The original table structure aka the daily tables were rowstore with a generic identity (ID) column to make each row unique. After 14 days, we would drop the primary key from the table, and add a clustered columnstore index. Even though the conversion to a clustered columnstore occurred, the ID column would remain. 

In the new table, we dropped the ID column. Dropping this column meant, trying to figure out the uniqueness of the rows inserted before a failure would be incredibly difficult. It was far easier, in the event of a failure, to just delete everything from the table for that day, and everything after that day (if it moved to another table), and restart the process.
 
### Data Validation
 
I realized I needed to validate that the total number of rows inserted into the new table matched the total number of rows from the old daily table. Sounds easy, right? Well, not necessarily.

Sure, the original tables were per day, but there were cases where instead of containing data from `date 00:00-23:59` it would potentially include some rows from `date+1`. This was a known issue (due to latency of logs arriving from HAProxy), and I was told the totals didn’t have to match perfectly - if we lost some rows, it was fine. The thing is, when you’re responsible for moving data from one system to another, you really want things to match...perfectly.

Because of the stragglers from a different day, I couldn’t just query the total rows from the old table and verify that the new table contained the same count. I had to come up with a way to query the count by day. I did this by adding a column to the new table called `OriginalLogTable` and then populating it with the name of the old table, for example `Log_2019_07_01`. Having this column allowed me to compare the count of rows in the old table, with the count in the new table by grouping by the `OriginalLogTable`. This solved my problem because when I finished a month, I could easily verify that each day matched between the old and new tables, using the script below:
 
{{< highlight sql>}}
if object_id('tempdb..#NewTableDetails') is not null
    drop table #NewTableDetails
 
create table #NewTableDetails 
(
	TrafficLogOrigTable varchar(50), 
	TrafficLogTotalRows bigint, 
	OriginalLogDate date
)
 
if object_id('tempdb..#LogDetails') is not null
    drop table #LogDetails
 
create table #LogDetails (LogDate date, OriginalTotalRows bigint)
 
declare @sql nvarchar(max) = ''
declare @startdate datetime = '2019-10-01'  
declare @enddate datetime = '2019-11-01'
 
if object_id('tempdb..#dates') is not null
    drop table #dates;
 
CREATE TABLE #dates
(
  [Date] DATE PRIMARY KEY,
  FirstOfMonth AS CONVERT(DATE, DATEADD(MONTH, DATEDIFF(MONTH, 0, [Date]), 0)),
  OldTableName as concat('Log_', year([date]), '_', right('0' + rtrim(month([date])), 2), '_', right('0' + rtrim(day([date])), 2)),
  NewTableName as concat('HAProxyLogs_', year([date]), '_', right('0' + rtrim(month([date])), 2))
);
 
INSERT #dates([Date]) 
SELECT d
FROM
(
  SELECT d = DATEADD(DAY, rn - 1, @StartDate)
  FROM 
  (
    SELECT TOP (DATEDIFF(DAY, @StartDate, @enddate)) 
      rn = ROW_NUMBER() OVER (ORDER BY s1.[object_id])
    FROM sys.all_objects AS s1
    CROSS JOIN sys.all_objects AS s2
    ORDER BY s1.[object_id]
  ) AS x
) AS y;
 
-- for each new table name get the count of rows, get list of OldTables 
-- Imported and then get the counts for each one
declare @newtablename nvarchar(100)
declare @date datetime
declare newtable_cursor cursor for
    select distinct NewTableName
    from #dates
    where [date] < @enddate
open newtable_cursor 
fetch next from newtable_cursor into @newtablename
 
while @@FETCH_STATUS = 0
begin
    set @sql = 'insert into #NewTableDetails (TrafficLogOrigTable, TrafficLogTotalRows) 
                select 
                    TrafficLogOrigTable = OriginalLogTable, 
                    TrafficLogTotalRows = count(*)
                from TrafficLogs.dbo.[' + @newtablename +'] 
                group by OriginalLogTable ';
 
    exec sp_executesql @sql
 
    FETCH NEXT FROM newtable_cursor INTO @newtablename
end
CLOSE newtable_cursor  
DEALLOCATE newtable_cursor  
 
update #NewTableDetails
set OriginalLogDate 
    = Cast(Replace(Replace(TrafficLogOrigTable, 'Log_', ''), '_', '-') as date)
 
declare @oldTableName nvarchar(100)
declare @oldtabledate date
declare table_cursor cursor for
    select OriginalLogDate, TrafficLogOrigTable
    from #NewTableDetails
open table_cursor 
fetch next from table_cursor into @oldtabledate, @oldTableName
 
while @@FETCH_STATUS = 0
begin
    set @sql = 'insert into #LogDetails (LogDate, OriginalTotalRows) 
                select 
                    LogDate = '''+ convert(varchar(10), @oldtabledate, 23) +''', 
                    OriginalTotalRows = count(*) 
                from HAProxyLogs.dbo.[' + @oldTableName +']';
    exec sp_executesql @sql
 
 
    FETCH NEXT FROM table_cursor INTO @oldtabledate, @oldTableName
end
CLOSE table_cursor  
DEALLOCATE table_cursor  
 
select 
    nt.OriginalLogDate, 
    nt.TrafficLogOrigTable, 
    nt.TrafficLogTotalRows, 
    ot.OriginalTotalRows, 
    ot.LogDate, 
    IsMatch = case when nt.TrafficLogTotalRows = ot.OriginalTotalRows then 'Y' else 'N' end
from #NewTableDetails nt
full join #LogDetails ot
    on nt.OriginalLogDate = ot.LogDate
order by nt.OriginalLogDate
{{< / highlight >}}
 
### Column Changes
 
I also discovered that we added new columns after the original daily tables were designed. This meant that over the course of the data migration the table structure would change. I adjusted the LinqPad script so it would add the new columns when needed based on date, this would make sure it continued to work and we wouldn’t skip new columns.

There were many more issues which involved debugging the script, but eventually I had a C# script that fixed all of the above problems. The new script looked a little like this:


{{< highlight cs>}}
<Query Kind="Program">
 <NuGetReference>Dapper</NuGetReference>
 <Namespace>Dapper</Namespace>
</Query>
 
void Main()
{
   DateTime date = new DateTime(2016, 07, 26);
   while (date < DateTime.UtcNow)
   {
       MoveDate(date);
       date = date.AddDays(1);
   }
}
 
static readonly List<string> cols = new List<string> { "<col list>" };
 
public void MoveDate(DateTime date)
{
   var tableName = GetTableName(date);
   var destTable = GetDestTableName(date);
   $"Attempting to migrate {date:yyyy-MM-dd} from {tableName} 
   		to {destTable}".Dump($"{date:yyyy-MM-dd}");
 
   if (date >= new DateTime(2017, 01, 12))
   {
       cols.Add("new col1");
       cols.Add("new col2");
   }
 
   using (var conn = GetConn())
   {
       int rowCount;
       try
       {
           rowCount = conn.QuerySingle<int>($"Select Count(*) 
		   				From HAProxyLogs.dbo.{tableName};");
       }
       catch (SqlException e)
       {
           ("  Error migrating: " + e.Message).Dump();
           return;
       }
       $"  Summary for {date:yyyy-MM-dd}".Dump();
       $"    {rowCount:n0} row(s) in {tableName}".Dump();
       var pb = new Util.ProgressBar($"{tableName} (0/{rowCount})");
       pb.Dump(tableName + " copy");
 
       Func<int> GetDestRowCount = () 
	    => conn.QuerySingle<int>($"Select Count(*) 
			   		From {destTable} 
					Where CreationDate >= @date 
					  And OriginalLogTable = '{tableName}';", new { date });
       Action<long, int> UpdatePB = (copied, total) =>
       {
           pb.Fraction = (double)copied / total;
           pb.Caption = $"{tableName} ({copied}/{total})";
       };
 
       var destRowCount = GetDestRowCount();
       $"There are  {destRowCount:n0} row(s) for table: {destTable}".Dump();
 
       if (destRowCount > 0)
       {
           $"Rows found in destination table - aborting!".Dump();
           return;
       }
 
       var sqlString = $"Select {string.Join(",", cols)}, OriginalLogTable = '{tableName}' 
	   			From HAProxyLogs.dbo.{tableName};";
       var reader = conn.ExecuteReader(sqlString);
 
       using (SqlConnection dest = GetConn())
       {
           dest.Open();
      
           using (SqlBulkCopy bc = new SqlBulkCopy(dest))
           {
              
               bc.BulkCopyTimeout = 15*60;
               bc.BatchSize = 1048576;
               bc.NotifyAfter = 100000;
               bc.DestinationTableName = GetDestTableName(date);
               bc.SqlRowsCopied += (e, s) => UpdatePB(s.RowsCopied, rowCount);
 
               foreach (var c in cols)
               {
                   bc.ColumnMappings.Add(c, c);
               }
 
               try
               {
                   bc.ColumnMappings.Add("OriginalLogTable", "OriginalLogTable");
                   bc.WriteToServer(reader);
               }
               catch (Exception ex)
               {
                   Console.WriteLine(ex.Message);
               }
               finally
               {
                   reader.Close();
               }
           }
 
           // try updating stats after full dump to see if that stops time-outs
           var statsString 
		= $"waitfor delay '00:00:05'; 
		    update statistics dbo.[{destTable}] [{destTable}_OriginalLogTable];";
           conn.Query(statsString);
 
           var destFinalCount = GetDestRowCount();
           $"After migration there are {destFinalCount:n0} row(s) in {destTable}".Dump();
       }
   }
}
 
 
public SqlConnection GetConn() =>
   new SqlConnection("Data Source=servername;Initial Catalog=TrafficLogs;Trusted_Connection=True;Connection Timeout = 1500;");
 
public string GetTableName(DateTime dt) => $"Log_{dt:yyyy_MM_dd}";
public string GetDestTableName(DateTime dt) => $"HAProxyLogs_{dt:yyyy_MM}";
 
{{< / highlight >}}

## It's So Slow

We all knew this was going to be a slow process, but I don't think anyone realized how slow. I did everything I could think of to try and speed up the script.
 
It took about 2-2.5 hours to insert each old table into the new table. By the middle of February, I had:
 
- 1332 tables left to migrate in New York
- and 782 tables left in Colorado
 
At the rate it was going, I knew <a href="https://twitter.com/tarynpivots/status/1098695568140906496" target="_blank">it was going to take months to complete</a>. 
 
I went back to the drawing board to figure out some way to move the data faster. Obviously, I wanted to follow the same steps - insert old data, verify the row count matches, move to the next day, repeat - I used the C# script as my model - I mean it worked, it was just a little slow. I tried a bunch of different things, none were blazingly fast:
 
- an SSIS package - sorry SSIS fans, this was even slower and I won't embarrass myself by sharing it
- a SQL script using dynamic SQL - this was also pretty slow

{{< highlight sql>}}
declare @startDate datetime = '2015-11-01'
declare @endDate datetime = '2015-12-01'

declare @OldTableName varchar(100)
declare @NewTableName varchar(100)

declare @cols varchar(max) = 'comma, separated, col, list'
declare @originalrowcount int
declare @newrowcount int
declare @finalrowcount int
declare @rowcountsql nvarchar(max)

declare @insertsql nvarchar(max)

declare @statssql nvarchar(max)

while @startDate < @endDate
begin

set @OldTableName 
  = concat('Log_', year(@startDate), '_'
	  , right('0' + rtrim(month(@startDate)), 2), '_'
	  , right('0' + rtrim(day(@startDate)), 2))
set @NewTableName 
  = concat('HAProxyLogs_', year(@startDate), '_'
	  , right('0' + rtrim(month(@startDate)), 2))

print concat('Attempting to migrate ', convert(varchar(10), @startDate, 120)
	  , ' from ', @OldTableName, ' to ', @NewTableName)
print ' '

if @startDate >= '2017-01-12'
  begin
    if (charindex('new col1', @cols) = 0)
  	  begin
	    set @cols = concat(@cols, ', new col1')
	  end

	if (charindex('new col2', @cols) = 0)
	  begin
	    set @cols = concat(@cols, ', new col2')
	  end
  end

set @rowcountsql = concat('select @cnt = count(*) from HAProxyLogs.dbo.[', @OldTableName, ']');
exec sp_executesql @rowcountsql, N'@cnt int output', @cnt = @originalrowcount output;

/*
print concat('Summary for ', convert(varchar(10), @startDate, 120)
	  , ' ', @originalrowcount, ' rows in ', @OldTableName);
print ' '
print concat('Starting copy of ', @OldTableName);
print ' '
*/

-- check if the new table has any rows in it
set @rowcountsql 
  = concat('select @cnt = count(*) from ', @NewTableName
  	  , ' where CreationDate >= ''', convert(varchar(10), @startDate, 120)
	  , ''' and OriginalLogTable = ''', @OldTableName, ''';')
--print @rowcountsql
exec sp_executesql @rowcountsql, N'@cnt int output', @cnt = @newrowcount output;

print concat('There are ', @newrowcount, ' row(s) for table: ', @NewTableName);
print ' '

If @newrowcount > 0
	begin
		print 'Rows found in the destination table - aborting'
		print ' '
		break;
	end

set @insertsql 
  = concat('insert into TrafficLogs.dbo.', @NewTableName, '(', @cols, ', OriginalLogTable) 
	  select ', @cols, ', OriginalLogTable = ''', @OldTableName, '''  
	  from HAProxyLogs.dbo.', @OldTableName);
print @insertsql
print ' '
exec sp_executesql @insertsql;

set @statssql 
  = concat('update statistics TrafficLogs.dbo.', @NewTableName
  	  , ' ', @NewTableName, '_OriginalLogTable');
print @statssql
print ' '
exec sp_executesql @statssql

set @rowcountsql 
  = concat('select @cnt = count(*) from ', @NewTableName
    	, ' where CreationDate >= ''', convert(varchar(10), @startDate, 120)
		, ''' and OriginalLogTable = ''', @OldTableName, ''';')
exec sp_executesql @rowcountsql, N'@cnt int output', @cnt = @finalrowcount output;

print concat('After migration there are ', @finalrowcount, ' row(s) in the ', @NewTableName);

If @originalrowcount <> @finalrowcount
  begin
    print 'final row does not match original stopping operation';
	print ' '
	break;
  end

set @startDate = dateadd(day, 1, @startDate)

END

{{< / highlight >}}
 
- a PowerShell version - slow but seemed to be more stable than the original C# script

{{< highlight powershell>}}
<#
  .SYNOPSIS
  Traffic Log Data Migration

  .EXAMPLE
  .\DataMigration.ps1 -ServerName name -StartDate 2019-01-01 -EndDate 2019-02-01
#>

[CmdletBinding()] #See http://technet.microsoft.com/en-us/library/hh847884(v=wps.620).aspx for CmdletBinding common parameters
param(
  [parameter(Mandatory = $true)]
  [string]$ServerName,
  [parameter(Mandatory = $true)]
  [DateTime]$StartDate,
  [parameter(Mandatory = $true)]
  [DateTime]$EndDate
)

# taken from https://blog.netnerds.net/2015/05/getting-total-number-of-rows-copied-in-sqlbulkcopy-using-powershell/
$source = 'namespace System.Data.SqlClient
{   
  using Reflection;
  public static class SqlBulkCopyExtension
  {
    const String _rowsCopiedFieldName = "_rowsCopied";
    static FieldInfo _rowsCopiedField = null;
    public static int RowsCopiedCount(this SqlBulkCopy bulkCopy)
    {
        if (_rowsCopiedField == null) _rowsCopiedField 
			= typeof(SqlBulkCopy).GetField(_rowsCopiedFieldName, BindingFlags.NonPublic 
			    | BindingFlags.GetField | BindingFlags.Instance);           
        return (int)_rowsCopiedField.GetValue(bulkCopy);
    }
  }
}
'
Add-Type -ReferencedAssemblies 'System.Data.dll' -TypeDefinition $source
$null = [Reflection.Assembly]::LoadWithPartialName("System.Data")

Function ConnectionString([string] $ServerName, [string] $DbName)
{
  "Data Source=$ServerName;Initial Catalog=TrafficLogs;Trusted_Connection=True;Connection Timeout = 2000;"
}

Function GetOriginalTableRowCount($sqlConnection, $table){
  $sqlConnection.open();
  $OriginalRowCountCmd = "select OriginalCount = count(*) from " + $table;
  $SqlCmd = New-Object  System.Data.SqlClient.SqlCommand($OriginalRowCountCmd, $sqlConnection)
  $SqlCmd.CommandTimeout = 0;
  $row_count = [Int64] $SqlCmd.ExecuteScalar()
  $sqlConnection.Close();
  return $row_count
}

Function GetNewTableRowCount($sqlConnection, $table){
  $sqlConnection.open();
  $RowCountCmd = "select NewCount = count(*) from " + $table 
  		+ " where CreationDate >= '"+($i.ToString("yyyy-MM-dd"))
		  +"' and OriginalLogTable = 'Log_"+($i.ToString("yyyy_MM_dd"))+"'";
  $SqlCmd = New-Object  System.Data.SqlClient.SqlCommand($RowCountCmd, $sqlConnection)
  $SqlCmd.CommandTimeout = 0;
  $row_count = [Int64] $SqlCmd.ExecuteScalar()
  $sqlConnection.Close();
  return $row_count
}

$cols = "col1", "col2", "col3", "...";

for ($i = $StartDate; $i -lt $EndDate; $i = $i.AddDays(1))
{
  $MigrationStart = Get-Date
  Write-Host "Migration started at: $MigrationStart"
  $OldTableName = "HAProxyLogs.dbo.Log_"+($i.ToString("yyyy_MM_dd"))
  $NewTableName = "TrafficLogs.dbo.HAProxyLogs_"+($i.ToString("yyyy_MM"))
  Write-Host "Attempting to migrate "($i.ToString("yyyy_MM_dd"))" from " $OldTableName  " to " $NewTableName;

  # build the connection string and open it
  $SrcConnStr = ConnectionString $ServerName
  $SrcConn  = New-Object System.Data.SqlClient.SQLConnection($SrcConnStr)

  # whats the row count on the original table in HAProxyLogs
  $originalRowCount = GetOriginalTableRowCount $SrcConn $OldTableName
  Write-Host "Summary for " ($i.ToString("yyyy-MM-dd"))": 
  	  	There are "$originalRowCount" rows in table "$OldTableName

  # make sure there is no data in the table currently for the day
  $newRowCount = GetNewTableRowCount $SrcConn $NewTableName

  if($newRowCount -gt 0){
    Write-Host "Rows found in the destination table - aborting migration"
    break;
  }
  else {
    Write-Host "No data for the date in the destination table - continuing migration"
  }

  # select the data we need from the original table
  $selectCmdText = "select "+ ($cols -join ', ') 
  		  +", OriginalLogTable = 'Log_"+($i.ToString("yyyy_MM_dd"))
		  +"' from "+$OldTableName;
  $SqlCommand = New-Object system.Data.SqlClient.SqlCommand($selectCmdText, $SrcConn)
  $SqlCommand.CommandTimeout = 0;
  $SrcConn.Open()
  [System.Data.SqlClient.SqlDataReader] $SqlReader = $SqlCommand.ExecuteReader()

  $destConnection = ConnectionString $ServerName
  $bulkCopy = New-Object Data.SqlClient.SqlBulkCopy($destConnection)
  $bulkCopy.BatchSize = 1048576
  $bulkCopy.BulkCopyTimeout = 0 # 20*60
  $bulkCopy.DestinationTableName = $NewTableName
  $bulkCopy.NotifyAfter = 500000

  # taken from https://blog.netnerds.net/2015/05/getting-total-number-of-rows-copied-in-sqlbulkcopy-using-powershell/
  $bulkCopy.Add_SqlRowscopied(
    {Write-Host "$($args[1].RowsCopied)/$originalRowCount rows copied 
	    from $OldTableName to $NewTableName" })

  foreach ($col in $cols){
    $bulkCopy.ColumnMappings.Add($col,$col)
  }

  try {
    Write-Host "Starting Migration"
    $bulkCopy.ColumnMappings.Add("OriginalLogTable", "OriginalLogTable");
    $bulkCopy.WriteToServer($SqlReader)

    $total = [System.Data.SqlClient.SqlBulkCopyExtension]::RowsCopiedCount($bulkcopy)
    Write-Host "$total total rows written"

  }
  catch {
    $ex = $_.Exception
    Write-Error $ex.Message
    While($ex.InnerException)
    {
      $ex = $ex.InnerException
      Write-Error $ex.Message
    }
  }
  Finally
  {
    $SqlReader.Close()
    $bulkCopy.Close()
    $SrcConn.Close()
  }

  # once done update statistics on the new table
  $statsCmdText = "update statistics "+$NewTableName
  	  	+" HAProxyLogs_"+($i.ToString("yyyy_MM"))+"_OriginalLogTable";
  Invoke-DbaQuery -SqlInstance $ServerName -Query $statsCmdText;

  Write-Host "After migration there are $total rows 
  		were written to $NewTableName table from $OldTableName";

  if($originalRowCount -ne $total){
    Write-Host "total rows migrated does not match. Stopping operation"
    break;
  }

  $total = 0;

}

{{< / highlight >}}
 
All of my options were slow. 
 
I ended up using the PowerShell option because I could easily run multiple months at once. I had 3 PowerShell sessions going, which let me chew through 3 months at a time. It was far easier to run several months in PowerShell than with the C# script. Utilizing PowerShell allowed me to get through a lot more data in a shorter amount of time. Don’t get me wrong it was still slow - each session was moving about a month of tables in about a week of work - but at least I was moving more data and chasing the target slightly faster. 

While the PowerShell script wasn’t perfect, and I still hit issues with timeouts and errors, it overall worked better. Now that I had a somewhat working process that didn’t need daily hand-holding, it was time to focus on some of the other problems with the migration.

 
## Continuous Issues of No Free Disk Space

When your servers are already limited in free space and you have to move 40TB of data around on the same server, it becomes a little like <a href="https://en.wikipedia.org/wiki/Whac-A-Mole" target="_blank">whack-a-mole</a> to get things moved. I had to get creative to free up space while continuing to move all the data to the new format. 


### Deleting and Shrinking

Yes, you read that right. Shrinking. Before I get to that, let me explain why I needed to do this. 

We didn't have new hardware yet and these were production SQL Servers - meaning the old database was still receiving new data and we were adding a new table everyday. As I was moving tables to the new database, we were adding tables with hundreds of millions of rows of data to the old database. This was all happening on the exact same server and drive with minimal free space. 

Hence the need to juggle. 
 
After moving each month, I would do a final validation, then it was time for some cleanup. This included deleting all of the daily tables for the month that were just migrated, as well as dropping the `OriginalLogTable` column from the monthly table in the new database. The daily tables were no longer needed because we had the data in the new table, so getting rid of duplicate data would allow me to gain back disk space...sort of.

I say sort of, because I was just moving data on the same drives. We changed formats from daily tables to monthly tables, and while we got better compression in the new tables, there weren’t huge gains in free space. We had the same number of rows of data stored in fewer tables, which meant less bits in the long run, but in order to gain back the space allocated to the old data files, I had to shrink them.

<img src="/image/2020/scales.png" width="300" height="300" style="float:right" alt="Balancing Scales" />
 
As I moved the data from the old files, I was increasing the size of the new data files. This was all being done on the E: drive that was 85-90% full, which I mentioned above. I was constantly trying to keep the used disk space in check. 

It was quite a balancing act. 

Each month of data was anywhere from 300GB to 1TB in size, and the 3 old data files were approximately 12TB each. At the end of each migration, I tried to shrink the old database files by the amount we just deleted. This took several attempts to get something that worked well.
 
Initially, I naively tried to run `DBCC SHRINKFILE(FileName, TRUNCATEONLY)` to free up space. As I expected, that did not work.
 
Next, I tried to run `DBCC SHRINKFILE(FileName, CurrentFileSize - SizeOfDataJustDeleted)` on each file. That didn’t work either. Well to be clear, it would have if we had more drive space on the D: drive where the log file was stored. The D: drive was already about 85% full with tempdb, the HAProxyLogs log file, and another datafile. When you run `DBCC SHRINKFILE` and let’s say you want to shrink a file by 300GB, then you <a href="https://karaszi.com/why-you-want-to-be-restrictive-with-shrink-of-database-files" target="_blank">need to have the ability to increase your database transaction log file by the amount you’re trying to shrink</a>. Since we were already constrained on drive space where the t-log was stored, I wasn’t able to shrink the files by the amount I deleted because I kept running out of space on the D: drive. 
 
I had a couple of options to free up space on the D: drive - move `tempdb` or move the other log/database files to our newly configured F: drive with 14TB of free space. I decided to move `tempdb` because it was as easy as executing the following command and restarting the SQL service. 
 
{{< highlight sql>}}
alter database tempdb 
	modify file (NAME = tempdev, FILENAME = 'F:\Data\tempdev.mdf', SIZE = 20000MB);
alter database tempdb 
	modify file (NAME = templog, FILENAME = 'F:\Data\templog.ldf', SIZE = 1000MB);
alter database tempdb 
	modify file (NAME = tempdev2, FILENAME = 'F:\Data\tempdev2.mdf', SIZE = 20000MB);
alter database tempdb 
	modify file (NAME = tempdev3, FILENAME = 'F:\Data\tempdev3.mdf', SIZE = 20000MB);
alter database tempdb 
	modify file (NAME = tempdev4, FILENAME = 'F:\Data\tempdev4.mdf', SIZE = 1000MB);
{{< / highlight >}}
 
By moving `tempdb` we had minimal downtime and I gained approximately 122GB in free space. While that gave me a little more room to shrink the old data files, it still wasn’t quite working as I wanted, so it was back to the drawing board. 
 
Trying to shrink the files in huge chunks of 300GB or more was too much, it was slow and resulted in blocking of the inserts into the old database. I found a <a href="https://medium.com/@anna.f/shrink-oversized-data-files-in-microsoft-sql-server-53fb640f893e" target="_blank">solution that seemed promising</a> and would involve shrinking the files in much smaller bits in a loop, so I figured I’d try it. It took a little bit of poking to get the <a href="https://twitter.com/tarynpivots/status/1095430853507788802"target="_blank">right number to shrink in every execution</a>, but I eventually ended up using something similar to this:
 
{{< highlight sql>}}
declare @from int                             
declare @leap int                             
declare @to int                             
declare @datafile varchar(128)
declare @cmd varchar (512)
declare @starttime datetime
declare @endtime datetime
 
/*settings*/      
set @from = 13533200            /*Current size in MB*/
set @to = 11600000              /*Goal size in MB*/ 
set @datafile = 'HAProxyLogs1'  /*Datafile name*/
set @leap =  10000            	/*Size of leaps in MB*/
 
while ((@from - @leap) > @to)                             
    begin
 
        set @starttime = getdate()
 
        print concat('started: ', @starttime)
        
        set @from = @from - @leap
        set @cmd = 'DBCC SHRINKFILE (' + @datafile +', ' + cast(@from as varchar(20)) + ')'
        print @cmd
        exec(@cmd)
        print '==>    SHRINK SCRIPT - '+ cast ((@from-@to) as  varchar (20)) + 'MB LEFT'        
        
        set @endtime = getdate()
 
        print concat('ended: ', getdate(), ' took :' , datediff(minute, @starttime, @endtime))
    end
 
set @cmd =  'DBCC SHRINKFILE (' + @datafile +', ' + cast(@to as varchar(20)) + ')'
print @cmd
exec(@cmd)
{{< / highlight >}}
 
I’d run the script against each file separately to fully recover the disk space from the month that was deleted. Again it was slow, but it worked.

I’m aware of all the issues with shrinking database files, and no, I wouldn’t normally do it. However, shrinking the files was necessary to move the project forward, as I had to get around the lack of disk space.

### Just Use the New Drive
 
Now that I had a script to migrate the data relatively quickly (the PowerShell one), and had a script to shrink the old data files, you’d think I’d be all set to just run with this project, right?
 
Not so fast.

When I set up the new TrafficLogs database I put the data files for the archive (the old stuff being migrated) on the drive with the spinny disks. We didn’t think about using the new F: drive with 14TB of free SSD space to use, mainly because  it wasn’t going to be enough for the entire migration. But after a few weeks, I decided to use that drive for a few reasons:
 
- They were NVMe SSDs. Which meant writing to them was much faster than writing to the the spinny drives
- It was about 14TB of free space, which let me move a huge amount of data without worrying about shrinking immediately. I could punt some of the whack-a-mole with the disk space.
 
Unfortunately, the only issue with doing this was once I filled the 14TB of disk space, I had to migrate the datafiles all back to the spinny drives. Which I did at the end of March 2019 after I ran out of drive space.

Was this ideal? No. Was it a pain? Yes. But again, it moved the entire project forward and I was still slowly getting all the data migrated over.


### Drives, Please Don't Die

In early May 2019, I was still plugging away at migrating data. At this point, I was back to where I started - copying data from the old database on the spinny drives, inserting it into the new database, deleting the old data, and then shrinking the files all on the same drives.

<img src="/image/2020/edrive_slowness.png" width="500" height="500" style="float:right"  alt="Read/Write Stats of Spinny Drives"/>

After months of doing this, I was seeing a noticeable slowdown in the process. The read/write delays on the drives had increased since January. The <a href="https://twitter.com/tarynpivots/status/1126581287471407104" target="_blank">average delays</a> of reads were over 100 milliseconds and about 80 milliseconds for writes.

The <a href="https://twitter.com/tarynpivots/status/1131620681794240513" target="_blank">overall response time</a> was terrible. 

I was struggling with <a href="https://twitter.com/tarynpivots/status/1130566711088799744" target="_blank">`PAGEIOLATCH_SH` waits</a> as I was trying to insert data into the new tables. 

We had been using the spinny drives for years, and I had been hitting them non-stop for about 4 months. I was <a href="https://twitter.com/tarynpivots/status/1127320396619890688" target="_blank">very worried the drives were going to fail</a> before I was done migrating all of the data.

Thankfully, they didn't. I eventually was able to move all of the old data over to the new database. It only <a href="https://twitter.com/tarynpivots/status/1144251803820810241" target="_blank">took 6 months to complete</a>.

![Traffic Logs Done](/image/2020/traffliclog_tables.jpg)

I was done, but not really. We still didn’t have the new servers which meant every table created would still need to be migrated until we had them, and had cutover to the new servers and database.

The moving target was still moving. 

## Finally, New Hardware

The plan was to get the new servers in June/July, but the purchase was delayed several months. After respeccing them to have all NVMe SSD storage, they finally arrived in late September. To make the final migration of the new hardware as easy as possible, from June through the end of September, I moved one table a day, keeping things up to date. 

In early October, the new servers were ready for me to start installing SQL Server on them. The end was near.

After lots of fussing with the Windows Server install, I eventually got installed SQL Server 2017 on both of them. It was time for the next obstacle moving the gigantic databases over.

### Backup and Restore Failure

The most obvious way to move a database from the old server to the new server is, of course, to take a backup and then restore it. So, that’s what <a href="https://twitter.com/tarynpivots/status/1182779767621181440" target="_blank">I did</a>, or I should say, tried to do. I say tried because when I took the backup, I set the script to write it out on the new server, but unfortunately, <a href="https://twitter.com/tarynpivots/status/1184804705844613125" target="_blank">the backup took most of the space</a> which meant I couldn’t restore it. It was a bit of a <a href="https://en.wikipedia.org/wiki/Facepalm#:~:text=A%20facepalm%20is%20the%20physical,in%20contact%20with%20the%20face." target="_blank">facepalm moment</a>.

We're a very lean shop when it comes to hardware, and this is by far our largest chunk of data we have to store, so there wasn’t a place anywhere on the network to store a 33-38TB full backup temporarily, while it was being restored to a server.

Time for more <a href="https://twitter.com/tarynpivots/status/1184893381308084226" target="_blank">juggling</a>. 

### Copying Tables...Again

Since I wasn’t able to take a backup and restore it, I was back to square one, trying to figure out how to quickly move this data. After spending almost the entire year moving tables one by one, the idea of having to do it all over again pretty much drove me to tears, but that’s what I tried next. 

I would be moving from server to server within the same datacenter and rack, so I wanted to see how fast it would be to use various methods to bulk insert the data from the old database to the new one. I tried:

<img src="/image/2020/slow_migrations.jpg" width="300" height="300" style="float:right; padding: 20px;" alt="Slow Migration Time" />

- <a href="https://twitter.com/tarynpivots/status/1185194987488567297" target="_blank">PowerShell method</a> - took <a href="https://twitter.com/tarynpivots/status/1185304145361637377" target="_blank">36 hours</a> to move one table with 4 billion rows
- SSIS version - took 2.5 days to move the same table
- <a href="https://docs.microsoft.com/en-us/sql/t-sql/functions/openrowset-transact-sql?view=sql-server-ver15" target="_blank">`OPENROWSET`</a> - took 16 hours to move 650 million rows

PowerShell was the winner when it came to speed, but Nick Craver wasn’t convinced. He decided to try it. He wrote a little LINQPad script and after running it <a href="https://twitter.com/Nick_Craver/status/1186456942400737281" target="_blank">he finally believed me that moving this much data sucked</a>. 

At this point, <a href="https://twitter.com/tarynpivots/status/1186465114754445313" target="_blank">I was devastated</a> because it looked like I was going to have to migrate everything for a second time. 

### Time to Try an Availability Group

I really, really, really did not want to move hundreds of tables again, so I threw out the idea of using an <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver15" target="_blank">availability group</a> to <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/automatically-initialize-always-on-availability-group?view=sql-server-ver15" target="_blank">automatically seed the databases</a> to the new servers. We already have AGs throughout our infrastructure, so I was familiar with using them, but I was unsure if it would successfully work for a database of this size. 

Before starting, I pinged <a href="https://www.seangallardy.com/" target="_blank">Sean Gallardy</a> and asked if he thought it was possible to automatically seed a database of between 30-40TB - he guessed it'd be slow but possible, maybe taking 1-3 days. I was 100% ok with waiting 1-3 days, if it saved me months of moving tables again. 

I decided to spin up the availability group on the Colorado server first, and if it was successful, I'd move to New York. In order to set this up, I would need to go through all these steps:

- Set up new Windows Failover Cluster with the two servers
- Take another backup, as I couldn't use the first one because the database was in <a href="https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/recovery-models-sql-server?view=sql-server-ver15" target="_blank">simple recovery instead of full</a>. This meant waiting another <a href="https://twitter.com/tarynpivots/status/1186819184249856000" target="_blank">11.5 hours</a> for the backup of the 33TB database in Colorado to finish
- Set up the AG and fingers crossed let it seed
- Initiate failover to the new server
- Destroy AG and shutdown old server

After backup in Colorado finished, I realized that we might have a small problem when trying to autoseed. The <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/automatically-initialize-always-on-availability-group?view=sql-server-ver15" target="_blank">docs explain</a>:

> Automatic seeding requires that the data and log file paths are the same on every SQL Server instance participating in the availability group

The old servers had the data on E: and F:, and the new servers were using D:. To be certain there were no issues, I quickly renamed the drive on the new server to E: and moved the files over using:

{{< highlight sql>}}
use [master]
​
alter database [TrafficLogs] 
modify file(name= TrafficLogs_Current, filename = 'E:\Data\TrafficLogs_Current.mdf');
​
alter database [TrafficLogs] 
modify file(name= TrafficLogs_log, filename = 'E:\Data\TrafficLogs_log.ldf');
{{< / highlight >}}

By changing the assigned letter to the drive, I also had to move `tempdb` to the new drive, but once all that was done it was time to try setting up the availability group.

I spun up the temporary AG in Colorado, waited and watched SQL Server. I was constantly checking both DMVs `sys.dm_hadr_automatic_seeding` and `sys.dm_hadr_physical_seeding_stats` to see the <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/automatically-initialize-always-on-availability-group?view=sql-server-ver15#monitor-automatic-seeding-availability-group" target="_blank">status of the seeding process</a>. I was concerned it would fail and we'd be back to the drawing board. After about 6 hours, it looked like we had seeded about 17TB of the database. 

We also could see from our SignalFX monitoring of network traffic that something was happening. The SignalFX chart below shows we were averaging about 150 - 180 Mb/sec of traffic to a box that wasn't being used by anything except for the database seeding. 

<img src="/image/2020/co_log_transfer_inprogress.png" width="720" height="360" style="padding: 20px;" alt="Network Traffic Flowing" />

By the next morning, the seeding had finished and we had a database with all the tables on the new Colorado server. Progress!!

<img src="/image/2020/opserver_seeding_co.png" width="720" height="360" style="padding: 20px;" alt="We have tables" />

Before initiating the failover, I ran a <a href="https://twitter.com/tarynpivots/status/1187516892170223616" target="_blank">`DBCC CHECKDB`</a> on the new server aka the secondary in the availability group to be sure the database was in a good state, and after <a href="https://twitter.com/tarynpivots/status/1187690141940244480" target="_blank">22 hours</a> it completed with no issues reported.  

It was time for the failover. I was a little nervous about doing it, but we were ecstatic, shocked, and impressed (all-in-one) that it worked.

While all of this work was happening in Colorado, I kicked off <a href="https://twitter.com/tarynpivots/status/1187179353425113089" target="_blank">the process on the New York server</a> since the database was  larger, and I knew it was going to take a long time. After taking <a href="https://twitter.com/tarynpivots/status/1187736811675668480" target="_blank">41 hours to backup the 40TB database</a>, I was ready to follow all the same steps on the New York server. I set up the cluster, availability group, seeded the database, ran a very long `DBCC CHECKDB`, and finished with a successful failover. 

The hardest parts were done, but it wasn't quite time to celebrate. I still had some final work to do. 

### Final Cleanup

<img src="/image/2020/fireworks.jpg" width="360" height="120" style="float:right; padding: 20px;" alt="Celebration Fireworks" />

We finally had the new servers in place with the databases, but we didn’t have the new traffic log data flowing to the new servers quite yet. The service that sends the traffic logs (Traffic Processing Service aka TPS) was still pointing to the old database. This was done purposely, to not interrupt any of the teams using the data. The idea was, once the servers were in place, we could side-by-side release the new version to push data in both places for a period of time (by sending syslog traffic from HAProxy to both servers). This would allow teams to port their processes to the new database and table structures in a gradual way.

The week after the failover, we released the new traffic processing service and I did a few more days of data migration from the old server. I also destroyed the availability group and the windows failover clusters. Once all that was done, it was finally time to celebrate.

## Final Thoughts

During my time as the DBA at Stack Overflow, I’ve had the opportunity to work on various large projects  (some documented <a href="https://www.tarynpivots.com/post/how-we-upgraded-stackoverflow-to-sql-server-2017/" target="_blank">here</a> and <a href="https://www.tarynpivots.com/post/how-stack-overflow-upgraded-from-windows-2012/" target="_blank">here</a>), but none have taken as long as this one. This took <a href="https://twitter.com/tarynpivots/status/1191458709630672896" target="_blank">about 11 months</a> with all of the moving, validating, deleting, and juggling space.

I honestly love taking on these big projects. Planning the entire thing, and taking it from start to finish is extremely gratifying. Even though I automated the majority of the work, this project was incredibly frustrating and I definitely don’t want to do it again anytime soon (unless I have plenty of drive space and don’t  have to juggle things for 11 months). This project was challenging in many ways, and considering I had to do the same work on two servers it was doubly challenging. In the end, I was very relieved to successfully use an availability group to seed the databases to the new servers, and finally be done with moving all the things.

Have you ever had to move multiple databases of this size? Were you lucky enough to have extra disk space to get it done? I’d be curious how others would have solved the problem of moving about 40TB of data (twice) with minimal free disk space? If you have thoughts, let me know.


