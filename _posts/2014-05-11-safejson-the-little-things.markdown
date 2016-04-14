---
published: true
title: safejson - It's the little things in life!
layout: post
categories: nodejs node.js javascript npm json
---

Humans. We're lazy. We also like things to be aesthetically pleasing. OK, Maybe that's just me.

As a developer who frequently works with JSON I don't like seeing a try catch each time JSON is handled in JavaScript, particularly in the realm of Node.js. That's why I wrote [safejson](https://www.npmjs.org/package/safejson); a tiny JavaScript utility to parse JSON using Node.js style async callbacks. It's compatible with browsers and Node.js environments. Win win!

Using try/catch works fine.

```javascript
try {
	return callback(null, JSON.parse(str))
} catch (e) {
	return callback(e, null);
}
```

But this is much nicer!

```javascript
safejson.parse(SOME_STRING, callback);
```

How about *JSON.stringify*? Yup.

```javascript
safejson.stringify(SOME_JSON, function(err, jsonstr) {
  if(err) return callback(err, null);

  doAnotherAsyncOp(jsonstr, callback);
});
```

It also accepts all the regular arguments that can be passed to the *parse* and *stringify* functions which is an added bonus and pretty useful.
