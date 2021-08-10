---
title: My SQL Server Toolbox
date: 2020-10-15 04:00:00 +0000
draft: false 
tags: [sql, sql-server, stack-overflow, performance-tuning, monitoring, monitoring-tools]
excerpt: A handy list of the tools I use when working with SQL Server. 
---

I get asked a lot about the tools I use at Stack Overflow to monitor and work with our SQL Servers. I figured it might be helpful to others to make the list public. I'll also do my best to keep it updated as things change.

The current list of both free and 3rd party paid tools is below:

## Free Tools

- [Opserver](https://github.com/opserver/Opserver) - Stack Exchange's Monitoring System - I pretty much live in our instances of Opserver because it gives me a one-stop shop to see the health of all of our SQL Servers 

- Brent Ozar's [First Reponder Kit](https://www.brentozar.com/responder/). This is filled with lots of goodies to help you understand what's going on with your server. Currently we install these scripts from the kit:

    - sp_Blitz.sql
    - sp_BlitzCache.sql
    - sp_BlitzFirst.sql
    - sp_BlitzIndex.sql
    - sp_BlitzLock.sql
    - sp_BlitzQueryStore.sql

- Adam Machanic's [sp_whoisactive](http://whoisactive.com/) - this is similar to executing `sp_who2` but provides a more detailed look at what is happening on your server

- [sp_hexadecimal and sp_help_revlogin](https://support.microsoft.com/en-us/help/918992/how-to-transfer-logins-and-passwords-between-instances-of-sql-server) to transfer logins between instances of SQL Server

- Ola Hallengren's [SQL Server Maintenance Solution](https://ola.hallengren.com/) for Backups, Integrity Checks, and Index and Statistics Maintenance. We don't use the entire package of scripts, but we currently use:

    - DatabaseIntegrityCheck.sql 
    - IndexOptimize.sql 
    - CommandExecute.sql 
    - CommandLog.sql 
    - DatabaseBackup.sql - we use a modified version of this script everywhere

- [dbatools](https://dbatools.io/) PowerShell modules 

- Erik Darling's [SQL Server Troubleshooting Scripts](https://github.com/erikdarlingdata/DarlingData) - additional tools to help you identify and troubleshoot problems with your SQL Servers

- Columnstore Maintenance Scripts from a few places:

    - Joe Obbish's [cci_maintenance](https://github.com/jobbish-sql/SQL-Server-Multi-Thread)
    - Nike Neugebauer's [Columnstore Indexes Scripts Library](https://github.com/NikoNeugebauer/CISL)


## 3rd Party Tools

- SentryOne [Plan Explorer](https://www.sentryone.com/plan-explorer) - this is a great tool to review execution plans. I find it far easier to review execution plans in Plan Explorer than in SQL Server Management Studio

- SentryOne [SQL Sentry](https://www.sentryone.com/products/sentryone-platform/sql-sentry/sql-server-performance-monitoring) monitors all of our SQL Servers and provides excellent details on performance and insight into specific queries that might be running during various CPU spikes or other periods of concern

- ApexSQL [SQL script generation](https://www.apexsql.com/sql-tools-script.aspx) - there are times we need to rebase our SQL migrations and this tool is a great way to script out all objects and data 


All the SQL scripts get deployed via process that I wrote. I'll share how I do that in my next post. 