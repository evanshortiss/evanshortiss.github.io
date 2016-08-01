---
published: false
title: A Small Story about Scaling
layout: post
categories: development javascript nodejs
---

## simple right?
On a recent client engagement we had the task of building a RESTful API using
node.js. This API would run as a service that another node.js application would
call on behalf of mobile devices to synchronise data to client devices. It
would also accept payloads from devices and send them through an ActiveMQ when
required. Simple right?

## trouble on the horizon
As we hit around 700 users in load testing we began to see dramatic performance
issues with response times spiking from sub 1 second to over 45 seconds for one
call in particular - concerning news considering we needed to support 4 times
that load! Most calls became slower in a linear fashion which was at least a
little reassuring since we weren't looking at a seemingly randomised issue.

Doing some napkin math we can determine our required throughput. Our API has 3
datasets exposed that users synchronise every 2 minutes via HTTP requests.
Since we must cater for the worst case scenario, we need to be able to sustain
approximately 10,500 concurrent requests if we include some of the less
commonly hit routes that it exposes.

## the game plan
After a quick team meeting we prioritised investigation into the following
areas:

* inefficient queries
* database indexing
* code analysis

### inefficient queries
The data access layer in this application had been written in the early stages
of this project, and since we were working in a high-pressure environment it
hadn't seen much love since those days. Upon analysis one of our consultants
immediately found a query that had slipped through the cracks. The
code below loosely demonstrates the issue.

{% highlight js %}

var mongo = require('./mongo');

exports.disassociateUserFromData = function (userId, callback) {
  var query = {
    // TODO
  };

  var updates = {
    $pull: {
      users: userId
    }
  };

  mongo.getCollection('data').update(query, updates, callback);
}

{% endhighlight %}

For those who aren't familiar with MongoDB this can easily be explained using
some SQL that would perform a similar operation.

{% highlight sql %}
UPDATE WHERE 1=1 SET colname=TODO
{% endhighlight %}

Nope. Not good. If you ever thought that implementing a code review process
might be a good idea then this is all the evidence needed. In high pressure
environments people can easily miss these things when rushing to get
features completed. Code reviews ensure that at least two people view all code
and increases the likelihood things like this are spotted before they slip
through the cracks.

Fixing this query immediately resolved the huge spike we saw in all requests
when testing with data similar to production volume, but we weren't out of the
woods just yet since our response times continued to climb rapidly after we
hit ~1200 users.

### database indexing
Next we turned to database indexing. Since we were using MongoDB it was easy to
gauge the efficiency of queries using the _query.explain_ functionality it
offers.

Here's an example of a simple query on a "data" collections that isn't using
any indexes:

{% highlight text %}
db.data.count()
4

db.data.find({name:'jane'}).explain()
{
	"cursor" : "BasicCursor",
	"isMultiKey" : false,
	"n" : 1,
	"nscannedObjects" : 4,
	"nscanned" : 4,
	"nscannedObjectsAllPlans" : 4,
	"nscannedAllPlans" : 4,
	"scanAndOrder" : false,
	"indexOnly" : false,
	"nYields" : 0,
	"nChunkSkips" : 0,
	"millis" : 1,
	"indexBounds" : {

	},
	"server" : "eshortiss-OSX.local:27017"
}
{% endhighlight %}

As you can see from the *count()* call, the database has 4 entries and explain
shows that all four were scanned (*nscanned*) to fulfill this query. Let's add
an index on the *name* property and see how we fare.

{% highlight text %}
db.data.ensureIndex({name: 1})
db.data.find({name:'jane'}).explain()
{
	"cursor" : "BtreeCursor name_1",
	"isMultiKey" : false,
	"n" : 1,
	"nscannedObjects" : 1,
	"nscanned" : 1,
	"nscannedObjectsAllPlans" : 1,
	"nscannedAllPlans" : 1,
	"scanAndOrder" : false,
	"indexOnly" : false,
	"nYields" : 0,
	"nChunkSkips" : 0,
	"millis" : 1,
	"indexBounds" : {
		"name" : [
			[
				"jane",
				"jane"
			]
		]
	},
	"server" : "eshortiss-OSX.local:27017"
}
{% endhighlight %}

After adding indexes the query has improved significantly and knew exactly where
to find what we needed without scanning the entire collection. We used this same
technique with our application. It turns out that while using non relational
databases offer enormous flexibility and allow you to change data structures
on a whim, you do need to ensure indexes are updated accordingly.

After updating our database indexes our API was happily serving over 2000
users, but we were still short of our target.

### stream all the things?
If you've been using node.js for a while you're probably aware that it has a
concept of streams. Streams are an awesome way to write elegant and performant
code that efficiently loads only small buffers of data into memory
(Readable) before sending them back out down a new (Writeable) stream. In a
nutshell, streams allow you to keep memory usage very low since they avoid
buffering then entire contents of a HTTP request or file into memory
to send it elsewhere, instead sending it piece by piece as it becomes available.

Here's a simple example of a file server that utilises streams:

{% highlight js %}

var fs = require('fs');
var app = require('express')();

app.get('/:filename', function (req, res) {
  // Just assume the file exists, create a stream to read it, and stream the
  // data back out to the http response object - amazing!
  fs.createReadStream(req.params.filename).pipe(res);
});

{% endhighlight %}

This seems awesome, right? We thought so too, and as a result decided to stream
MongoDB responses in this manner using the database _cursor.next()_
functionality to fetch responses one Object at a time and write them to the
HTTP response stream. Surely this was better than buffering hundreds of JSON
Objects from MongoDB into memory all at once, right? Wrong.

For our scenario,
this meant that if we had ~10,000 concurrent requests, each receiving 100
Objects, over 1,000,000 function calls would be made back and fourth between our
application code and the MongoDB node.js driver to fulfill all requests.
When we changed our code to get all data in a single call using *toArray()*
like the one below, we were happily serving the load that over 3,500 users
would generate.

{% highlight js %}

var app = require('express')();
var mongo = require('./mongo');

app.get('/data', function (req, res, next) {
  function onMongoData (err, data) {
    if (err) {
      next(err);
    } else {
      res.json(data);
    }
  }

  mongo.getCollection('data').find({
    userId: req.query.userId
  }).toArray(onMongoData);
});

{% endhighlight %}


### summary
* code reviews are important
* devops and MongoDB afford very flexible schemas, just don't forget to index
* node.js streams are not optimal for all situations
