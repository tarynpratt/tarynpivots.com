---
title: 'Did somebody say pivot?'
date: Fri, 09 Aug 2013 00:56:29 +0000
draft: false
tags: [crosstab, data, pivot, SQL, thoughts, transform]
excerpt: "PIVOT my favorite SQL function"
---

_Paging bluefeet, there is a `PIVOT` question to be answered._

While that might seem like a joke, it has really happened, especially over on Stack Overflow. If you have seen any of my posts, then the chances are that I was answering a `PIVOT` [question](http://stackoverflow.com/search?tab=votes&q=user%3a426671%20[pivot]%20or%20[pivot-table]%20or%20[crosstab]%20or%20[transpose]%20or%20[pivot-without-aggregate]%20or%20[unpivot]%20is%3aanswer) (or something similar). At this time of this post almost 20% of my total answers (over 3k) have been on pivot questions.

You might ask yourself, _why pivot?_ The simple answer is because I love them. I have heard the arguments,[1](http://michaeljswart.com/2011/06/forget-about-pivot/) "_don't do this type of data transformation on a server do it in the application layer_", etc. but my feeling is if there is a way to do it then go for it.

While not every database has a `PIVOT` function, I will answer or attempt to provide a solution on just about any RDBMS.

What's the point of this post? My goal is to write a series of posts outlining different methods to `PIVOT` data in a variety of databases. I probably won't be able to add much more to what is already out there on this topic, but this will be my spin on pivoting because I love it so!!

First up will be MySQL...Stay tuned.