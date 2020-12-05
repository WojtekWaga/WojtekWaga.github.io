---
layout: page
title: To backup or not to backup.
categories: Sysadmin
---
As a saying goes, "There are two groups of people the ones who do backups and the ones who will be doing them". As the first group has little to worry about this article is focused on those who are about to find their perfect backup solution. Some Linux knowledge is required, though. A pefect reader is someone who owns no more than a handful of systems and praises security above all.

*3-2-1 backup strategy* that is recognized as a best practise tells you to keep 3 copies of data on 2 different media types with 1 off-site backup. Different media types will give you security not only against things like moisture or strong electomagnetic radiation but also different media types tend to age at different rates.

Backups fall into one of the three categories: *Full*, *Differential* and *Incremental*. A full backup is no less than a copy of your data. For practical reasons this is not something you would do on a daily basis. For a 100GB worth of data you will get 1TB for 10 snapshots. And the more snapshots you keep the better. If you keep only a single version and you destroy your data you have only a day or so to figure it out. Otherwise your change will propagate to this only copy and destroy it altogether. Differential backups can mitigate this problem by storing only changes from the last full backup. And to restore yout backup you need to get to the last full backup and apply the differential one.

<<picture differential backup>>

Can we do better? It looks we can. We can keep differences from differential backups the same way we do for the full ones. And this is exactly the third category we were to discuss. Incremental backups are the most space efficient of the three. This, however comes at the price of comlpexity and durability. If you do a full backup on Jauary the first and incremental ones throughout the year you will need to apply 364 deltas if thing go awry on December 30th. If the risk of data corruption of one chunk is P then what you aim for is 364*P.

**There** are a variety of media one can use for backup. They differ in size and 


|Medium type | Capacity in GB | Life span in years | Price per TB |
| ----------- | ----------- |
|CD | 0.7 | 3-5 | $270 |
|DVD | 4.7 | 3-5 | $48 |
|M-Disc | 4.7 | 1000 | $750 |
|M-Disc BD-R | 25-100 | 1000 | $184 |
|BD-R LTH | 25-100 | |
|BD-R HTL | 25-100 | |
|Memory cards | 0-1000 | |
|SSD | 0-8000 | |
|HDD | 0-12000  | ~5 |
|LTO | 2500-12000  | 30 | $11 |
