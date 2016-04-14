---
published: true
title: Setting Bower Dependencies to a SHA
layout: post
categories: bower javascript
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
