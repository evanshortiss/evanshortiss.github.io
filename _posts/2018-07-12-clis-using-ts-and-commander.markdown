---
published: false
title: Creating a Node.js CLI using TypeScript and Commander
layout: post
categories: development nodejs javascript typescript
---

## Introduction
Creating a Node.js CLI with TypeScript is something I've done twice now.
Typically I'll use the [commander](https://npmjs.com/packages/commander) module 
when creating a Node.js CLI in plain JavaScript, so doing so using TypeScript 
seems like a no brainer. Choosing `commander` makes sense to me since it's a
batteries included type module, and does nice things like automatically
generating help output, parsing incoming options, outputs version info, supports
sub-commands like Git (type "git" in your terminal then press tab and you'll
see what I mean) and generally does nice things for us.

In this blogpost I'm going to outline the process I followed when creating my
latest CLI. Trial and error was required to come up with this structure that I 
am personally pleased with given my needs, but if you think it's shit or have a 
method you prefer then, as we'd say in Ireland,
[that's grand](https://www.irishslang.info/general-slang/general/thats-grand)!

We're just going to create a simple really IP address utility that supports two
commands:

* `ipprint get <interface>` - Retrieve our public IP or IP of attached interfaces
* `ipprint list` - List our public and private IP address

Before getting started make sure you have Node.js v8+ and npm v5+ installed. I
personally use [nvm](https://github.com/creationix/nvm) to install and manage my
Node.js versions so I can simply run `nvm use $NODE_VERSION` when I need to use
a specific version of Node.js, and then run `npm install -g npm@$NPM_VERSION`
for the version I need.

## Project Structure
The general structure of the repository for our CLI will be as follows:

```
├── bin
│   ├── command-name.ts
├── src
│   ├── cmd
│   │   ├── command-name-implementation.ts
│   ├── other-modules.ts
├── tsconfig.json
├── package.json
```

You can quickly initialise it by doing the following:

```bash
$ mkdir ipprint
$ cd ipprint
$ mkdir -p src/cmd
$ mkdir bin

# Create our package.json and fill in some details
$ npm init

# Install development dependencies
$ npm intsall typescript @types/node -D

# Install dependencies used at runtime
$ npm install commander axios -S
```

Nothing strange so far if you've worked with modern JavaScript, besides the
`tsconfig.json`. If you're new to TypeScript then all you need to know is that
the `tsconfig.json` defines rules for the `tsc` program to compile our `.ts`
files to `.js`. Here's what it looks like:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "noImplicitAny": true,
    "declaration": true,
    "preserveConstEnums": true,
    "sourceMap": true,
    "target": "es5",
    "lib": [
      "es2015"
    ],
    "strict": true,
    "removeComments": false
  },
  "include": [
    "src/*.ts",
    "bin/*.ts"
  ]
}
```

## Entry Point
Since we're building a CLI we'll need to register the entry point for our
program in our `package.json` so when it's installed it can be added to our PATH
and we're able to easily invoke it. To do this we need to add a
[bin](https://docs.npmjs.com/files/package.json#bin) field to our `package.json`
file like so:

```json
{
  "name": "ipprint",
  "version": "0.1.0",
  "bin": "bin/ipprint.js"
}
```

Note that we specified just `bin/ipprint.js`. This means that npm will register
the module name (ipprint) as the command name that we invoke from our terminal
and it will point to our `bin/ipprint.js` file. 

```ts
#!/usr/bin/env node

// Import commander using our decorated instance
import program from '../src/program'

// Register our subcommands
program.command(
  'search',
  'finds specific xyz'
)
program.command(
  'get',
  'retrieve a specific xyz'
)

// If no command is provided list the entire help output
if (!process.argv.slice(2).length) {
  program.outputHelp()
} else {
  program.parse(process.argv)
}
```

## Sub-Command Implementation


```ts
#!/usr/bin/env node

import ansible from '../ansible'
import { join } from 'path'
import { notify } from 'node-notifier'
import * as Listr from 'listr'
import * as oc from '../oc'
import * as docker from '../docker'
import * as mcp from '../mobile-core'

const input = require('listr-input')

let url: string

interface DownloadContext {
  releases: mcp.GitHubTag[]
  release: string
}

export interface CommandOptions {
  dockeruser?: string
  dockerpass?: string
  tag?: string
  branch?: string
}

export interface FuncOptions {
  dockeruser: string
  dockerpass: string
  tag?: string
  branch?: string
}

export async function run(opts: FuncOptions) {
  const tasks = new Listr([
    {
      title: 'Verify OpenShift 🔴',
      task: (ctx, task) => {
        task.output = 'Checking "oc version"...'
        return oc.verifyInstallation()
      }
    },
    {
      title: 'Verify Ansible 🤖',
      task: (ctx, task) => {
        task.output = 'Checking "ansible --version"...'
        return ansible()
      }
    },
    {
      title: 'Verify Docker 🐳',
      task: (ctx, task) => {
        task.output = 'Running "docker version"...'
        return docker.verifyInstallation()
      }
    }
  ])

  await tasks.run()
  console.log(
    `\n📱  OpenShift Origin with Mobile Core is available at: ${url}\n`
  )
  console.log('Mobile Services will be available via the Catalog in a few moments.')

  notify({
    title: 'AeroGear Mobile Core',
    icon: join(__dirname, '../../aerogear.png'),
    message: `Deployment of Mobile Core on OpenShift complete`,
    sound: true
  })
}
```

## Sub-Command Binary

```ts
#!/usr/bin/env node

// Import our command implementation that will be invoked
import * as up from '../src/cmd/up'

// Import our wrapped commander module to ensure we report errors
import program from '../src/program'

// Define our command options
program
  .option(
    '-u, --dockeruser <s>',
    'Docker Hub username. Defaults to DOCKERHUB_USER environment variable.'
  )
  .option(
    '-u, --dockerpass <s>',
    'Docker Hub password. Defaults to DOCKERHUB_PASS environment variable.'
  )
  .option(
    '-t, --tag <s>',
    'Specifies a Mobile Core tag to deploy.'
  )
  .option(
    '-b, --branch <s>',
    'Deploys a specific branch Mobile Core by using the master branch.'
  )

// Parse the incoming arguments
program.parse(process.argv)

// Create our options object for the command we are going to run
const opts: up.CommandOptions = {
  tag: program.tag,
  branch: program.branch,
  dockerpass: program.dockerpass || process.env.DOCKERHUB_USER,
  dockeruser: program.dockerpass || process.env.DOCKERHUB_PASS
}

// Perform validation checks
if (opts.tag && opts.branch) {
  throw new Error('--branch and --tag cannot be used together, pass either branch or tag')
} else if (!opts.dockerpass || !opts.dockeruser) {
  throw new Error('--dockeruser and --dockerpass are required, or DOCKERHUB_USER and DOCKERHUB_PASS environment variables must be set')
} else {
  // Invoke our command implementation with valid options
  up.run(opts as up.FuncOptions)
}
```

## Boilerplate
Generally we want each command to exhibit the same behaviour, particularly as it
relates to error handling. Since each of our commands are broken down into
subcommands we'd potentially create a bunch of repetitive code to achieve this, 
so instead we can simply create a module that re-exports commander, but with our
boilerplate pieces added.

This is as simple as creating a file `src/program.ts` and adding the following
contents.

```ts
import * as program from 'commander'
import chalk from 'chalk'

function errorHandler(e: any) {
  if (e.stack) {
    console.log(`\n ${chalk.red(e.toString())}`)
  } else {
    console.log(`\n ${chalk.red(e)}`)
  }
}

process.on('uncaughtException', errorHandler)
process.on('unhandledRejection', errorHandler)

export default program
```

Now, whenever we write a subcommand, instead of requiring `commander` directly 
we require our `src/program.ts` file since it guarantees that we will have
sensible and consistent error handling implemented for our CLI.


##