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
      next(err);
    } else {
      cache.set('/users', users);
      res.json(users);
    }
  }

  cache.get('/users', function onCacheRes (err, users) {
    if (err) {
      legacy.getUsers(onUsers);
    } else if (users) {
      onUsers(null, users)
    } else {
      legacy.getUsers(onUsers)
    }
  });
});

{% endhighlight %}

There's nothing particularly wrong with this code, but I'd rather not have to
write this for each of my routes since it will mean more testing effort,
increased likelihood of bugs, and is not [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).
Also, what if you need custom caching logic based on the response from the
_getUsers_ call? This can get messy pretty quickly.

### npm to the rescue?
A better solution to this problem is to create an express middleware that does
the caching for you. Naturally I thought "surely this has been done already",
and headed to npmjs.org to find such a module, but was met with some
disappointment. Most of the modules in the wild perform caching, but come with
caveats:

* They override methods on the _response (res)_ Object which can lead to
errors and confusion as well as slow down in response time if done incorrectly
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

Since I tend to be a perfectionist at times, these issues wouldn't stand! There
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
with the existing express modules and it also serves as a standalone caching
solution that can be used in any node.js or JavaScript application.

The end result? Pretty neat:

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
  engine: require('expeditious-engine-memory')(),
  // Parse objects as they enter/exit cache
  objectMode: true
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

// Later in your application...
cache.keys({}, function (err, keys) {
 if (err) {
   console.error('failed to get keys', err);
 } else {
   // Delete/modify keys using cache.del/cache.set
 }
});

{% endhighlight %}


### summary
There are many ways to architect a solution, and while this works for some
node.js applications, you might have another approach or layer for solving your
caching needs.

Go forth and cut those response times down to size!
