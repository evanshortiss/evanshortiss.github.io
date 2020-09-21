---
published: true
title: Timers in TypeScript and Node.js
layout: post
categories: development nodejs typescript
redirect_from:
    - /development/nodejs/typescript/2016/11/16/timers-in-typescript.html
---

TLDR; Use `global.setTimeout` instead of `setTimeout`

I recently started playing with TypeScript and find that I'm really enjoying it.
Rather than starting a new project for my first foray into this new JavaScript
superset, I'm converting an older one.

During this conversion to TypeScript I've run into a minor few issues, but this
seemed worth sharing. Basically I needed to use a `setTimeout` in my code to
defer an operation, but the available types of `NodeJS.Timer` and `WindowTimers`
were confusing. Which is correct? At first it appeared neither was correct since
I was receiving the following error `[ts] Type 'number' is not assignable to
type 'Timer | null'.`

Eventually I figured maybe there was some conflict in term of the timer types in
node.js vs. browser JavaScript so I tried the following:


```bash
let timer:NodeJS.Timer;

timer = global.setTimeout(myFunction, 1000);
```

By specifying `global.setTimeout` (the equivalent of `window`, in node.js) all
was cleared up and the TypeScript compiler was happy once again.
