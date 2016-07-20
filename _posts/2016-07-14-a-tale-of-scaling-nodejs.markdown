---
published: false
title: Small Scale - MongoDB and Node.js
layout: post
categories: development javascript nodejs
---

## simple right?
On a recent client engagement we had the task of building a RESTful API using
node.js. This API would run in a Microservice that another would call on behalf
of mobile devices to synchronise data to client devices. It would also accept
payloads from devices and send them through an ActiveMQ when required.
Simple right? All was going well until we began load testing with datasets that
accurately reflected production.

## trouble on the horizon
As we reached around over 700 users we began to see dramatic performance
issues with response times spiking from sub 1 second to over 45 seconds for one
call in particular - concerning news considering we needed to support 4 times
that load! Most calls became slower in a linear fashion which was at least a
little reassuring.

Doing some napkin math we can determine our required throughput. Our API has 4
datasets exposed that users synchronise every 2 minutes via HTTP requests.
Since we're catering for the worst case scenario that means we need to be able
to sustain approximately 12,000 concurrent requests if we include some of the
less commonly hit routes that it exposes.

## the game plan
We held a team meeting to plan investigate these issues and tackled it from
these channels:

* inefficient queries
* database indexing
* code analysis

### inefficient queries
Almost immediately we found a query that had slipped through the cracks. The
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
some SQL.

{% highlight sql %}
UPDATE WHERE 1=1 SET colname=TODO
{% endhighlight %}

Oh no. That's not good. This was written in the early days of the application
and was never examined since. If you ever thought that implementing a
code review process might be a good idea, now you know it is. In high pressure
environments people can easily overlook simple things when rushing to get
features completed. Code reviews ensure that at least two people view all code
and increases the likelihood things like this are spotted before they slip
through the cracks.

Fixing this query immediately resolved the huge spike we saw in all requests
when testing with data similar to production volume, but we weren't out of the
woods just yet since our response times continued to climb rapidly after we
hit 1000 users.

### database indexing
Next we turned to database indexing. Since we were using MongoDB it was easy to
gauge the efficiency of queries using the _explain_ functionality offered.

Here's an example of a simple query that isn't using any indexes:

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

As you can see, the database has 4 entries and explain shows that all four were
scanned to fulfill this query. Let's add an index and see how we fare.

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
on whim, you need to remember to update your indexes accordingly.

After updating our database indexes our API was happily serving over 2000
users, but we were still around 1000 users short of our target.

### early optimisation is the root of...
If you've been using node.js for a while you're probably aware that it has a
concept of streams, although you might not use them too often. Streams are an
awesome way to write elegant and performant code that efficiently loads only
small buffers of data into memory before sending them back out down a new
stream. In a nutshell, streams allow you to keep memory usage low since you
don't have to buffer then entire contents of a HTTP request or file into memory
to send it elsewhere. Here's a simple example of a file server that utilises
stream:

{% highlight js %}

var fs = require('fs');
var express = require('express');

app.get('/:filename', function (req, res) {
  // Just assume the file exists, create a stream to read it, and stream the
  // data back out to the http response object - amazing!
  fs.createReadStream(req.params.filename)
    .pipe(res);
})

{% endhighlight %}
