---
published: true
title: Sharable JavaScript Modules
layout: post
categories: development javascript
redirect_from:
  - /development/javascript/2014/09/18/sharable-javascript-modules.html
---


PhoneGap/Cordova is enabling the development of mobile applications that are written using web technologies, but these applications aren't just mobile websites; they're fully fledged mobile applications that can leverage on native device functionality and provide smooth hardware accelerated animations using CSS3 for a native-like experience. With the advent of Node.js server-side JavaScript became a realistic, scalable option and also made structuring JavaScript far easier than we were used to in previous environments.

Sharing similar environments on client and cloud enables us to share code between both, and you'll be surprised to see just how easy this sharing can be.

Using CommonJS design patterns and one of my favourite tools, Browserify, makes it easy to share code. If you're skeptical then check out these projects I wrote that work on the client and server with little or no environment detection required:

* [Hype JS API](https://github.com/evanshortiss/hype.js) - Get track lists, album art and keys required to play tracks
* [Vec2D (Vector2D)](https://github.com/evanshortiss/vec2d) - 2D Vector library focused on flexible performance
* [fhlog](https://github.com/evanshortiss/fhlog) - Logger with a consistent interface on client and cloud.
* [safejson](https://github.com/evanshortiss/safejson) - Safely parse JSON via callbacks

# Benefits
So how does this sharing really benefit you?

* Shared, consistent APIs on client and cloud
* NPM can be used for dependencies on client and cloud
* A single file will be created meaning less load on your website if you Browserify your JS.
* Easily share business logic between client and cloud.
* Remain in a consistent programming paradigm

# The How

### Prequisites
You'll need Node.js, NPM and Browserify installed to accomplish this. NPM comes bundled with all recent version of Node.js so just download Node.js if you don't already have it. To install Browserify just run the following command:

```
$ npm i browserify
```

### Development
There's not much to cover here if you know how Node.js' CommonJS module system operates. You write your JavaScript in the usual Node.js style and can then run your code on the server just like you're used to doing, but the real idea here is to make these operable on a client browser!

To run your module in a browser you'll need to _Browserify_ it. This is the easy part! As an example assume we wrote a module called _sqaured_ and want to run it in a browser.

Here's the module code:
{% highlight javascript %}
/**
 * file: squared.js
 * Takes a number input and returns a squared result
 */

module.exports = function (num) {
  return Math.pow(num, 2);
};
{% endhighlight %}

And here's how we can package it for a browser:

```
$ browserify -e ./lib/sqaured.js -s sqaure -o ./browser/sqaure.js
```

So what does the above command do? It takes an entry point (-e) and creates a standalone module (-s) with the name _sqaure_ which is bound to the _window_ variable and writes the resulting code (-o) to _./browser/sqaure.js_.

Here's how we could use it:

{% highlight javascript %}
window.sqaure(2);
{% endhighlight %}

Any function that is bound to the exports object in your entry script (-e) is made accessible on the _window_ object.

### What about core Node.js modules?
So you're probably wondering how things like _require('http')_ or _require('util')_ work if you're running in the browser and what about UDP (dgram) and TCP (net). Right? Well that's easy enough. Where possible Browserify will provide the original module, for example _util_, _querystring_, _assert_ and _events_. In the case of others a shim is applied if available but naturally browsers can't send UDP packets willy nilly and open TCP sockets (although WebSockets exist!). Take a look [here](https://github.com/substack/node-browserify/blob/master/lib/builtins.js) to see shims vs inclusion of the standard Node.js core module.


### So. How can I handle missing native modules?
When writing modules that will be shared across the client and cloud you need to change very little in your coding practices as demonstrated. One area that will need to change is accessing native functions such as HTTP.

You need to write a module with a common interface for accessing HTTP or whatever non-standard functionality you require. You should probably do this for most projects anyway to avoid code duplication. Because this module has a common interface you simply swap out the implementations based on environment, and this swapping won't require extra coding, which is pretty sweet.

Below is an example of how this can be handled using Browserify.

Node.js HTTP abstraction:

{% highlight javascript %}
/**
 * file: http-wrapper.js
 * This file will be used for HTTP when
 * running in a node env.
 */

var request = require('request');

exports.get = function (url, callback) {
  request.get(url, callback);
};
{% endhighlight %}

Browser HTTP abstraction:

{% highlight javascript %}
/**
 * file: http-wrapper-browser.js
 * This file will be used for HTTP in browsers
 */

var xhr = require('xhr');

exports.get = function (url, callback) {
  xhr(
    {
      method: 'GET',
      url: url
    },
    callback
  );
};
{% endhighlight %}

So how do you use your HTTP abstraction? The way you always do, _require_ it!

{% highlight javascript %}

/**
 * file: index.js
 */

var http = require('./http-wrapper.js');

function onResponse (err, res, body) {
  if (err) {
    console.error('Oh no!');
  } else {
    console.log(body);
  }
}

http.get('http://www.google.com', onResponse);
{% endhighlight %}

You may have noticed we used the Node.js HTTP wrapper in _index.js_ in the above example. We do this as the  default should always be to use the Node.js version. When Browserifying this module we can substitute the Node.js wrapper with the browser version by adding a configuration in the _package.json_ of this project using the _browser_ field. The browser field below is read when Browserify is bundling our project and replaces all requires for _http-wrapper.js_ with _http-wrapper-browser.js_. Now that's pretty darn cool.

```
{
  "name": "GoogleGetter",
  "version": "0.0.0",
  "description": "A simple module sharing example",
  "main": "./index.js",
  "author": "Evan Shortiss",
  "license": "MIT",
  "browser": {
    "./http-wrapper.js": "./http-wrapper-browser.js"
  },
  "dependencies": {
    "xhr": "*",
    "request": "*"
  }
}
```

Now we can Browserify this module and use it in either environment!

If you're really sharp you might say "Hey! _xhr_ and _request_ share a very similar API. Why not just have one http file and replace _request_ with _xhr_?" Well. You're right! We could have written a single _http.js_ like so:

{% highlight javascript %}
/**
 * file: http.js
 */

var request = require('request');

exports.get = function (url, callback) {
  request({
    method: 'GET',
    url: url
  }, callback);
};
{% endhighlight %}

With this all we need to do is update the _package.json_ to have the following _browser_ field, but for the sake of the example it seemed appropriate to assume the APIs would be different.

```json
{
  "browser": {
    "request": "xhr"
  }
}
```


# Using Client Modules via NPM
If you published this module to NPM and wanted to use it in another client-side project you wouldn't need to target the Browserified build. Instead, Browserify will be clever enough to also build this module despite the fact it's a dependency.

Go forth and Browserify!
