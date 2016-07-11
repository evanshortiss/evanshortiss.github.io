---
published: true
title: Better Caching for Express Applications
layout: post
categories: development javascript nodejs
---

### node.js is awesome
node.js is known for it's high throughput I/O capabilities and lightning fast
JavaScript execution engine, V8. Both of these together enable developers to
easily create high-performance, lightweight web applications. Despite these
features, bottlenecks are unavoidable at times in web applications, even those
built with node.js and the awesome express web framework.

### solving bottlenecks
Sometimes there's a simple solution for such problems; caching. By caching
responses from slower external systems at our node.js layer we can dramatically
reduce load on those systems and improve our response times in one fell sweep!
While this is a great solution, the implementation is often less than ideal and
frequently looks something like this.

{% highlight js %}
var cache = require('./cache')
  , legacy = require('./old-system-api');

app.get('/users', function (req, res, next) {

  function onUsers (err, users) {
    if (err) {
      // Error, pass it to the express error handler
      next(err);
    } else {
      // Store items in cache and respond
      cache.set('/users', users);
      res.json(users);
    }
  }

  cache.get('/users', function onCacheRes (err, users) {
    if (err) {
      // Error, we need to run the actual function
      legacy.getUsers(onUsers);
    } else if (users) {
      // Awesome, we got data from the cache!
      onUsers(null, users)
    } else {
      // No data was cached, gotta run the function
      legacy.getUsers(onUsers)
    }
  });
});

app.listen(process.env.APP_PORT || 3000);

{% endhighlight %}

There's nothing particularly wrong with this code, but I'd rather not have to
write this for each of my routes since it will mean more testing effort,
increased likelihood of bugs, and is not [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).
Also, what if you need custom caching logic based on the response from the
_getUsers_ call? This can get messy pretty quickly, and is often too intrusive
for me since it affects how functions are written.

### npm to the rescue?
A better solution to this problem is to create an express middleware that does
the caching for you. Naturally I thought "surely this has been done already",
and headed to npmjs.org to find such a module, but was met with some
disappointment. Most of the modules in the wild perform caching, but come with
caveats:

* They override functions on the _response (res)_ Object which can lead to
errors and confusion as well as slow down in response time if done incorrectly
* Some functions on the _response (res)_ Object are forgotten about meaning
caching does not occur and you're lost as to why.
* A cache store interface is not exposed so it's not possible to
programmatically interact with the underlying cache when you need to delete or
modify entries
* Using node.js memory for caching. This means your node.js process could use
far more resources than you'd like
* If it does use a different datastore then you're forced to use it, e.g redis
(redis is great, but might not be viable for everyone)
* Not accounting for parallel requests for the same resources, and therefore
resulting in many large buffers in memory resulting in memory and CPU spikes
* Lack of ability to override ttl for different status codes meaning a
temporary error such as a 500 could result in bad response being cached and
returned repeatedly for longer than necessary
* Unable to generate cache keys using a custom function meaning loss of control
over cache conflicts
* They're unable to cache piped responses, i.e _request.get(URL).pipe(res)_

Since I tend to be a perfectionist at times, these issues wouldn't stand. There
was only one solution; write yet another caching module.
I guess [this](https://xkcd.com/927/) is relevant?

### expeditious to the rescue?
In an either bold, or insane move I decided to try my hand at creating a better
solution to this problem and ended up creating the project _expeditious_. Needless
to say it got out of hand quickly, and currently there are three expeditious
related modules I've published:

* [expeditious](https://github.com/evanshortiss/expeditious) - The core caching interface
* [expeditious-engine-memory](https://github.com/evanshortiss/expeditious-engine-memory) - An engine that can be passed to a core instance
* [express-expeditious](https://github.com/evanshortiss/express-expeditious) - A caching middleware for express that uses an instance
of expeditious as the cache

The benefits of this modular approach are numerous:

* Core is lightweight and independent of the underlying data store
* Data stores can be created as needed. You need to store data in CouchDB? Go
ahead!
* API is standard regardless of storage engine used, meaning a single line of
code is the only change required to change the data store
* expeditious instances used by middleware such as express-expeditious are
available for programmatic modification meaning middleware is not a black hole

Beside this, express-expeditious offers a solution for the drawbacks I found
with the existing express caching modules. The expeditious core also serves as
a standalone caching solution that can be used in any node.js or JavaScript
application in a browser.

### using express-expeditious

The end result? Pretty neat I think:

{% highlight js %}
var expeditious = require('expeditious');
var app = require('express')();

// Our cache instance
var cache = expeditious({
  // Namespace for this cache
  namespace: 'express',
  // Default expiry
  defaultTtl: (120 * 1000),
  // Where items are stored
  engine: require('expeditious-engine-memory')()
});

// Middleware instance
var expressCache = require('express-expeditious')({
  expeditious: cache,
  statusCodeExpires: {
    // We don't want to cache these for the 2 minute default
    500: (30 * 1000),
    502: (60 * 1000)
  }
});

// Cache any GET requests that come into our application
app.use(expressCache);

// Route that responds in 5 seconds, will respond immediately for subsequent
// calls for the next 120 seconds thanks to express-expeditious
app.get('/ping', function (req, res) {
  setTimeout(function () {
    res.end('pong');
  }, 5000);
});

app.listen(3000);
{% endhighlight %}

This is a trivial example and is by no means the only way to use
express-expeditious. You can apply it to specific routes or _express.Router_
instances for granular control. Thanks to the manner in which these modules
work in unison, we can programmatically access the _cache.del_ method to remove
"/ping" if some event occurred and new data became available.

### the results

Taking a look at this fabricated test where only 25 concurrent requests can be
made, we see that the total time for 1000 requests goes from 80.4 seconds to
2.4 seconds. Not bad!

#### without cache

{% highlight text %}

$ loadtest -n 1000 -c 25 http://127.0.0.1:3000/not-cached
Target URL:          http://127.0.0.1:3000/not-cached
Max requests:        1000
Concurrency level:   25
Agent:               none

Completed requests:  1000
Total errors:        0
Total time:          80.420119519 s
Requests per second: 12
Total time:          80.420119519 s

Percentage of the requests served within a certain time
  50%      2008 ms
  90%      2016 ms
  95%      2022 ms
  99%      2040 ms
 100%      2047 ms (longest request)

{% endhighlight %}


#### with cache

{% highlight text %}

$ loadtest -n 1000 -c 25 http://127.0.0.1:3000/cached
Target URL:          http://127.0.0.1:3000/cached
Max requests:        1000
Concurrency level:   25
Agent:               none

Completed requests:  1000
Total errors:        0
Total time:          2.354407228 s
Requests per second: 425
Total time:          2.354407228 s

Percentage of the requests served within a certain time
  50%      6 ms
  90%      11 ms
  95%      48 ms
  99%      2018 ms
 100%      2027 ms (longest request)

{% endhighlight %}

### summary
There are many ways to architect a solution, and while this works for some
node.js applications, you might have another approach or layer for solving your
caching needs.

If you'd like to get started checkout the example for [_express-expeditious_](https://github.com/evanshortiss/express-expeditious/tree/master/example).

Go forth and cut those response times down to size!
