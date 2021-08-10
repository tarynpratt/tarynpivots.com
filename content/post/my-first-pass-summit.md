---
title: 'Finally, My first PASS Summit'
date: Mon, 02 Dec 2019 08:00:00 +0000
draft: false
tags: [pass-summit, SQL, pass, sql-server, thoughts, wit-panel, sql-community, career, work-adventures]
excerpt: "My Thoughts on attending my first PASS Summit"
---

I've been working with SQL Server for a long time, and have always wanted to attend PASS. For one reason of another, with numerous employers, I hadn’t been able to go. The answer was always no. `<insert sad face>` 

The biggest blocker was always the cost.

The conference is expensive. 

Seattle is expensive. 

It has always been impossible to get my employer to foot the bill for it. 

This year I was incredibly lucky that my employer, [Stack Overflow](https://stackoverflow.com), paid for my trip to PASS Summit. I was thrilled to finally be able to attend. Since I was given the opportunity to go, I wanted to write out my thoughts as a PASS "First-Timer", as well as some session highlights. 

## A bit about me

I'm an introvert. Walking into a place...by myself, where I know no one is incredibly difficult. It triggers lots of anxiety which makes the idea of attending a conference with thousands of people who I don't know, insanely hard. Sure, I know people at PASS Summit because I have seen their faces online from following them on Twitter, and have interacted with them virtually, but I have never met most of them in person. In order to meet them, I need to actually meet them...like talk to them...which is a struggle. I was going to try my best to work past that and take in everything I could.

## Scheduling Overload

The week before PASS, I sat down and filled out my schedule which was a bit challenging. There were so many sessions that I wanted to attend. My final schedule had a ton of sessions at the same time. I did this in the event I would change my mind last minute, so I’d have an alternate option. I knew I was overplanning, and was being  overly ambitious, but I was going to push myself to hit as many sessions as I could. 

## First Timer Buddy Program

I saw mentions of the Buddy Program and wasn't sure if I was going to sign up, but I'm glad I did. For those who don't know, the Buddy program at PASS Summit pairs you with someone who has been to the conference before, and they offer suggestions on events and sessions to attend. If you're lucky and if time allows, you'll meet them and have a familiar face in the thousands of people. I was lucky to get paired with [Matt Gordon](https://twitter.com/sqlatspeed). Matt reached out a couple of weeks before the Summit and suggested a couple of sessions, then the week of the conference we met and had lunch together as a group. It was great to meet other first-timers in a smaller setting. 

## WIT Reception

I signed up for the WIT Reception which took place on the Tuesday night before the official start of the conference. I loved this event. It was smaller than many of the other events at the conference, and it was a great way to meet people before the conference officially started, and gave me a chance to have some familiar faces to find in the crowds of people the rest of the week. Not to mention, I met some really awesome people at the event. 

The WIT happy hour overlapped with the First Timers Welcome Reception. Since I was comfortable at the WIT event, I did not leave to attend the First Timer Event. Was that the right decision? I don't know. I met some great people at the WIT event and didn't want to leave, however, I know that if I went to the First Timer Event, I probably would have met other newbies, which might have been beneficial. 


## Day One

### [Performance Tuning Azure SQL Databases](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=92582) with David Maxwell

At this time, we only use physical SQL Servers in our datacenters. As a result, I don't have a lot of hands-on experience with Azure SQL, so I figured I would take the opportunity to learn a bit while at PASS. I started with a how to performance tune Azure SQL databases. This session was really good to hear about some of the differences in on-prem and Azure SQL and how you tackle performance issues using Extended Events and Automating Tuning. 

### [Improving Availability in SQL Server and Azure SQL Database with Accelerated Database Recovery and Resumable Operations](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=98848) with Pam Lahoud

This session showed off one of the new features of SQL Server 2019 that I'm excited to test in our environments, Accelerated Database Recovery. It allows the databases to recover faster from long-running transactions that were aborted and are rolling back. 

Prior to SQL Server 2019, you might have to wait hours for a rollback of transaction on a long-running query before the server would recover, all the while you could have queries blocked, etc. With the new release, if there is a long running transaction that gets killed, then the database will be unblocked immediately. This is a cool new feature of SQL Server 2019 that has some overhead, but I'm excited to test it out. 

The session also covered Resumable Index Operations which allows an index rebuild operation to be paused and then resumed or aborted at another time. Again, a cool feature I'm excited to play with.


## Day Two

### Keynote with Tarah Wheeler - Cybersecurity is Everyone's Problem

There is no doubt this was one of my favorite sessions of the conference. [Tarah Wheeler](https://twitter.com/tarah) is an expert in cybersecurity and an author. She talked about the conflict between cybersecurity and data science - the struggle between keeping or purging data in the age of three internets - the European Union (EU and GDPR), China, and then the rest of the world. Since data is constantly growing, how do you satisfy the need to delete data under different regulatory requirements. The problem is huge and is going to keep compounding as we continue to store data. The question she posed in this slide captures how complicated the situation is.

![Asking the tough questions](/image/2019/keynote_tarahwheeler.jpg)

The Keynote was incredible on every level and hands down, it was the best thing I attended all week.

### [Columnstore Indexes in 2017-2019](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=92589) with Niko Neugebauer

I have read many of Niko's blog posts on Clustered Columnstore, so getting the opportunity to see him talk about them in person was a no brainer. 

Within just a few minutes, I thought, "hey that might be our problem." The biggest lightbulb moment in this session was how dictionary size and memory pressure can break row groups in ways that are potentially unexpected. As soon as I heard this, I knew what I was going to investigate it when I was back in the office. This session was great because I had takeaways that I could possibly use or research immediately.

### [Azure SQL Database - Lessons Learned from the Trenches](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=90954) with Jose Manuel Jurado Diaz and Roberto Cavalcanti

I also loved this session. The speakers went through different scenarios that customers have hit while using Azure SQL Databases. 

Seeing different tips and tricks to look for when performance tuning in that environment gave me a lot to think about for future work in the cloud. Things as simple as an implicit conversion can perform significantly different on Azure SQL Database once you hit an increase in data that an old cached plan can no longer handle. 

### [Improving Columnstore Load Scalability on Large Servers](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=91912) with Joe Obbish

I had heard there were 180 slides for this session and was sure my head would be spinning from the amount of information. Instead of being overwhelmed by it, Joe did a great job of going over how to optimize loading millions of rows of data into a clustered columnstore table in just 6 minutes.

He went through each step in his optimization process, explaining what he looked for to tune in the next step. With each attempt the processing time dropped considerably. 

Besides watching how to performance tune huge data loads into clustered columnstore tables, I learned several bits that I could take back and see about implementing in our environment. This was another top session for the week. 

## Day Three

### [Batch Execution Mode on Rowstore Indexes](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=92529) with Niko Neugebauer</h4>

This session included some great demos on the new feature in SQL Server 2019, Batch Execution Mode on traditional rowstore indexes. I'm not sure how much benefit we will get with this in our environment, but I'm curious to get SQL Server 2019 installed on more servers to see if we do. 

### [Deep Dive into Blocking and Deadlocks Troubleshooting](https://www.pass.org/summit/2019/Learn/SessionDetails.aspx?sid=92528) with Dmitri Korotkevitch

Blocking and Deadlocks every DBAs nemesis. At our scale, I fight with them a lot, so having the chance to attend a session about troubleshooting was a must. Even though I have experience with dealing with both blocking and deadlocks, it was a great overview on why they occur, and how best to troubleshoot them. This was a solid session to attend. 

## Final Thoughts

The week flew by. It was information overload in each session that I attended, but I'm so happy I finally had the chance to go. Even though I was out of my comfort zone, I met a ton of people and learned a lot while I was there. 

I hope I'll be able to attend again in the future...maybe even next year in Houston. 