---
title: Recovering Lost Linked Servers
date: 2020-02-06 06:00:38 +0000
draft: false
tags: [sql, sql-server, stack-overflow, server-migration, server-upgrade, windows-server-2016, maintenance, projects]
excerpt: How I retrieved linked servers that were not scripted out before an upgrade.
---

Recently, I kicked off a project to start moving us to [SQL Server 2019](https://docs.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-ver15?view=sql-server-ver15). During my initial review of our servers, I found quite a few (9 total) that were still running on Windows Server 2012 R2. This meant that I would need to upgrade the operating system and move us to SQL Server 2019. Having completed plenty of [SQL Server](https://www.tarynpivots.com/post/how-we-upgraded-stackoverflow-to-sql-server-2017/) upgrades, as well as [operating system upgrades](https://www.tarynpivots.com/post/how-stack-overflow-upgraded-from-windows-2012/), I couldn't possibly make a mistake, right? Wrong...I completely forgot to script out the linked servers on the server I upgraded this week. I screwed up and decided to write about how I went about fixing it.

## My Plan

This week, I [tackled the server](https://meta.stackexchange.com/q/342991/164200) running [Stack Exchange Data Explorer (SEDE)](https://data.stackexchange.com/). In the past, when I have upgraded an operating system on a SQL Server I work off a list:

1. Take a backup of all system databases - if I'm reinstalling the same version of SQL Server, I could restore the `master`, `model`, and `msdb` databases to minimize some of my setup again
2. Script out any objects in the `master` that we need, this would be tables and stored procedures. We store objects for maintenance purposes in `master`...please don't scream at me.
3. Generate creation scripts for all jobs
4. Export logins and users from the server 
5. Make sure I have everything else I need scripted before kicking off a rebuild

## Upgrade Day

Before starting the rebuild, I did one final update of SQL Server 2017 to bring it to the latest patch we were using, [CU 18](https://support.microsoft.com/en-us/help/4527377/cumulative-update-18-for-sql-server-2017). This was to make sure it was fully up-to-date before I rebuilt the server. 

You might be curious why I patched SQL Server 2017 when we were going to SQL Server 2019? I did this because the server that runs data explorer has a lot of specific permissions, and it also has about 350 databases. My initial goal was to restore the system databases to minimize the risk of missing a permission or something in the shuffle when setting up the new server. Once the system databases were restored, I'd upgrade to SQL Server 2019.

After patching SQL Server, I checked all the steps above, and thought I had everything I needed to recreate the server, so I kicked off the rebuild. About 30 minutes into the process, I had a total facepalm moment and realized I forgot to script the linked servers. It was too late to do anything about it at that time, but thankfully I had the system databases which would save me. Once Windows Server 2016 was done installing, it was time to install SQL Server 2017 and then _attempt_ the restore of the system databases.

I say _attempt_ to restore the system databases because while I've restored the `master` database before, [it's not fun](https://twitter.com/tarynpivots/status/1225131722397741056), or at least, not my definition of fun. The biggest pain point I have is getting the single connection to the server. I know how to put the server in [single user mode to restore the `master` database](https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-the-master-database-transact-sql?view=sql-server-ver15), but at Stack Overflow we have a lot of monitoring systems that are actively checking the server status. Unless I disabled all those logins, it'd be nearly impossible to get that single connection. 

I opened up a command window and started going through all the steps to restore the `master` database. I executed:

    net stop MSSQLSERVER
    net start MSSQLSERVER /m

Then, I opened another command window and connected to the instance by using:

    sqlcmd -s .\MSSQLSERVER

When I hit go, I received the dreaded error:

> Login failed for user `<xxx>` Reason: Server is in single user mode. Only one administrator can connect at this time.

Boo. I tried connecting a few more times and each time I couldn't get the single connection. I disabled some logins for our monitoring services and couldn't connect to the instance because something was getting the single connection first. I may also have been hitting my head on my desk out of frustration. After fighting with it for far longer than I'm going to admit, I gave up. I knew I was going to just have to rebuild the server from my scripts, and attempt to recreate the linked servers from scratch. Knowing it was inevitable, I went ahead and proceeded with the upgrade to SQL Server 2019. When that was done, I executed all my scripts, got the databases back in place, and released [data explorer](https://twitter.com/tarynpivots/status/1225164597822246912) back to the world.

The only things still missing were the linked servers. While I was happy to have moved the SQL upgrades along, I was [very angry with myself for missing that script](https://twitter.com/tarynpivots/status/1225216451746775042) in the first place. The server had not received much love, and I didn't have the linked server, which was created long before me, scripted out and uploaded to GitHub (yeah, I know that was also dumb). I setup the linked servers by hand but couldn't get them to connect. After a very long day I knew I wouldn't need the linked server until the weekend, so I left it to be fixed the next morning. 

## Recovering the Lost Linked Servers

The next day, I had to get the linked servers working. My attempts to create them by hand failed miserably. I had the system backups from before the upgrade, so I wanted to try to get the details out of them. I did several things that didn't work, including restoring the `master` database to a different server, under a different name (i.e. `master_old`) and trying to query the linked server info. 

Finally, I decided to create a new test server, install SQL Server 2017, and restore the `master` database to it. I setup a new build process to create a VM with a single disk drive, with just enough space for Windows and SQL to be installed on it. I installed SQL Server 2017 to the same CU I needed, and it was time to try the restore again. My thinking was, I could pretty much do anything to this SQL instance in order to get to the linked server data I wanted. 

Using the same commands as above, I set the instance to single user mode, and then attempted to connect to the instance. No error message this time. I was in the instance as the sole user...success. Now that I was connected it was time to restore:

{{< highlight sql>}}
RESTORE DATABASE master FROM DISK = 'C:\Backups\master.bak' WITH REPLACE
{{< / highlight >}}

I received no errors. It looked like everything worked. I opened up the SQL Server Configuration Manager, started the SQL service, and refreshed the page. The SQL service stopped. Uh oh, something is wrong. I went to the error log and popped it open. There was a giant error:

> 2020-02-06 16:42:24.74 spid9s      Starting up database 'model'.

> 2020-02-06 16:42:24.74 spid9s      Error: 17204, Severity: 16, State: 1.

> 2020-02-06 16:42:24.74 spid9s      FCB::Open failed: Could not open file C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\DATA\model.mdf for file number 1.  OS error: 3(The system cannot find the path specified.).

> 2020-02-06 16:42:24.74 spid9s      Database 'model' cannot be opened due to inaccessible files or insufficient memory or disk space.  See the SQL Server errorlog for details.

> 2020-02-06 16:42:24.74 spid9s      SQL Server shutdown has been initiated

> 2020-02-06 16:42:24.74 spid9s      SQL Trace was stopped due to server shutdown. Trace ID = '1'. This is an informational message only; no user action is required.

I saw the issue right away. The server was looking for the `model` database in the `MSSQL12.MSSQLSERVER` directory, but that didn't exist any longer. The old server had it's files in the directory for SQL Server 2014 aka MSSQL12, but on this new server everything is located in the directory for `MSSQL14.MSSQLSERVER`. As I mentioned, I didn't really care what I did to this test machine, so I manually created the directory that it was looking for - `C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\DATA` and made sure that the SQL user account had access to the directory. Then I just copied and pasted the `model` database from  `MSSQL14.MSSQLSERVER` and dropped it into the `MSSQL12.MSSQLSERVER` directory.

I attempted to start the SQL service, and again it failed. Back to the error log, I went. This time the error was:

> 2020-02-06 16:48:42.11 spid9s      Clearing tempdb database.

> 2020-02-06 16:48:42.11 spid9s      Error: 5123, Severity: 16, State: 1.

> 2020-02-06 16:48:42.11 spid9s      CREATE FILE encountered operating system error 3(The system cannot find the path specified.) while attempting to open or create the physical file 'D:\Data\tempdb.mdf'.

> 2020-02-06 16:48:42.12 spid9s      Could not create tempdb. You may not have enough disk space available. Free additional disk space by deleting other files on the tempdb drive and then restart SQL Server. Check for additional errors in the operating system error log that may indicate why the tempdb files could not be initialized.

> 2020-02-06 16:48:42.12 spid9s      SQL Server shutdown has been initiated

> 2020-02-06 16:48:42.12 spid9s      SQL Trace was stopped due to server shutdown. Trace ID = '1'. This is an informational message only; no user action is required.

There was some progress because it was a different error, right? 

Now it's complaining because I don't have a `D:\` drive on my test instance. I built the server with a single drive, but luckily I can easily fix that for a VM. I logged into our VM platform, quickly added a tiny new disk drive, then formatted it on my test server. Once the `D:\` drive was available, I created `D:\Data` and granted permission to the SQL user account to access the directory. 

It was time to go back to the SQL Server Configuration Manager to start the service. As I kicked the service to start, I kept thinking 'please work, please work'. This time the service started and stayed running. I logged into SSMS, connected to the instance, and went looking for my Linked Servers. 

Guess what? They were there. Somehow it worked. I was able to script out both linked servers and execute them where they belong. After some testing, I confirmed that the linked servers were working on the original server I built. 

It's quite possible that there are other ways to recover something like this. I'm also [relieved](https://twitter.com/tarynpivots/status/1225475415822589952) this worked and I know I'm lucky to have the tools to be able to quickly spin up a new test SQL Server to hack away on. I learned a valuable lesson - verify I have everything that I need when rebuilding servers. 

Oh, and by the way, yes I did upload those linked server creation scripts to GitHub. 





