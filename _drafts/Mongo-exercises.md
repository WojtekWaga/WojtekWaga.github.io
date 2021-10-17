---
layout: page
title: MongoDB Exercises
categories: MongoDB 
---

When I was searching for some exercises to practise querying MongoDB there was little to be found on the internet. All I found was rather simple but what is more important it was all a synthetic. It was like the usual FAQ section on a typical company's webpage covering questions like: "How can you keep proces so low?" that I cannot imagine being ever asked.

Each exercise is in a separate section and there is no particular order except for the task 0 which needs to come first as it's about data preparation. We will use apublicly available dataset from Airbnb hosted by [Inside Airbnb](http://insideairbnb.com/get-the-data.html). Please consider supporting them. I chose Vienna data set and my answers will be based on it but you can use whichever is near and dear to you. Just you may need to run my queries agains it to compare the result if your query looks different.

All exercises assume you have an access to MongoDB 5.x and preferably the mongosh shell. If you don't have a MongoDB instance at your disposal you can run one instantly using docker:

``` bash
docker run --name mongo -d mongo:5.0.3
``` 

and once finished and you have your shell back:

``` bash
docker exec -ti mongo mongosh
```

et voila, you are connected to your very own MongoDB instance.

### Exercise 0: Load Airbnb dataset

Inside airbnb offers multiple files to download for each city but for now we will focus on only three of them: listings.csv.gz containing detailed information on places to book, calendar.csv.gz with information on actual bookings and reviews.csv.gz with, guess what, users' reviews. 

Download all 3 files and load them into "airbnb" database. The collections should be named listings, calendar and reviews.

#### Answer:

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

Congratulations, you have a ready to use dataset and can proceed to any exercise you want.

### Exercise 1

Check document count for each collecion.

my output:
``` bash
[ { listings: 9625 }, { reviews: 190972 }, { calendar: 3512760 } ]
```

#### Answer
``` bash
test> use airbnb
switched to db airbnb
airbnb> db.getCollectionNames().map( x=> ({[x] : db[x].countDocuments()}) )
[ { listings: 11583 }, { reviews: 355004 }, { calendar: 4227814 } ]
```

### Exercise 2

Find the three most popular amenities.

my outout:
``` bash
[
  { _id: 'Wifi', count: 11067 },
  { _id: 'Kitchen', count: 10756 },
  { _id: 'Heating', count: 10454 }
]
```

hint: amenities field is in a pretty wierd format - string containing an array, you will get much simpler query if you prepare your data first and change this to a normal array.

answer:

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

description:
* project - remove all unnecessary fields from further processing
* unwind - changes array of elements i.e. [1,2,3] into separate documents 1, 2 ,3
* group - groups by amenity type and counts occurences
* sort - sort in descending order
* limit - pick top 3

### Exercise 3 Find the number of places with no wifi



answer:

this does not work:
```bash
db.listings.find({"amenities":{$ne: "Wifi"}}).count()
```

because some places are having the likes of 'Wifi – 15 Mbps' we need to search all descriptions for case insensitive text match for "wifi" to cover "Fast wifi" as well as "Wifi - 15Mbps".


