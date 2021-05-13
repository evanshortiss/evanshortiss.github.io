---
published: true
title: Pretty Print for Pino Logs with Stern
layout: post
categories: kubernetes openshift nodejs
---

Posting this almost as a "note to self", so I'll keep it short.

Here's how to pretty print [pino](https://github.com/pinojs/pino) logs if\
you're using [stern](https://github.com/wercker/stern) to view them from a
Node.js Pod running on Kubernetes or OpenShift:

```bash
stern $POD_SELECTOR -o raw | npx pino-pretty -t
```

I use `npx pino-pretty`, but running `npm install -g pino-pretty` will allow
you to do the following:

```bash
# Install pino-pretty globally
npm install -g pino-pretty

# Use it to view the logs
stern $POD_SELECTOR -o raw | pino-pretty -t
```
