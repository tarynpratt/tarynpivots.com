---
title: 'Hitting Parallel_Redo_Flow_Control waits with Availability Groups'
date: Mon, 09 Dec 2019 06:00:38 +0000
draft: false
tags: [sql, sql-server, stack-overflow, distributed-availability-group, availability-group, waits, parallel-redo-flow-control, outage, maintenance, wait-stats]
excerpt: "Index maintenance job caused excessive parallel_redo_flow_control waits in an Always On Availability Group resulting in an outage"
---

In late June 2019, <a href="https://twitter.com/StackStatus/status/1143971084623941632" target="_blank">June 26th</a> to be exact, we experienced an outage on Stack Overflow for about 11 minutes. It's not unusual that we had an outage. They happen. Not often, but they do still happen. This one, however, was a little different because it was caused by a maintenance job that was running on our primary SQL Server for Stack Overflow. 

The job that caused it was something I'd noticed about a month prior, but <a href="https://twitter.com/tarynpivots/status/1130962290033844224" target="_blank">had stopped it</a> before an actual outage occurred. Once corrected, I didn't see any sign of it again, so I figured we were fine. I was wrong. This time around, the job triggered the same issue and unfortunately, took us down for longer than I would like. 

You're probably thinking &mdash; wait a minute, a maintenance job triggered an outage? What type of job did that?

## Hang On, Please Explain Yourself

While I can't be 100% sure of the trigger, I'm 99.9% sure, because the job was running before the outage, so the timing is right. After looking through our monitoring logs, everything pointed to the job being the cause, so yes, I'm confident it caused it.

We don't have regular maintenance windows for any of our servers, so we run jobs throughout the week, and if possible, try to schedule them during low-usage times. In this case, the job  was an index maintenance job. 

Now, before you scream at me about running an index maintenance job, I'm not going to argue the pros and cons of using it or whether or not we should run it &mdash; we can do that at another time. For this post, just accept the fact that we were running a job to rebuild/reorganize indexes.

### Some Background

We run <a href="https://docs.microsoft.com/en-us/sql/sql-server/editions-and-components-of-sql-server-2017?view=sql-server-ver15" target="_blank">SQL Server 2017 Enterprise Edition</a> which gives us the ability to rebuild indexes online, but we never had a scheduled job for it. If an index needed to be rebuilt, we'd manually do it. Ideally, we would automate this and target the indexes suffering from the most fragmentation, rebuild it, and move on to the next one.  

After I [finished the OS upgrade of our main servers](/post/how-stack-overflow-upgraded-from-windows-2012/) we were in a better place to try running an index optimize job on a regular basis. We waited until after the upgrade from Windows Server 2012 because of the improved throughput we would get with our secondaries. Once the upgrades were done, we knew we could try to run the job more often. (Side note: why we waited until after we did the upgrade to run the index job is a story for another time, but let's just say it caused a different outage from running out of disk space.)  

Using most of <a href ="https://ola.hallengren.com/" target="_blank">Ola Hallengren's maintenance solution</a> across our servers, I set up the Index Optimize job to look for specific fragmentation levels, and depending on the server, I spaced out the execution frequency. I needed to give enough time in between executions for the logs to transmit to our secondaries. On some servers, I scheduled the job to run hourly, and on others it was scheduled to run every two hours. 

In addition to frequency, I also had to be mindful of how long they ran. For example, running the job on the server that holds the Stack Exchange network of databases (~375 databases), would take hours to run through all the indexes to be defragmented. For that server, the job was scheduled to look for indexes with X% fragmentation and would run for 5-10 minutes. This spacing prevented overloading the transaction logs in between log backups, and would help minimize what needed to be transported to each of our secondaries. 

My thinking was, we'd be far more efficient biting off smaller chunks of changes 5-10 minutes at a time, versus running it for many, many hours, and generating tons of log data that needed to be pushed to the secondaries. Doing this in bits also allowed our remote secondaries in Colorado to keep up with the transaction log backup. If I ran the job for too long, Colorado would fall further and further behind while trying to write the transaction log, so it made far more sense to do this a little at a time. 

## Enough of the Background, How Did you Break Production?

About a month before the outage, I caught the issue and stopped it before we got to the level of the outage...well, that didn't happen this time. 

I'm going to take another step back and give a bit more background.

As mentioned in some of my other posts, we use <a href="https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server?view=sql-server-ver150" target="_blank">SQL Server's Always On Availability Groups</a>. Most of our clusters have 3 servers - one primary, one local secondary, and one remote secondary. In our primary datacenter - we have two servers - the primary receives write and read traffic, while our local secondary receives a lot of the read-only traffic. 

We offload a lot to our readable secondary (at some point, if there's interest, I'll provide an overview of what we do). The secondary, in both of our main clusters, is used for many of our scheduled jobs and many of these scheduled jobs run hourly from across our web tier. 

### Outage Timeline

The timeline of the events will give a bit of insight into what exactly happened when we went offline. Using our monitoring tools, I was able to determine the timeline as:

- The index maintenance job ran on the primary server
- The job rebuilt/reorganized an index against a specific table
- The index rebuild was in progress of being replayed to the local secondary - the NY secondary server
- As the change to the index was in the process of being written to the secondary, the scheduler triggered a job which needed to query the secondary and the table where the log changes being written to 
- The scheduler hit blocking when attempting to query the table on the secondary server because the index rebuild was still writing to it
- The replay of the index rebuild was running just as the scheduler attempting to query the same table, resulting in a significant number of sessions being blocked
- The secondary was overloaded with queries attempting to query the table (both the scheduled queries and normal traffic against that table) and we basically fell over due to blocked sessions resulting in connection pool exhaustion upstream

The outage was the result of a domino effect, but the starting domino was the index maintenance job. 

The size of the index, and the criticality of the table also added to the issue, but the redo to the secondary while we were trying to query it pushed us over the edge and the server we couldn't keep up. This resulted in hundreds of sessions blocked. Since nothing could query the required tables, we went offline.

As soon as we started to hit the blocking sessions, I received an alert from our instance of <a href="https://www.sentryone.com/products/sentryone-platform/sql-sentry/sql-server-performance-monitoring" target="_blank">Sentry One SQL Sentry</a> that there was blocking on the server. I quickly opened the email and saw that we were hitting <a href="https://blogs.msdn.microsoft.com/sql_server_team/sql-server-20162017-availability-group-secondary-replica-redo-model-and-performance/" target="_blank">`PARALLEL_REDO_FLOW_CONTROL` waits</a>. The Microsoft Docs explain that this wait

> Occurs when the main redo thread cannot dispatch more transaction log records when log cache array for dispatched transaction log is full.

> Indicates that one or more parallel redo worker threads cannot keep up with main redo thread transaction log dispatching speed or are blocked by some resources such as other type of waits. When this wait occurs frequently, parallel redo worker threads does not function efficiently

It makes sense that this was our issue. We were overloading the server with the underlying change to the index. The transaction log was very large during this time, and was trying to write to the secondary server. At this exact same moment, we also needed to query that table for scheduled jobs. Since the table was unavailable for use, the application could not continue and we went offline. 

Since I saw the behavior in the previous month, I knew what to do. I quickly connected to the secondary server and ran <a href="http://whoisactive.com/" target="_blank">Adam Machanic's `sp_whoisactive`</a> to see all the sessions attempting to query, and see what was being blocked. Once I had that list, it was time to `kill` all the sessions. I needed to give a bit of time for the index rebuild to write to the server, and by killing the sessions it was just enough to let things settle down. Unfortunately, we were down for about 11 minutes, but between the monitoring from Sentry One and seeing the issue pop up the month before, I knew that I would give us some breathing room by killing `spids`. Once the sessions were killed (helping unblock application connection pools as well), the log was able to write to the server and we came back up. 

While this explains how we fixed the outage, that's not the point of this post. 

## Then, What is the Point of This?

If you run Always On Availability Groups, and you use a readable secondary, you need to be mindful of the issues with `PARALLEL_REDO_FLOW_CONTROL`. These waits resulted in enough blocking to take us offline. 

Yes, I'm aware that there is a <a href="https://docs.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql?view=sql-server-ver15" target="_blank">trace flag 3459</a> that we can turn on, but we don't have that enabled for our Stack Overflow SQL Server. Most of the time, it doesn't make sense for us to disable parallelism (due to scale), so we don't. 

In order to prevent this from happening again during the index maintenance, I moved the impacted database to a separate index maintenance job that runs only on the weekend, when we have lower traffic and less opportunities to collide with other jobs. Once I moved that database to a separate job, we stopped hitting blocking issues with `PARALLEL_REDO_FLOW_CONTROL` and that job.

This solution works for us, but your situation might be different. If you run Always On Availability Groups, and you utilize a readable secondary, be mindful of those `PARALLEL_REDO_FLOW_CONTROL` waits, since they can result in significant blocking for an application reading from the secondary. In our case, adjusting when we ran our job helped especially since we don't use the Trace Flag 3459, but that trace flag might work for you. My recommendation would be to test different situations with your AGs and maintenance jobs to see what works. 

Have you hit this same issue? Did you fix it using the trace flag or some other way? 