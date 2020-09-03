---
layout: post
title: "Introducing the Geo API in Redis"
date: 2015-07-07 08:24:00
tags:
	- redis
	- redis geo
	- geohash
	- coordinates
comments: true
excerpt: |-
  GEO commands are a useful but not well known feature of Redis. In this post
  we'll show a real use case example by implementing a coordinates-based
  researchtool. We'll then deep-dive into the Geohash encoding technique that
  Redis uses underneath.
---

Since a few days, the [Geo API][geoapi]{:target="_blank"} has been introduced in Redis.
At the time of writing, the work is quite complete but still considered in
progress: everything you'll read here is actually available in the unstable
development branch and not yet released for production (plans are to release it
with the next [3.2][redis32]{:target="_blank"} version).

The Geo API consists of a set of new commands that add support for storing and
querying pairs of longitude/latitude coordinates into Redis keys. **GeoSet** is
the name of the data structure holding a set of `(x,y)` coordinates. Actually,
there isn’t any new data structure under the hood: a GeoSet is simply
a Redis SortedSet.

Let's have a quick look at the syntax of the new `GEO*` commands:

* `GEOADD key long lat name [long2 lat2 name2 ... longN latN nameN]`

*Add a member with the specified long/lat coordinates to a GeoSet.*

* `GEORADIUS key long lat radius unit [WITHDIST] [WITHHASH] [WITHCOORD] [ASC|DESC] [COUNT count]`

*Search a GeoSet for members within ‘radius’ distance from given coordinates.
Valid units for radius are m, km, ft, mi.*

* `GEORADIUSBYMEMBER key member radius unit [WITHDIST] [WITHHASH] [WITHCOORD] [ASC|DESC] [COUNT count]`

*The same as GEORADIUS, but the search is centered on the coordinates of a
member of the GeoSet.*

* `GEOPOS key elem1 elem2 ... elemN`

*Return the long/lat coordinates of each specified element in a GeoSet.*

* `GEODIST key elem1 elem2 [unit]`

*Return the distance in unit (meters by default) between elem1 and elem2 in a
GeoSet.*

* `GEOHASH key elem1 elem2 ... elemN`

*Return the GeoHash string representation (an array of 11 characters) of the
coordinates of the specified elements.*

* `GEOENCODE long lat [radius unit]`

*Translate the given coordinates to the GeoHash integer value with highest
accuracy.*

* `GEODECODE hash`

*Translate a GeoHash integer value back to a pair of coordinates.*

I think at this point you're wondering what a GeoHash is ... before digging into
details let's have a visual example.

## Building an Hotel Reservation system with map search

A picture is worth a thousand words, so let's build a real-world example.

I downloaded (thanks to [Overpass turbo](http://overpass-turbo.eu){:target="_blank"} a small dataset with the coordinates of a set of hotels in Rome (a very small set indeed) and imported it into Redis.
This is the [GeoJSON
dataset](https://gist.github.com/cristiangreco/e806521f70370eaa1c1b){:target="_blank"}
and an example script to load it. It is as easy as adding coordinates with
symbolic names in a GeoSet and then populating a key for each hotel data (using
such names in order to later retrieve it).

	geoadd hotels 12.4844774 41.9184129 h:259299923 12.4920147 41.9175913 h:261733735 ...
	set h:259299923 '{"geometry": {"type": "Point", "coordinates": [12.4844774, 41.9184129]} ...'
	set h:261733735 '{"geometry": {"type": "Point", "coordinates": [12.4920147, 41.9175913]} ...'
	...

All the hotels visualized on a map:

<div style="text-align: center" markdown="1">
![all Hotels on map](/images/all.png)
</div>

Now let's suppose you're searching for an accommodation near [Stazione
Termini](https://en.wikipedia.org/wiki/Roma_Termini_railway_station){:target="_blank"}
(not really the best place to stay in Rome). How do you retrieve the hotels
within 1km from your point of interest?

	GEORADIUS hotels 12.499705 41.902372 1 km
	 1) "h:530719587"
	 2) "h:3412957318"
	 3) "h:3554601725"
	 4) "h:1743613641"
	 5) "h:3292538681"
	...

Fine, but a friend of mine suggested me a cheap accommodation near Termini,
let’s say property known as `h:2309332158`. Ouch, no availability for my
dates... let’s search for the accommodations at shortest distance:

	GEORADIUSBYMEMBER hotels h:2309332158 1 km withdist asc
	 1) 1) "h:2309332158"
	    2) "0.0000"
	 2) 1) "h:3403814918"
	    2) "0.0627"
	 3) 1) "h:3403814919"
	    2) "0.0657"
	 4) 1) "h:3403817806"
	    2) "0.0688"
	 5) 1) "h:2397317566"
	    2) "0.0928"
	...

This is how the result could be presented to the user:

<div style="text-align: center" markdown="1">
![1km range search](/images/100.png)
</div>

Let's retrieve the position on the map of two of the hotels in this list:

	GEOPOS hotels h:3403814919 h:3403817806
	1) 1) "12.497514188289642"
	   2) "41.899867204456598"
	2) 1) "12.497755587100983"
	   2) "41.900039565495433”

Quite close to each other? Yeah, say less than 30 meters:

	GEODIST hotels h:3403814919 h:3403817806
	"27.693296515757325"

### Under the hood: GeoHashing

We said before that a GeoSet in Redis is implemented as a SortedSet (actually
the `GEOADD` command translates your input to a proper `ZADD` command). All
members in a SortedSet have a score value associated used for sorting (indeed)
and range indexing. But how is it possible to translate a pair of coordinates to
a score?

[GeoHashing](https://en.wikipedia.org/wiki/Geohash){:target="_blank"} is a
technique to encode a pair of longitude/latitude coordinates into a single ASCII
string (usually encoded `base32`). It all started with the
[geohash.org](http://geohash.org){:target="_blank"} service as a way to
represent locations with short and easily shareable urls.

The GeoHash encoding algorithm offers arbitrary precision. The accuracy of the
representation depends on the number of bits used. Also, the raw binary form can
be interpreted as an integer instead of a base32 string. This is the intuition
behind the usage of GeoHash as a score for a SortedSet.

The integer GeoHash implementation in Redis derives from
[Ardb](https://github.com/yinqiwen/ardb){:target="_blank"}, but has been
completely reimplemented by [Matt
Stancliff](https://matt.sh/redis-geo#_origin-story){:target="_blank"} and is
based on a 52bit integer representation (which gives an accuracy of less than 1
meter). Why 52bit? Because SortedSet uses double values for members scores, and
a double can safely hold a 52bit integer without loss of accuracy.

The GeoHash score of a GeoSet member can be exposed with a `ZRANGE` command
(GeoSets are SortedSets, right?):

	ZRANGE hotels 0 -1 withscores
	  1) "h:2138691946"
	  2) "3480341633710855"
	  3) "h:2302346446"
	  4) "3480341645075993"
	  5) "h:2985974223"
	  6) "3480341646134189"
	...

The corresponding base32 string representation can be obtained in this way:

	GEOHASH hotels h:530719587
	1) "sr2ykhnegy0"

See how it translates back to coordinates at
[http://geohash.org/sr2ykhnegy0](http://geohash.org/sr2ykhnegy0){:target="_blank"}.

So, the integer GeoHash encoding is a Redis-specific implementation detail, and
Geo API provides two specific commands to deal with it:

	GEOENCODE 12.498113 41.8994816
	1) (integer) 3480343095387795
	2) 1) "12.498112320899963"
	   2) "41.899480659479806"
	3) 1) "12.498117685317993"
	   2) "41.899483194200954"
	4) 1) "12.498115003108978"
	   2) "41.89948192684038"
	5) 1) "3480343095387795"
	   2) "3480343095387796"

	GEODECODE 3480343095387795
	1) 1) "12.498112320899963"
	   2) "41.899480659479806"
	2) 1) "12.498117685317993"
	   2) "41.899483194200954"
	3) 1) "12.498115003108978"
	   2) "41.89948192684038"

The output may look scary at first, but keep in mind that a GeoHash represents
actually an area rather than a single point (in fact, you can add a radius to
GEOENCODE). So the first pair of coordinates in this multi-bulk reply is the
minimum corner (i.e. the south-west point) of the bounding box, then the maximum
corner (north-est point) and an averaged center of the area.

The GEOENCODE command also outputs two scores to be used as range query
parameters. A fundamental property of GeoHashing is that a range search between
a lower and a higher range (excluded) will contain all members with
corresponding coordinates in this range.

Given a pair of coordinates, we use GEOENCODE to determine the GeoHash that
represents its 1km bounding box with highest accuracy:

	GEOENCODE 12.498113 41.8994816 1000 m
	1) (integer) 3480343080337408
	2) 1) "12.48046875"
	   2) "41.892249100012208"
	3) 1) "12.50244140625"
	   2) "41.902631317880861"
	4) 1) "12.491455078125"
	   2) "41.897440208946534"
	5) 1) "3480343080337408"
	   2) "3480343097114624"

Then, a range search using the returned low-high GeoHashes produces:

	ZRANGEBYSCORE hotels 3480343080337408 3480343097114624
	 1) "h:3192164485"
	 2) "h:1649776328"
	 3) "h:985232400"
	...

A visual representation of the bounding box: note that the initial (highlighted)
value is not the center of the box, but just falls within it.

<div style="text-align: center" markdown="1">
![1km bounding box](/images/bb.png)
</div>

[geoapi]: https://github.com/antirez/redis/blob/unstable/src/geo.c
[redis32]: http://antirez.com/news/89
