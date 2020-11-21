---
layout: page
title: 'Weird, weirder, MongoDB'
date: 2020-10-04 12:06:47
categories: MongoDB
---
Moving from a SQL-backed database to MongoDB can be quite an experience. Your tables are gone along with the schema that used to be in the top 3 responsibilities of your architects and the last but not least you can think of relations as "it's complicated'. Soon after migration you start to realize that the tables you loved were no big deal and collections though slightly different will do the trick. Lack of the schema is more handy than not as you can enforce schema on the API level and easily upgrade your code without the need of ever calling the likes of ALTER TABLE. Relations are a bit different as data is relational by nature and you want to care for it in your DB. This last sentence makes it clear why a NoSQL database is a better term than non-relational database. 
Things done! You have a running MongoDB instance that keeps your data and answers your questions all backed by carefully created indexes. You feel like an expert looking down on all the people still using obsolete SQL. Congratulations! You've just reached the peak of "Mount Stupid".

![Alt text](https://upload.wikimedia.org/wikipedia/commons/thumb/4/46/Dunning%E2%80%93Kruger_Effect_01.svg/577px-Dunning%E2%80%93Kruger_Effect_01.svg.png)

Have you ever compared the response time or the disk space allocated say against SQLite*? You may be surprised.

*) Author of this blog has no affiliation with SQLite and just as everyone cannot get his head around why there are no RIGHT JONIS in SQLite and you need to turn your picture by 180 degrees and utilize LEFT JOINS instead. 

Now the time to descent to the "Valley of Despair". First thing you realize is that typing SQL is somehow easier to typing MongoDB aggregations. It's likely that if there were no brackets highlighting in the mongo shell MongoDB would have been doomed long ago as your typical aggregation ends with something like: "}]}}])". Next thing is that JSONs you see all over your screen are not always JSONs. They may be JSONs, "MongoDB Extended JSONs" in two revisions (v1 and v2) and two flavors (cannonical and regular) or none of them. 


``` JavaScript
> db.something.createIndex({"a":1,"b":1})
> db.something.find({"a":1,"b":1})
```

Is the document '{"a":1,"b":1}' in the above expressions a JSON or not. The answer is it depends. Line 2 does not look bad but for indexes order is crucial and JSON format does not seem to care much:

"An object is an unordered collection of zero or more name/value pairs, where a name is a string and a value is a string, number, boolean, null, object, or array." (Source RFC 7159)

same story for the extended version:

Parsers MUST NOT consider key order as having significance. For example, the document {"$code": "function(){}", "$scope": {}} must be considered identical to {"$scope": {}, "$code": "function(){}"}.

there is an exception though: 

Keys within Canonical Extended JSON type wrapper objects SHOULD be emitted in the order described. (Source [Extended JSON Specification](https://github.com/mongodb/specifications/blob/master/source/extended-json.rst))

but applies only to "type wrapper" which is not the case here.

If you dump your indexes to a file and try to load them I wish you good luck. This is a quite lucky process after all and most likely you will succeed. It's only months down the line on some prod environment in the middle of the night that this will fail.

Let's search for a particular object now...

``` JavaScript
> db.sth.find({"_id" : ObjectId("5f7060514cf75c9907f9aeb9")})
{ "_id" : ObjectId("5f7060514cf75c9907f9aeb9"), "a" : 1, "b" : 1 }
```
and mongoexport it!

``` JavaScript
$ mongoexport --collection sth -q'{"_id" : ObjectId("5f7060514cf75c9907f9aeb9")}'
2020-09-27T09:55:19.679+0000    connected to: mongodb://localhost/
2020-09-27T09:55:19.679+0000    Failed: error parsing query as Extended JSON: invalid JSON input. Position: 9. Character: O
```
None of our JSONs formats again just announced in a more verbose way.

MongoDB let's you sort in memory if your data has no order guaranteed by an index (just like any other DB). The only difference here is that MongoDB wants to punish you much harder for this and your sort is limited to 32MB of memory for find().sort() and if you use equivalent sort in aggregate you get 100MB instead (why the difference? why those values?). You can put a lengthy parameter to allow disk use during sorting that suppressess the limit but it's too easy to leave an in-memory soft in the code when the dataset is small and instead of degrading performance over time get a full-blown crash. 

Another surprise is the group command or rather a pipeline stage. MongoDB distinguishes two types of operations: blocking and non-blocking. The difference is that non-blocking operations start returning data immediately whereas blocking ones need to first get all the data from the db before returning even the first item. The concept is quite simple. Let's assume we have no index. "Give me 100 documents" is a non-blocking one as the DB can start streaming results straight away. "Give me the smallest number" on the other hand will need to wait for the DB engine to look at every single number before returning the smallest one. If you have a proper index then sorting (or "give me the smalest number") becomes non-blocking. But for some reason group does not take advantage and instead of returning groups on the go it waits to see all data blocks and what's even more funny loses ordering (more info: https://jira.mongodb.org/browse/SERVER-4507).

Every SQL programmer knows that grouping without aggregations and distinct are equivalent operations. Same here just distinct fell into magic potion and works way faster than group. In the next post we'll start to climb the slope of enlightenment with the help of an 'ESR' rule. I will also quote some timings from different group commands.
