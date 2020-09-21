---
published: true
title: Setting Bower Dependencies to a SHA
layout: post
categories: development javascript
redirect_from:
  - /development/javascript/2015/04/12/bower-dependencies-sha.html
---

This is just a quick one as a reminder to myself and a tip to everyone else.

Today I needed to set a bower dependency to a specific commit SHA. It was surprisingly more dificult than I expected. After tying many different command syntaxes etc. that littered Stackoverflow I eventually added the following to my bower.json.

```
{
  "dependencies": {
    "angular-material": "git@github.com:angular/material.git#dc93bffe25cd5b2c28e6eac7adfe954c69c672c6"
  }
}
```

After adding this I received no strange #128 errors and other git issues.

Hope this helps someone else!
