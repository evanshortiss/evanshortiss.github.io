---
published: true
title: Node-ifying the HTML5/Cordova FileSystem API
layout: post
categories: browserify javascript node.js nodejs npm cordova
---


This post assumes you're vaguely familiar with the HTML5 FileSystemAPI and the equivalent Cordova implementation of the API [org.apache.cordova.file](https://github.com/apache/cordova-plugin-file). Check out this awesome [HTML5 Rocks article](http://www.html5rocks.com/en/tutorials/file/filesystem/) about the FileSystem API to get acquainted. Although you can probably appreciate this without reading all of the linked articles.

Back in April this year I started working on this project in Boston Logan Airport after my flight home was delayed by 2 hours. Now I've finally finished it, by once again by doing some work whilst waiting on a Christmas flight home in Logan Airport and wrapping things up back home in Ireland. That's a pretty long timeline! The word "finished" is used loosely above; the project does what I need it to for now which is pretty much "complete".

## The Project
As part of another project I require an easy to use manner of accessing the HTML5 FileSystem; this means I absolutely don't want to use the plain old FileSystem API like so:

```javascript
function writeFile(success, fail) {
  // fileSystem can be assumed defined previously
  fileSystem.root.getDirectory('/test/', {
    create: true,
    exclusive: false
  }, function(dirEntry) {
      dirEntry.getFile('somefile.txt', {
        create: true,
        exclusive: false
      }, function(file) {
        file.createWriter(function(writer) {
          writer.onwrite = function(evt) {
              success(null);
          };

          writer.onerror = function(evt) {
              fail(evt.target.error);
          };

          writer.write(new Blob([data], {
            type: 'text/plain'
          }));
      }, fail);
    }, fail);
  }, fail);
}
```

There's nothing wrong with the FileSystem API really, but that's far to much effort for one to read and write (no pun intended). I'd prefer a Node.js style API like the one below:

```javascript
function writeFile(callback) {
  fs.mkdir('test', function (err) {
    if (err) {
      callback(err, null);
    } else {
      fs.writeFile('test/somefile.txt', 'Hello world', callback);
    }
  });
}
```

Clearly the second piece of code here is more concise, understandable to anyone (especially if they've used Node.js), and much less error prone to write. Oh, it also uses a single callback pattern so you can pop it into [async](https://github.com/caolan/async) calls without the need to wrap your success and failure callbacks. Nice, right!?

## Introducing _html5-fs_
Well that second example above is exactly what I did with the module [html5-fs](https://www.github.com/evanshortiss/html5-fs). I ended up developing the module as part of a component for another small project I was working on. My reason for doing this was because the other project is also a Node.js module wrapped using Browserify, but needs the ability to read and write to the file system which requires two very different APIs, one on the client (primarily Cordova applications) and one on the cloud (Node.js). I didn't feel like writing an abstraction that would bridge the calls within the project; instead I wanted a module that I could directly drop in as a replacement for the standard Node.js module on the client with as few differences as possible. In other words I'd just add the following to my _package.json_ and wouldn't need any extra logic to handle client vs. server environments:

```json
{
  "browser": {
    "fs": "html5-fs"
  }
}
```

This line would tell browserify that when it was bundling my JavaScript that any require calls to _fs_ would be replaced with _html5-fs_. Sounds bizarre? Probably, but because _html5-fs_ has the same interface as _fs_ it works, albeit with some exceptions if you need to use the entire feature set of the Node.js API.

Another project I'm tinkering on already works using this module as a drop in replacement for _fs_ when bundled for the client. It's a JavaScript logger that works on client and cloud and writes logs to disk storage for upload to a server later when the application has time to do so. This feature will probably be primarily used by mobile applications running the logger. The [repo is here](https://github.com/evanshortiss/fhlog) if you want an example of a project using this module.

## Creating the Wrapper

#### Dual Callback Translation
The FileSystem API uses a dual callback structure by default. Node.js uses a single callback structure and manages errors by passing an error, if one occurred, as the first parameter to the callback with results passed as additional parameters.

Users of _html5-fs_ need only provide a single callback as this is wrapped via the below functions and then passed to the native FileSystem calls for you. Essentially these functions just take the args to the usual success and failure callbacks and provide them as a tuple _[err, result]_ to be passed to your provided single callback as args depending on the result.

```javascript

/**
 * Wrap a callback for use as a success callback.
 * @param    {Function} callback
 * @return   {Function}
 */
exports.wrapSuccess = function(callback) {
  return function() {
    var args = [null].concat(Array.prototype.slice.call(arguments));

    callback.apply(callback, args);
  };
};


/**
 * Wrap a callback for use as a failure callback.
 * @param    {Function} callback
 * @return   {Function}
 */
exports.wrapFail = function(callback) {
  return function() {
    var args = Array.prototype.slice.call(arguments)
      , e = args[0];

    callback.apply(callback, [e, null]);
  };
};

```

#### Browser Inconsistencies
Thankfully browser inconsistencies have thus far been minimal, with the minor exception of how they expose their FileSystem initialisation API calls.

Chrome and Opera expose either _window.webkitStorageInfo_ and/or _window.navigator.webkitPersistentStorage_ whilst the Cordova API exposes _window.requestFileSystem_.

All of these expose a method of requesting a FileSystem instance. For example:

```javascript

// Most recent API in browsers
window.navigator.webkitRequestFileSystem(
	bytes,
    success,
    fail);

// Deprecated browser API
window.webkitRequestFileSystem(
      window.PERSISTENT,
      bytes,
      success,
      fail);

// Used in Cordova apps and older browsers
window.requestFileSystem(
      window.LocalFileSystem.PERSISTENT,
      bytes,
      success,
      fail);

```

Using _html5-fs_ users only need to call the following and not worry about the calls demonstrated above:

```javascript
fs.init(bytes, callback);
```

You might rightly say something such as "Hey, that's not compatible with the Node.js _fs_ module, we can't just drop it in as a replacement!" and you'd be _almost_ right. Unfortuantely it's required, but can easily be worked around in a few ways, the simplest of which is provided below.

```javascript

function initApplication(callback) {
	if (fs.init) {
    	// We're using html5-fs
  		fs.init(bytes, callback)
	} else {
    	// Using Node.js fs, no init required
    	callback(null, null);
    }
}


```


#### Debugging
The Android WebView proved to be a little more problematic to work with than other browsers, or could be better described as finicky. Tests that were passing on iOS and desktop browsers were failing on Android. It took me a long time to figure out why these were failing. The first issue was that Blob wasn't supported on the Android browser so writing files failed due to an incorrect platform check that I was performing. It looked a little like this

```javascript
if (isMobile()) {
  // I never got in here on Android
  writer.write(data)
} else {
  // This was always executed
  writer.write(new Blob([data], {
    type: type
  }));
}

```

Another issue I encountered was that on Android if I tried to create a directory that wasn't at the root of the application's storage directory it would always fail with a timeout error. Digging deeper with _adb logcat_ revealed that a null pointer error was thrown in the native FileSystem plugin code whenever such a call was made. I resolved this by getting a reference to the containing directory first and then creating the new file or folder using that reference; not a perfect solution but a workable one.

[Weinre](https://www.npmjs.com/package/weinre) is your friend when it comes to debugging Cordova apps and was great here as it gave me a JavaScript playground to run my library on Andoird in an interactive manner. Just run it like so:

```
npm i -g weinre
weinre --httpPort {PORT} --boundHost {YOUR-IP}
```

Then visit {YOUR-IP}:{PORT} in a web browser and add the script tag it provides to your Android or iOS application _index.html_. Alternatively with iOS you can also use the Safari debugger to connect to the iOS Simulator or an application that was signed with a developer profile running on device which is awesome.

#### Testing
Naturally, I wanted to automate testing as much as possible but doing this with on-device testing for Cordova applications isn't as straightforward as just spinning up a Karma server and directing the device browser to it; this wouldn't allow us to test the Cordova environment which was a requirement but was fine for desktop Chrome and Opera (the only two browsers that support the FileSystem API).  

For Cordova applications on iOS and Android I performed testing by dumping the required files in the Cordova _www_ directory and setting up a test environment using the _mocha init_ command. The Cordova _www_ folder contains all the test files that the browser uses so both run th exact same tests. To run the tests Karma is used for browsers and _cordova emulate [platform]_ is used for iOS and Android.

Testing can be carried out by running either:

```
grunt test // Tests browsers (Opera and Chrome)
grunt test-ios
grunt test-android
```


The following output is shown when running the iOS, Android, and Karma tests respectively.

![iOS Simulator](https://dl.dropboxusercontent.com/u/4401092/blog/images/2014/Dec/html5_fs_ios.png)

![Android Emulator](https://dl.dropboxusercontent.com/u/4401092/blog/images/2014/Dec/html5_fs_android.png)

![Karma Tests](https://dl.dropboxusercontent.com/u/4401092/blog/images/2014/Dec/html5_fs_karma.png)


## What Next?
As alluded to previously this library could have far more functionality built in. Adding support for more standard Node.js _fs_ module functions such as _fs.watch_ or _fs.rename_ might be a nice start. Permissions and synchronous calls probably aren't required although could be useful. I'll add things as needed, and feel free to contribute to the [GitHub repo](https://github.com/evanshortiss/html5-fs/) for the project.
