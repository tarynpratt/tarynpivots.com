---
title: 'Using powershell to fix server space issues'
date: Sun, 12 Apr 2015 18:30:08 +0000
draft: false
tags: [development, powershell, Projects, SQL, sql-server, Work Adventures]
excerpt: "Using Powershell with SQL Server"
---

A few weeks ago, we were running into severe disk space and memory issues on our development servers at work. Our set-up is a bit odd, we have 3 servers - one for the transactions, one for the web interface, and the final one for reporting. Using [transactional replication](https://msdn.microsoft.com/en-us/library/ms151176(v=sql.110).aspx) we have databases that can exist on all 3 servers. Yes, it's can be a real nightmare to maintain, but anyone who works with replication already knows this. Each development team gets its own version of production "in a box". These copies are on VM slices with limited memory and disk size.

Our issues were happening on the web database and reporting servers. We have replication and sql jobs running, development teams testing, and a variety of other things hitting the servers pretty hard and no memory to process it, so we were running out of disk space. There were several days in a row that I noticed we were down to 20MB of space on our web database server. This lack of resources was causing replication to fail throughout our environment, which resulted in delays to our current sprint.

I'm the early bird on my team and every morning I was executing the following code to "clean-up" our environment via shrinking the log files. Eeeek, the [dreaded shrinking](http://stackoverflow.com/q/56628/426671) of [log files](http://www.brentozar.com/archive/2009/08/stop-shrinking-your-database-files-seriously-now/). I know you don't want to do this really ever, but it's a development environment and it's how we handle our limited memory issues.

My first check to was see the size of the logs by running [`DBCC SQLPERF (LOGSPACE)`](https://msdn.microsoft.com/en-us/library/ms189768.aspx). This command gives the transaction log space for all of the databases on the server. Once I got the list I was able to target the databases that had excessively large log files in order to shrink them.

Next, I'd get the name of the log file to shrink via:

{{< highlight sql>}}
SELECT name
FROM sys.master_files
WHERE database_id = db_id()
	AND type = 1
{{< / highlight >}}

Lastly, using the name of the log above I'd run [`DBCC SHRINKFILE`](https://msdn.microsoft.com/en-us/library/ms189493(v=sql.110).aspx) to drop the size of the logs and restore a bit of memory on our box.

Doing this first thing in the morning for multiple databases on multiple servers for multiple days in a row was terrible. So, I decided I needed to automate this. Considering I learned a bit about [Powershell from Mike Fal at SQL Saturday Phoenix](http://onlybluefeet.com/2015/04/12/sql-saturday-adventures/), I thought "hey, let me take a stab at writing my first Powershell script to do this for me!!"

To be honest, I wasn't exactly sure where to start so I asked Mike for a bit of help on looping through all databases on a server. He was nice enough to give me a [starting script](http://pastebin.com/xNRZMGDm). Using this, I attempted to incorporate my code above into it with the purpose of shrinking all log files on the server. I'm sure there are much better ways to do this but here is my first Powershell script.

{{< highlight powershell>}}
Import-Module sqlps -DisableNameChecking

$dbs = Invoke-SqlCmd -ServerInstance 'servername' -Query "select name from sys.databases where database_id > 4"

foreach($db in $dbs){
	$Log = Invoke-Sqlcmd -ServerInstance 'servername' -Database $db.name -Query "select name from sys.master\_files where database\_id = db_id() and type = 1"
	$LogName = $Log.name
	$query = "DBCC SHRINKFile($LogName, 1)"
	Invoke-Sqlcmd -ServerInstance 'servername' -Database $db.name -Query $query
}
{{< / highlight >}}

This does exactly what I need it to do. I can set it up on our environments to run nightly to shrink the logs so I no longer need to manually "fix" stuff every day. I've passed it along to our engineering services group to set this up in all of the development environments when they create them with new production copies.

The script is available on [GitHub](https://github.com/onlybluefeet/posh_stuff/blob/master/Shrink_Logs.ps1) if anyone is interested in using it, or tweaking it, or criticizing it, etc.