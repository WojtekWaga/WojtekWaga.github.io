---
layout: blog
title: MongoDB Exercises
categories: MongoDB 
---

When I was searching for some exercises to practise querying MongoDB there was little to be found on the internet. All I found was rather simple but what is more important it was all synthetic examples. It was like the usual FAQ section on a typical company's webpage covering questions like: "How can you keep prices so low?" that I cannot imagine being ever asked.

Each exercise here is in a separate section and there is no particular order except for the exercise 0 which needs to come first as it's about data preparation. We will use a publicly available dataset from Airbnb hosted by [Inside Airbnb](http://insideairbnb.com/get-the-data.html) (please consider supporting them). I have chosen Vienna data set and my answers will be based on it but you can use whichever is near and dear to you. Just you may need to run my queries against it to get the expected output should your query looks different.

All exercises assume you have access to MongoDB 5.x and imost preferably the mongosh shell. If you don't have a MongoDB instance at your disposal you can run one instantly using docker:

``` bash
docker run --name mongo -d mongo:5.0.3
``` 

once finished and you have your shell back:

``` bash
docker exec -ti mongo mongosh
```

et voila, you are connected to your very own MongoDB instance.

## Exercises

### Exercise 0: Load Airbnb dataset

Inside airbnb offers multiple files to download for each city but for now we will focus on only three of them: listings.csv.gz containing detailed information on places to book, calendar.csv.gz with information on actual bookings and reviews.csv.gz with, guess what, users' reviews. 

Download all the 3 files and load them into "airbnb" database. The collections should be named after files: listings, calendar and reviews.


### Exercise 1: Check document count for each collecion.

**Expected output:**
``` bash
[ { listings: 11583 }, { reviews: 355004 }, { calendar: 4227814 } ]
```

### Exercise 2: Find the three most popular amenities.

**Expected output:**
``` bash
[
  { _id: 'Wifi', count: 11067 },
  { _id: 'Kitchen', count: 10756 },
  { _id: 'Heating', count: 10454 }
]
```

hint: amenities field is in a pretty wierd format - string containing an array, you will get much simpler query if you prepare your data first and change this to a normal array.

### Exercise 3: Find the number of places with no wifi

Please keep in mind that people sometimes put things like "wifi - 15Mbps" instead of just wifi of Wifi.

**Expected output:**
386

More exercises to come.

## Answers

### Exercise 0

Non docker version:
``` bash
gunzip ./*.gz
mongoimport --db airbnb --collection listings --type csv --headerline ./listings.csv
mongoimport --db airbnb --collection reviews --type csv --headerline ./reviews.csv
mongoimport --db airbnb --collection calendar --type csv --headerline ./calendar.csv
```

docker version:
``` bash
gunzip ./*.gz
docker exec -i mongo mongoimport --db airbnb --collection listings --type csv --headerline < ./listings.csv
docker exec -i mongo mongoimport --db airbnb --collection reviews --type csv --headerline < ./reviews.csv
docker exec -i mongo mongoimport --db airbnb --collection calendar --type csv --headerline < ./calendar.csv
```

without prior decompression:
``` bash
zcat ./reviews.csv.gz | mongoimport --db airbnb --collection reviews --type csv --headerline
zcat ./listings.csv.gz | mongoimport --db airbnb --collection listings --type csv --headerline
zcat ./calendar.csv.gz | mongoimport --db airbnb --collection calendar --type csv --headerline
```

and with iteration over all compressed files in a directory... just kidding, but you shoud try on your own :]

### Exercise 1

``` bash
test> use airbnb
switched to db airbnb
airbnb> db.getCollectionNames().map( x=> ({[x] : db[x].countDocuments()}) )
```

### Exercise 2
deserialize array:
``` bash
db.listings.updateMany({}, [ {
           $set: {
               "amenities": {
                   $function: {
		     body: "function(x){return JSON.parse(x)}",
		     args: ["$amenities"],
		     lang:"js"
		     }
		   }
	         }
	       }
	     ]
)
```

find the answer:
``` bash
db.listings.aggregate([
  {$project: { "_id": 0, "amenities": 1 }},
  {$unwind: "$amenities"},
  {$group: {"_id":"$amenities", "count":{$sum: 1}}},
  {$sort: {"count":-1}}, {$limit: 3}
])
``` 

In case you are wondering why we need this: 'body: "function(x){return JSON.parse(x)}' instead of just: 'body: JSON.parse' or 'x=>JSON.parse(x)'. Give them a try. I have just raised this as a [SERVER-60770](https://jira.mongodb.org/browse/SERVER-60770). I mentioned only the first case but adding proper JS semantics to this mechanism should fix both altogether.

description:
* project - remove all unnecessary fields from further processing
* unwind - changes array of elements i.e. [1,2,3] into separate documents 1, 2 ,3
* group - groups by amenity type and counts occurences
* sort - sort in descending order
* limit - pick top 3



### Exercise 3
```bash
db.listings.find({"amenities":{$not: /wifi/i}}).count()
```


