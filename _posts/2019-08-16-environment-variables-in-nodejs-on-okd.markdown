---
published: true
title: Environment Variables, TypeScript, and Containerised Applications
layout: post
categories: development
---

<!-- TODO ADD OLD BLOG POST LINK -->

When I originally wrote `env-var`, I wanted a module that provided a
layer of safety when writing applications that implemented the *Config* aspect
of the "The Twelve-Factor App". Since environment variables are exposed as
strings and can be `undefined` it's easy to make blunders using them.

At that time I wasn't interested in TypeScript, and I downplayed it's potential
value by saying things such as "My unit tests have me covered" or "just write
simple small functions". After trying TypeScript for real my opinion changed,
but I'm not here to sell TypeScript; I want to document how `env-var` adding
TypeScript support has been incredibly useful and how I use it in a workflow.

_Note: Before going any further I have to give [Mike Burkman]() a shout out. He led me down the TypeScript rabbit hole and helped develop the type definitions for `env-var`!_

So, I like to write slightly larger programs and applications using TypeScript.
I also enjoy using `env-var` even more than before since it automatically
exposes environment variables in the expected type. For example, assume a
program relies on an environment variable named `MOCK_DATA_ENABLED` being
set to either `true` or `false` and uses it in an `if` statement. Seems simple,
right?

```js
const MOCK_DATA_ENABLED = process.env.MOCK_DATA_ENABLED

if (MOCK_DATA_ENABLED) {
  // This will execute if MOCK_DATA_ENABLED=true
  // This will also execute if MOCK_DATA_ENABLED=false
  // Oh no! MOCK_DATAENABLED is a string not a boolean!
}
```

The program receives the `true` or `false` value as a *string*! This error
won't be flagged by a linter, so a unit test is necessary to catch it. It
would be easy to make this mistake if you're not paying attention, or just
haven't had enough tea, coffee, or `$BEVERAGE_OF_CHOICE. Avoiding the mistake
is simple with TypeScript if using the correct type annotations:

```ts
// Compiler error: Type 'string' is not assignable to type 'boolean'
const MOCK_DATA_ENABLED: boolean = process.env.MOCK_DATA_ENABLED
```

A simple adjustment to determine if the value is equal to a specific string
will then correct the compiler error.

```ts
const MOCK_DATA_ENABLED: boolean = process.env.MOCK_DATA_ENABLED === 'true'
```

This is an easy fix, but with numbers, URLs, and other data types this becomes
more complex. Using `env-var` makes it much cleaner due to its fluid API:

```ts
import * as env from 'env-var'

// asBool() will check MOCK_DATA_ENABLED is '0', '1', 'false', or 'true' and
// convert the value to a valid boolean value or throw and error
const MOCK_DATA_ENABLED: boolean = env.get('MOCK_DATA_ENABLED').asBool()

// required() will throw an exception if the variable is not set
// asUrlString() will thrown an error if the URL is malformed
const API_SERVICE_URL: string = env.get('API_SERVICE_URL').required().asUrlString()
```

## Rolling it Up

If I'm writing something simple I'll usually just use the `env.get` function
that `env-var` exposes directly as you saw above, but if I want to centralise
the config I'll sometimes create a module to expose it.

```ts
import { from } from 'env-var'

export interface Config {
  MOCK_DATA_ENABLED: boolean
  MAX_CONCURRENCY: number
  API_SERVICE_URL: string
}

export function getConfig (environment: NodeJS.ProcessEnv): Config {
  // Create an env-var instance with the given process.env object
  // During testing this can be an object with whatever values are required
  const env = from(environment)

  return Object.freeze({
    MOCK_DATA_ENABLED: env.get('MOCK_DATA_ENABLED').asBool() || false,
    MAX_CONCURRENCY: env.get('MAX_CONCURRENCY').asIntPositive() || 10,
    API_SERVICE_URL: env.get('API_SERVICE_URL').required().asUrlString()
  })
}
```

I can then easily use this anywhere in my program to fetch my environment
variables and be sure they are of the correct type and have been validated
for my program's requirements.

```ts
import { getConfig } from '@app/config.js'
import pMap from 'p-map'

const config = getConfig(process.env)

/**
 * Get info for each provided username
 */
export function getInfoForUsers (usernames: string[]) {
  // Fetch up to MAX_CONCURRENCY users in parallel
  return pMap(
    usernames,
    getUserInfo,
    {
      concurrency: config.MAX_CONCURRENCY
    }
  )
}

/**
 * Get info for a single user from an API using their username
 */
export async function getUserInfo (username: string) => {
  if (config.MOCK_DATA_ENABLED) {
    return {
      username,
      dob: '01/01/1970',
      firstname: 'Evan',
      lastname: 'Shortiss'
    }
  } else {
    // Create a URL object by combing the username and API_SERVICE_URL
    // e.g https://api.example.com/someuser
    const url = new URL(`/${username}`, config.API_SERVICE_URL)
    const response = await got(url)

    return response.data
  }
}
```

## Deployment and Development

When it comes to choosing modules to use in my Node.js application I tend to
favour modules that allow me to achieve something in the least intrusive manner
possible. That's why I tend to use `env-var` and `dotenv`. They're complimentary
and don't get in the way of each other.

* `env-var` - Validate environment variables and coerce to correct types
* `dotenv` - Define default environment variables using files

Combining both of these provides an elegant solution for managing application
configuration that's incredibly flexible and unopinionated. With `dotenv` you
can create a `.env` file in the root of your repository and then add
`require('dotenv').config()` to the entry point of your code to load everything
defined in the file into `process.env`. Personally, I prefer to invoke `dotenv`
as part of starting my program since this removes the need to explicitly call
it in code for its [side-effects](https://en.wikipedia.org/wiki/Side_effect_(computer_science)).
I also like to specify the file `dotenv` should read using its supported CLI
args like so:

`node -r dotenv/config application.js dotenv_config_path=./env/dev.env`

In my `application.js` I can then use `env-var` to perform some sanity checks
before launching my application.

```javascript
const env = require('env-var')

// Use the PORT defined in the environment. If none is set use 8080
const PORT = env.get('PORT').asPortNumber() || 8080
// Server logging must explicitly the environment (by dotenv or by the host)
const SERVER_LOGGING_ENABLED = env.get('SERVER_LOGGING_ENABLED').required().asBool()

const fastify = require('fastify')({
  logger: SERVER_LOGGING_ENABLED
})

fastify.listen(8080, '0.0.0.0', (err) => {
  if (err) throw err
})
```