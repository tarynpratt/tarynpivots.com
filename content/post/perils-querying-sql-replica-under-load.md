---
title: 'The Perils of Querying SQL Server Replicas Under Load'
date: 2020-01-15 06:00:38 +0000
draft: false
tags: [sql, sql-server, bug, stack-overflow, availability-group, secondary-server, readable-secondary, distributed-availability-group]
excerpt: "Slow transaction log redo causes incorrect data beiing queried from secondary SQL Server"
---

Last week at Stack Overflow we had an internal hack-a-thon, or as we call it, a make-a-thon. I was on the bug-bashing team, which is the team that attempts to fix smallish bugs we haven’t gotten around to fixing, due to other time-constraints. I was [tagged to investigate a bug](https://meta.stackoverflow.com/q/384675/426671) about duplicate badges being awarded because it looked to possibly be an easy fix in SQL. At first glance it looked simple enough, but once I started digging in, I figured out very quickly it wouldn’t be.

## A Little Background

If you're not familiar with [badges](https://stackoverflow.com/help/badges) on Stack Overflow, they are awarded for performing actions on the site. For example, the badge in the bug report, [Suffrage](https://stackoverflow.com/help/badges/804/suffrage) is awarded for voting 30 times in a day. Some of our badges can be awarded multiple times while others are awarded only once. The Suffrage badge is supposed to be a one time badge, which is why it is odd that someone had it twice.

## Initial Investigation

Based on the bug report, I knew three pieces of information: 1) the badge, 2) who it was awarded to and 3) when it happened. I turned to the database to see if I could get any more info. 

My first step was to see if there were other cases where the Suffrage badge was awarded twice. 

{{< highlight sql>}}
SELECT count(*)
FROM Users2Badges
WHERE BadgeId = 804 -- Suffrage
HAVING COUNT(*) > 1
{{< / highlight >}}

Nope. Only one user was lucky enough to get it twice. Then I decided to see if anything was unusual about the day it was awarded. When I say unusual, I mean, did we have any outages, or, was there anything that could have glitched when the badges were granted?

The badge in question was awarded on June 22nd 2018, so you might be wondering how in the world would I be able to track down outages, etc. from 1.5 years ago? 

I'm glad you asked. When we throw certain exceptions on the site, they get sent to our internal chatrooms. All I had to do was go back to the transcript for that specific day, and do a quick scan to see if anything jumped out at me. Lo and behold something did. A whole lot of badge grant exceptions:

![Chat Exceptions](/image/2020/chat_exceptions.png)

### A Bit More Background

Around that time, June 2018, we were feeling the crunch of low free space on our primary SQL Server SSDs. We had &lt; 10% free, which wasn't great, so we investigated what, if anything, we could do without having to buy new drives. During that research, we came across our `Log` table, which exists in all of our databases. These tables are used to capture messages from the application. Seems harmless, right? Normally, yes, but we had been logging data into these tables for years and never deleted a single row for most log types. The `Log` tables were storing a ridiculous number of rows across **ALL OF OUR DATABASES**. We were going to gain a ton of space back by purging this old data. Here is just one database as an example:

> **StackExchange.Tor:** 
> 
> - Database Size: 10.3 GB
> - Log Table Size: 10.12 GB

Since we needed to purge data from these tables across the entire network, we used our scheduler to do it. The scheduler would allow us to do this in batches, in order to not slam the SQL Servers. The goal was to chew through the deletions, but not cause blocking or timeouts on any other process. It was a game of cat and mouse with the servers as we attempted to figure out the correct batch size. We basically were going to execute this everywhere:

{{< highlight sql>}}
DELETE Top (50000)
FROM Log
WHERE LogEntryType = 30
  AND CreationDate < GETUTCDATE() - 60
{{< / highlight >}}

We tried a 50k batch size which resulted in timeouts, similar to the screenshot above. We tried 25k which worked. We upped it again to 40k, and it failed. Finally, we dropped it back down to 25k and just let it delete as needed. Any time the scheduled deletions threw exceptions, we had a record of it in our chatroom. These messages in chat were critical to see what was happening when the duplicate badge was granted. 

After looking through the chat transcripts, I had a far better understanding of what was taking place when the badge was awarded incorrectly. I also was pretty sure I knew the cause of the duplicate awards, I just needed to prove it. 

## Digging Deeper

Based on what I saw in the transcripts, my gut was telling me the issue had to do with the massive influx to the transaction logs making the `log_send_queue_size` sky-rocket. This resulted in a much larger amount of data that needed to be written to the secondary, in other words the `redo_queue_size` was also extremely large (both values are available by querying [`sys.dm_hadr_database_replica_states`](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-hadr-database-replica-states-transact-sql?view=sql-server-ver15)). As a result, we were reading dirty data when we awarded the badges. We award badges in this manner:

1. Query the readable secondary to verify who should get the badge &mdash; we use the secondary to help spread out our read workload
2. Award the badge on the primary

We grant badges every 5 minutes, so if the secondary server was overloaded in trying to write a lot of transactions due to another process, we might be reading stale data on the next execution. I had one example of this from the bug report, I just needed a few more to prove that was the case. Time to go back to the database.

I wrote up an ugly little query to get a list of all duplicate badges awarded. 

{{< highlight sql>}}
;WITH IncorrectAwards as 
(
    SELECT ub.UserId, ub.BadgeId, Total = count(*)
    FROM Users2Badges ub
    WHERE EXISTS (SELECT 1
                    FROM Badges b 
                    WHERE b.Single = 1
                        AND ub.BadgeId = b.Id)
    GROUP BY ub.UserId, ub.BadgeId
    HAVING count(*) > 1
)
SELECT *
FROM Users2Badges ub
WHERE EXISTS (SELECT 1
                FROM IncorrectAwards ia
                WHERE ub.UserId = ia.UserId
                    and ub.BadgeId = ia.BadgeId)
{{< / highlight >}}

Using the results and the exceptions in our chatroom, I was able to confirm that we had other instances where a huge process hit the servers, overloaded the transaction logs, and resulted in duplicate badges. And what do you know I found a bunch of them! 

I didn't need to look very far back in time either. Back in November 2019, we performed a [massive reputation recalculation](https://stackoverflow.blog/2019/11/13/were-rewarding-the-question-askers/) which involved awarding reputation to question askers across the entire network. Let's just say we overwhelmed our servers just a little bit during this time and the redo of the transaction logs in our Availability Groups and Distributed Availability Groups was incredibly slow. Since we were hammering the secondaries with a lot of transactions, we unfortunately were reading staler and staler data when we awarded badges which lead to other duplicates. 

## Fixing the Issue...Or Not

Now that I discovered the issue, it should be an easy fix, right? Well, sort of. Yeah, I could easily delete the duplicate badges and move on &mdash; which of course fixes the initial bug report, but doesn't solve the underlying issue. 

What happens the next time we run something massive on the primary and we overload the transaction logs, creating a delay in the data on the secondary? The same thing would most likely happen. The problem is, we need to use the secondary to check to see who should get the badge, we can't move that back to the primary. Honestly, we're not exactly sure how we're going to fix it yet. We have some ideas like maybe checking the size of the redo queue before we execute the badge grants and if it's `some undetermined size threshold` we skip the next badge grant, or some other fix we haven't thought of yet.

## Final Thoughts

You're probably wondering why I wrote this post since we didn't even fix the bug? Well, mainly because we use our readable secondary a lot and I'm sure we aren't the only ones. While I knew we always needed to be mindful of the redo of the transaction logs to our secondaries in the availability groups, I need to be more vigilant in watching the impact to hopefully prevent a lot of clean-up later on. 

By the way, since this initial bug was caused by purging the `Log` tables due to low free drive space, I figured I'd let you know that purging all that old unneeded data brought us from 90%+ used to <50% used. It triggered some duplicate badges, but it saved us lots of money in buying new SSDs and has given us a few more years of breathing room. 

Also if you've hit any issues like this and have suggestions on how to fix it, let me know. I'm super curious. 

