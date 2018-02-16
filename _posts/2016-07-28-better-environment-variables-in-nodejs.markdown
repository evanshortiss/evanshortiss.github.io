---
published: true
title: Working with Environment Variables in node.js
layout: post
categories: development nodejs
---

TLDR; Reading environment variables, asserting their values, and parsing them
to the correct type, can be laborious in node.js. [env-var](https://github.com/evanshortiss/env-var) is an attempt improve upon this.

Utilising environment variables is an effective solution for managing
environment dependent configurations in a software solution regardless of the
language you're using. Here's how we can access environment variables in
node.js.

{% highlight javascript %}
var MAX_BATCH_SIZE = process.env.MAX_BATCH_SIZE;
{% endhighlight %}

If we run this program as shown below, then it would receive a String with the
value of the MAX_BATCH_SIZE environment variable.

```
$ MAX_BATCH_SIZE=10 node my-program.js
```

This is pretty simple stuff at a high-level, but is imperfect when it comes
to unit testing, verifying variables are set, and ensuring they are set
within specific constraints. Take our MAX_BATCH_SIZE for example; by default it
will be of type String when loaded into our program despite our desire for it to
be of type number, and how do we ensure it is set?

## assertions for variables and values
On a recent project we needed to ensure that certain variables were set, and
that they were set to valid values. We could have just read in the variables,
assumed they were correct and run the program, but that could easily lead to
unforseen runtime errors, even due to an innocent typo.

One such example was where we needed environment variables that controlled the
concurrency levels of certain operations. This resulted in the following type
of code.

{% highlight javascript %}
var assert = require('assert');

// Read the var, and use parseInt to make it a number
var MAX_BATCH_SIZE = parseInt(process.env.MAX_BATCH_SIZE, 10);

// Check the var is a valid number, if not throw
assert(
  typeof MAX_BATCH_SIZE === 'number' &&
  !isNaN(MAX_BATCH_SIZE) &&
  MAX_BATCH_SIZE >= 0,
  'MAX_BATCH_SIZE env var must be a positive number'
);
{% endhighlight %}

Each system that required variables had code similar to this meaning we were
writing repetitive, and error prone code. Besides the fact that it's not DRY,
increases testing effort, and margin for error - it just doesn't read well.
Sure we can extract the conditional logic into a named function, but it'd be
nice if something was available that did this for us since it's a simple and
common use case.

## testing and process.env
When it comes to testing code we need mechanisms that allow us to stub
out the environment object. In the case of node.js, the *process.env* object
is a mutable global variable. You might be
thinking *"awesome, I can just configure it inside each test case"*, but this
isn't a great idea since if you screw this up, the resulting corruption of
shared state could have negative affects on other test cases.
This shared state can be rage inducing to debug when it causes bugs. At the
very least it'll annoy you and waste valuable time, and is easily avoided.

As a solution, you could perform some sort of inversion of control setup where
you declare a module as demonstrated below, and can therefore inject a mock
*process.env*.

{% highlight javascript %}
// Loads our SQL config from env vars.
// Throws if a required val is missing
module.exports = function getSqlOptions (environment) {
  var ret = {};

  [
    'SQL_USER',
    'SQL_PASS',
    'SQL_HOST',
    'SQL_PORT'
  ].forEach(function (n) {
    ret[n] = environment[n];

    assert.equal(
      typeof ret[n],
      'string',
      'variable ' + n + ' must be set'
    );
  });

  // Return our created object that contains the vars from process.env
  return ret;
};
{% endhighlight %}

This solves the problem since your test can inject whatever it likes for
*environment*, thereby avoiding the dangers of shared state, but you need to
use this pattern everywhere *process.env* is required. You also still need to
perform assertions and parsing on the environment variables depending on their
use. Not a big deal, but not ideal either.

## modularising  
I'm privileged to work with some awesome folks, and during code reviews and some
conversations we agreed we could easily address these issues, and that's
what happened. We were already using, *env-var*, a module I had developed a few
months back to manage reading variables and to address the testing issues
discussed above, so I felt it necessary to improve this module after our chat.

The improvements implemented in the *env-var* module allow us to easily verify
our environment is configured correctly, and has vastly improved readability.

{% highlight javascript %}
var env = require('env-var');

// Verify the variable is set, parse to an integer, and ensure it's >=0
var MAX_BATCH_SIZE = env('MAX_BATCH_SIZE').required().asPositiveInt()
{% endhighlight %}

If MAX_BATCH_SIZE were not set or was a negative integer, an exception like the
following would be raised.

{% highlight text %}
VError: env-var: MAX_BATCH_SIZE should be a valid integer, but was undefined
    at Object.isInt (my-project/node_modules/env-var/lib/type-checks.js:36:15)
    at Object.types.isPositiveInt (my-project/node_modules/env-var/lib/type-checks.js:9:23)
    at Object.asInt [as asPositiveInt] (my-project/node_modules/env-var/lib/variable.js:24:20)
{% endhighlight %}

This one line of code is far more expressive than manual assertions and when
working on a team where code is being read as much as written this is really
beneficial.
