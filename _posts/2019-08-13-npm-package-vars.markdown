---
published: true
title: npm script variables
layout: post
categories: development nodejs
redirect_from:
    - /development/nodejs/2019/08/13/npm-package-vars.html
---

Recently I learned about the [package.json vars](https://docs.npmjs.com/misc/scripts#packagejson-vars) feature of npm scripts.
This is a really neat feature that allows you to access fields from your
*package.json* as via variables in commands included in the `scripts` section.

The variables follow the format `npm_package_${FIELD}_${SUBFIELD}`. For
example, `npm_package_version` will contain the value of the `version`
field in your *package.json*. 

You can even access find dependencies and their versions. If you have express
4.16.0 in the `dependencies` section of your *package.json*, then an
environment variable named `npm_package_dependencies_express=~4.16.0` would be
available when executing your scripts!

This might seem trivial, but it came in useful in a stateless GitHub file
watcher I developed. You can see in the `container:tag` [script here](https://github.com/evanshortiss/stateless-github-file-watch/blob/master/package.json#L14)
that I used `npm_package_version` to create a Docker tag. Before I knew about
this feature I was in the process of writing some messy bash to parse out the
`version` field.

Many more useful fields are available so check them out for yourself. A quick
and easy way to list them is to add something like the following to your
`scripts` in a *package.json* and run it using `npm run list-vars`:

```
"list-vars": "echo $(env) | tr ' ' '\n' | grep -i 'npm_'"
```

_Note: This is not a perfect script, and is only tested on macOS. It splits on spaces so it won't work on fields such as `author` since that will typically contain spaces, but it's good enough to get the point across!_

The output below shows a subset of the variables this printed on my machine.
Some of these could be really useful in your scripts!

```
npm_package_license=MIT
npm_package_scripts_deploy=serverless
npm_package_dependencies_barelog=~0.1.0
npm_package_devDependencies_serverless=~1.43.0
npm_package_husky_hooks_pre_commit=npm
npm_package_bugs_url=https://github.com/evanshortiss/serverless-github-file-watch/issues
npm_package_version="1.0.0"
```


