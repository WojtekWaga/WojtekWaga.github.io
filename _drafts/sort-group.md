---
layout: page
title: The $sort, the $group the ugly.
categories: MongoDB 
---

MongoDB let's you sort in memory if your data has no order guaranteed by an index (just like any other DB). The only difference here is that MongoDB wants to punish you much harder for this and your sort is limited to 32MB of memory for find().sort() and if you use equivalent sort in aggregate you get 100MB instead (why the difference? why those values?). You can put a lengthy parameter to allow disk use during sorting that suppressess the limit but it's too easy to leave an in-memory soft in the code when the dataset is small and instead of degrading performance over time get a full-blown crash. 

Another surprise is the group command or rather a pipeline stage. MongoDB distinguishes two types of operations: blocking and non-blocking. The difference is that non-blocking operations start returning data immediately whereas blocking ones need to first get all the data from the db before returning even the first item. The concept is quite simple. Let's assume we have no index. "Give me 100 documents" is a non-blocking one as the DB can start streaming results straight away. "Give me the smallest number" on the other hand will need to wait for the DB engine to look at every single number before returning the smallest one. If you have a proper index then sorting (or "give me the smalest number") becomes non-blocking. But for some reason group does not take advantage and instead of returning groups on the go it waits to see all data blocks and what's even more funny loses ordering (more info: https://jira.mongodb.org/browse/SERVER-4507).

Every SQL programmer knows that grouping without aggregations and distinct are equivalent operations. Same here just distinct fell into magic potion and works way faster than group. In the next post we'll start to climb the slope of enlightenment with the help of an 'ESR' rule. I will also quote some timings from different group commands.
