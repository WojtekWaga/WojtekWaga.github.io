---
layout: page
title: To backup or not to backup.
categories: Sysadmin
---
As a saying goes, "There are two groups of people the ones who do backups and the ones who will be doing them". As the first group has little to worry about this article is focused on those who are about to find their perfect backup solution. Some Linux knowledge is required, though. A pefect reader is someone who owns no more than a handful of systems and praises security above all.

*3-2-1 backup strategy* that is recognized as a best practise tells you to keep 3 copies of data on 2 different media types with 1 off-site backup. Different media types will give you security not only against things like moisture or strong electomagnetic radiation but also different media types tend to age at different rates.

Backups fall into one of the three categories: *Full*, *Differential* and *Incremental*. A full backup is no less than a copy of your data. For practical reasons this is not something you would do on a daily basis. For a 100GB worth of data you will get 1TB for 10 snapshots and so on. And the more snapshots you keep the better. If you keep only a single version and you destroy your data you have only a day or so to figure it out. Otherwise your change will propagate to this only copy and destroy it altogether. Differential backups can mitigate this problem by storing only changes from the last full backup. And to restore your backup you need to get to the last full backup and apply the differential one.

<<picture differential backup>>

Can we do better? It looks we can. We can keep differences from differential backups the same way we do for the full ones. And this is exactly the third category we were to discuss. Incremental backups are the most space efficient of the three. This, however comes at the price of comlpexity and durability. If you do a full backup on Jauary the first and incremental ones throughout the year you will need to apply 364 deltas if thing go awry on December 30th. If the risk of data corruption of one chunk is P then what you aim for is 364*P.

There are a variety of media one can use for backup. They differ in size, durability, cost and ease of use. Here are some examples:

| Medium type | Capacity in GB | Life span in years | Price per TB |
| ----------- | ----------- | ----------- | ----------- |
|CD | 0.7 | 5-10 | $270 |
|DVD | 4.7 | 5-10 | $48 |
|M-Disc | 4.7 | 1000 | $750 |
|M-Disc BD-R | 25-100 | ? | $184 |
|BD-R LTH | 25-100 | 5-10 | $22 |
|BD-R HTL | 25-100 | 50-300 | $36 |
|Memory cards | 0-1000 | ~5 | $270 |
|SSD | 0-8000 | 5-10 | $136 |
|HDD | 0-12000  | ~5 | $30 |
|LTO | 2500-12000  | 30 | $11 |

The question is: what does it mean for M-Discs to have a life-span of a thousand years? Does it always fail the year after? Do all the disks live that long? The short answer is *no*. 1000 years is the mean life span so you may expect half the disks to fail before with some even way before. Milleniata, the company behind the brand also claims that 95% of disks will make it to around 500 years which gives more insight.

By the looks of it there are four items worth attention here: LTO, M-Disc BD-R and BD-R HTL and of course good old HDD. LTO is not something you would use at home unless you have a [tape library](https://www.youtube.com/watch?v=CVN93H6EuAU) down in your basement. M-Disc BD-R and a normal BD-R are more common than not. They share the form factor and storage size of 25GB (BD-XL 100GB). M-Discs can even be read in normal BD players but it takes an M-Disc certified device to burn one. What's really different between the two is the price and the longevity claims. One can find some studies suggesting that both M-Disks and high quality BD-R are more or less the same in terms of longevity.

Now, once we have chosen the medium type for our backup (my choice is HDD because of the ease of use for automated backups plus most important thigs to me also go on BD-R HTL) we have to find out where to store our off-site copy. There are a lot of options here as well. But the choice is somewhat simpler. We will consider only cloud storage and providers supported by the tool we will use. 

Our requirements for a perfect backup solution:
* full automation
* runs periodically
* encrypts our data
* splits data into volumes of a given size
* supports incremental backups
* free of charge
* runs on Linux (ARM distros included)

[Duplicity](http://duplicity.nongnu.org/) is a tool that meets all of these points. It's a really simple tool that backs up one folder to another. You can run it as simply as:

``` Bash
$ duplicity one_directory file:///home/user/another_directory
```

