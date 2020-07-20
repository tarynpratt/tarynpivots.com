---
title: SQL Server 2019 Tuple Mover Behavior Change
date: 2020-07-20 04:00:00 +0000
draft: false
tags: [sql, sql-server, sql-server-2019, server-upgrade, maintenance, clustered-columnstore, tuple-mover]
excerpt: Change to tuple mover behavior in SQL Server 2019
---

This is a follow-up to <a href="https://www.tarynpivots.com/post/aggressive-clustered-columnstore-cleanup/" target="_blank">my post about an issue with clustered columnstore</a>, when upgrading from SQL Server 2017 to SQL Server 2019. After extensive testing and working with support, I wanted to share some information about a change in SQL Server 2019 that might impact others.

## Overview

I suggest reading <a href="https://www.tarynpivots.com/post/aggressive-clustered-columnstore-cleanup/" target="_blank">my other post first</a>, it'll only take a few minutes. I'll wait...
 
However, if you really don't want to read it, here's a quick recap on the initial issue.
 
In early February 2020, a lot of data was deleted from some <a href="https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-described?view=sql-server-2014" target=”_blank”>clustered columnstore indexes</a> in our PRIZM database. Some of the tables were rebuilt, but 11 tables weren't since we don't have maintenance windows, and that would involve downtime. The rebuilds would happen once we upgraded to SQL Server 2019, to take advantage of the ability to rebuild those columnstore indexes online.
 
The day we upgraded to SQL Server 2019, there was rapid growth of the transaction logs for PRIZM. The only way to get the growth to stop, was to enable both <a href="https://docs.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql?view=sql-server-ver15" target="_blank">trace flags 634 and 661</a>.

|  Trace Flag |   Definition   |
|-------------|----------------|
|    634      | Disables the background columnstore compression task. SQL Server periodically runs the tuple mover background task that compresses columnstore index rowgroups with uncompressed data, one such rowgroup at a time. |
|    661      | Disables the ghost record removal process. For more information, see this <a href="https://support.microsoft.com/kb/920093" target="_blank">Microsoft Support article.</a> |

After working with support, we were able to remove trace flag 661, but still had 634 in place, so more work needed to be done.

## Enabling Another Trace Flag

Removing trace flag 634 from the production server was the next step, but <a href="https://www.tarynpivots.com/post/aggressive-clustered-columnstore-cleanup/#oh-tuple-mover-what-are-you-doing" target="_blank">every attempt</a>, led to the rapid increase in the transaction logs and a spike in query timeouts on the server. 
 
After passing lots of details to support, they requested we enable an undocumented trace flag - 10257 - to see if we still experienced the transaction log growth. Since this trace flag isn't documented, the description we were given is:
 
> 10257 - disables the columnstore tuple mover background merge process. The tuple mover background merge runs a reorganize on a few rowgroups at a time with more conservative thresholds.
 
Time for more testing!!
 
I removed trace flag 634, and enabled trace flag 10257 to see how the transaction logs responded. The swap was done for about an hour and during this time the transaction logs didn't have the rapid growth, but there still was an increase in query timeouts. 

The rowgroup states also changed:

| Rowgroup State | Before with TF 634 (only) | After with TF 10257 (only) |
|----------------|--------------------|----------------------|
|  Compressed    |  9844 | 9927 |
|    Closed      |  261 | 194 |
| Open | 721 | 721 |
| Tombstone | 48 | 68| 

The change in the numbers showed the tuple mover was running, but with the background merge process disabled there wasn't the rapid growth of the logs (which was a good thing). However, we had to deal with query timeouts when trace flag 634 was removed. Now what?

## More Investigating

I wanted a backup from before the upgrade to do two things:
 
1. Check the state of the rowgroups before our initial upgrade. I was hoping to see how many were tombstones, open, closed, compressed?
2. Could I recreate the issue by upgrading a server from SQL Server 2017 to SQL Server 2019?
 
It had been a month since we upgraded our SQL Servers, which meant our backups were only available offsite. After a <a href="https://twitter.com/tarynpivots/status/1255883993670578176" target="_blank">little fighting</a>, I was finally able to pull it from storage.
 
I initially restored the database to one of our existing test servers with SQL Server 2019 on it. I wanted to see the state of the rowgroups on the tables that had not been rebuilt. Did we have hundreds of tombstones or not?  Unfortunately, after restoring the database <a href="https://twitter.com/tarynpivots/status/1256012151447158784" target="_blank">it didn't show me what I was looking for</a>. Based on a suggestion from Andy Mallon (<a href="https://am2.co/" target="_blank">b</a> | <a href="https://twitter.com/AMtwo" target="_blank">t</a>), I tried to <a href="https://twitter.com/AMtwo/status/1256017346449342465" target="_blank">restore with `STANDBY`</a> to see if that made any difference...it didn't. I was able to get stats, but it didn't have the rowgroups in a tombstone state.
 
Next, I decided to restore the database to a server with SQL Server 2017 on it, but since I already upgraded all of our servers to 2019, one wasn't readily available. Thankfully, I have the ability to spin up test servers as needed, so that's what I did.
 
After restoring the database to the 2017 SQL Server, it had the following:
 
- No ghost records
- No TOMBSTONE rowgroups
- 666 OPEN rowgroups
- 0 closed rowgroups
- The transaction log was approximately 19.5 GB
 
Hmmm, ok there was nothing too concerning in the restored database. No red flags or alarm bells. It was time to upgrade to SQL Server 2019 to see what happens.
 
Immediately after upgrading to SQL Server 2019, everything appeared ok. Then, I waited about 30 minutes to capture the stats again. After 30 minutes on 2019, we had:
 
- 24k ghost records
- 79 TOMBSTONE rowgroups
- 666 OPEN rowgroups
- 0 closed rowgroups
- and a transaction log that is 43GB
 
Initially, I didn't have either trace flag (634, or 661) enabled. Once the trace flags were enabled, the growth of the transaction logs stopped. Welp, I had reproduced the issue.
 
Support also asked me to grab the results of the following query when the database was still on SQL Server 2017, and again, after I upgraded to SQL Server 2019 to get the status of deleted rows:


{{< highlight sql>}}
select
    i.name,
    rg.object_id,
    rg.partition_number,
    rg.row_group_id,
    rg.delta_store_hobt_id,
    rg.state_desc,
    rg.total_rows,
    rg.deleted_rows,
    100.0*(ISNULL(rg.deleted_rows,0))/total_rows AS 'Fragmentation',
    rg.created_time,
    rg.closed_time
from sys.dm_db_column_store_row_group_physical_stats rg
inner join sys.indexes i
    on rg.object_id = i.object_id
    and rg.index_id = i.index_id
order by i.name
{{< / highlight >}}

I passed this info along to them, and we were finally able to put it all together.  

## Putting the Pieces Together

Now that we had more data points, we could step back and look at the big picture, which allowed us to determine what caused the problem in the first place, and how to prevent it going forward.

### The Root Cause

The deletions we did in early February were the start of the problem. Without having maintenance windows, we were unable to rebuild the tables, leaving rowgroups in a highly fragmented state with a lot of deleted rows - many of them had 100% deletion. We didn't see an issue in SQL Server 2017, because the tuple mover only triggered when a rowgroup closed and the tuple mover moved it to a compressed state.

### Behavior Change

In SQL Server 2019, the tuple mover has a helper called the background merge task. The <a href="https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-query-performance?view=sql-server-ver15" target="_blank">docs have some details about it</a>:
 
> Starting with SQL Server 2019 (15.x), the tuple-mover is helped by a background merge task that automatically compresses smaller OPEN delta rowgroups that have existed for some time as determined by an internal threshold, or merges COMPRESSED rowgroups from where a large number of rows has been deleted. This improves the columnstore index quality over time.
 
> If deleting large amounts of data from the columnstore index is required, consider splitting that operation into smaller delete batches over time, allowing the background merge task to handle the task of merging smaller rowgroups and improve index quality, eliminating the need to schedule index reorganization maintenance windows after data deletion.
 
In other words, the new background merge task has benefits, such as, helping the tuple mover clean up smaller rowgroups to improve the overall quality of the index. This task also automatically kicks in to merge compressed rowgroups where the number of deleted rows hits an internal threshold...kinda sounds a little like what we had.

### The Log Growth Issue

We experienced rapid transaction log growth the day we upgraded to SQL Server 2019 because the background merge task was triggered by the deleted rows we had in various tables. The log growth was due to the merge task trying to free the space in the rowgroup. 

Since we had a lot of rowgroups with deleted rows, the merge task was aggressively cleaning them up, which led to exponential growth of the  logs. By enabling trace flag 634, we disabled the tuple mover process which stopped the logs from growing. If we wanted to keep the tuple mover enabled, then we'd need to use the new trace flag 10257 to disable the new background merge task.

Initially, the new background merge task was extremely aggressive, which is why we saw so many query timeouts. There were a lot of rowgroups that needed cleaning, so the process ran all the time, resulting in contention with our normal load on the server. 
 
We never saw this behavior in any of our testing because we weren't deleting data from our development environments.

### The Resolution

In order to remove trace flag 634 from production, we had a couple of options:
 
1. Disable the trace flag and let the tuple mover finish its business
2. Rebuild the tables
 
We went with option 1. 

I disabled the trace flag and monitored the transaction log which took several hours to finish. During which, we experienced minimal log growth. We also saw query timeouts, but these dropped significantly as the number of rowgroups being processed by the background merge task steadily dropped to zero. Eventually, we had no rowgroups that needed to be cleaned up, which meant no further log growth, and no additional query timeouts from the tuple mover background merge task.

We finally resolved this issue that lingered from the day we upgraded.

## Avoiding this Issue 

If you're planning on moving to SQL Server 2019, and you have clustered columnstore indexes, and deleted data from them, then check the total number of deleted rows. If you have a lot, plan to perform maintenance before upgrading, or to utilize the trace flags to keep the tuple mover in check while cleaning the tables.
 
Don't be like us, especially if you delete a lot of data from your clustered columnstore indexes.
 
A critical thing to remember with SQL Server 2019 is, if you need to delete data from your columnstore indexes, do it in smaller batches and trickle deletes. This allows the background merge task to perform the clean up slowly, and minimizes some of the pain we experienced when upgrading.

As a reminder, before enabling any trace flags on your server (especially in production) make sure you know what you're doing, or, you're working with support and they suggest using it. The new background merge task is there to assist the tuple mover in keeping columnstore indexes in a better state. In other words, ideally, you'll leave it alone and let it do its job. If you're performing maintenance on your columnstore indexes, the background merge task should not negatively impact you, and should actually help with index quality.

## Final Thoughts

I tested SQL Server 2019 for months before we upgraded, and didn't see this behavior at any point. But, as many of us know, upgrades are always fun. There is always a risk of issues when installing new versions. It doesn't matter how much you test or plan, there's always a possibility of something going wrong.
 
I hope this helps someone who is thinking about upgrading, uses clustered columnstore, and doesn't have an easy way to perform maintenance in their production environment. Check those deleted rows, and test, test, and test again, if you're planning on upgrading.

At some point soon, I'll put pen to paper or fingers to keyboard and start writing about the upgrade to SQL Server 2019, along with what benefits we've gotten from it so far. 

I also would like to personally thank Pedro Lopes (<a href="https://twitter.com/SQLPedro" target="_blank">t</a>), for helping with this issue and others we encountered after upgrading to SQL Server 2019, as well as reviewing this post.
