---
published: true
title: Prettify Logs using FHC and Bunyan
layout: post
categories: development rhmap
redirect_from:
    - /development/rhmap/2016/05/25/pretty-print-logs-using-fhc.html
---

*This post assumes you have a Red Hat Mobile Platform instance. You can sign up
for a community edition instance at
[openshift.feedhenry.com](https://openshift.feedhenry.com/) using your
OpenShift account.*

# Not so Pretty Logs

Just yesterday I blogged about the _fh-bunyan_ module, and how it makes
sensible logging decisions on your behalf when your node.js code is deployed on
Red Hat Mobile Application Platform. One of the interesting things about the
_bunyan_ module is that it prints logs in JSON format. Logging in this manner
is extremely valuable since it means you can stream logs
to multiple stores and persist them in a structured manner that can be queried
easily. Unfortunately this also makes reading these logs a little tough! Just
look at the screenshot and sample below - it's not what I would consider
user friendly.

```json
{"name":"/application.js","hostname":"ps-consult-na-rht-dev","pid":9736,"level":30,"msg":"App started at: Tue May 24 2016 20:22:03 GMT+0000 (UTC) on port: 8184","time":"2016-05-24T20:22:03.763Z","v":0}
```

![fh-bunyan logs](/res/img/posts/2016-05-25-pretty-print-logs-using-fhc/Screen%20Shot%202016-05-25%20at%2008.13.55.png)


# ~~Not so~~ Pretty Logs
To get these logs to print in a friendly format we have numerous options, but
I consider using the _fhc_ CLI to be best solution. Using the _fhc app logs_
command allows users to:

* Delete log files
* View log files
* Tail logs in realtime

Tailing logs is what we're after for now, so try the following:

```bash
# You might need sudo to global install - using nvm can get around this
npm i -g fh-fhc@latest-2

# Target your RHMAP instance
fhc target your-domain.feedhenry.com

# Login with username and password - you will be prompted for them
fhc login

# Tail logs from the set $CLOUD_APP_ID
fhc app logs tail --app=$CLOUD_APP_ID --env=$CLOUD_APP_ENV
```

This will print your logs to the terminal, but they're still not formatted in a
friendly manner since _fhc_ simply streams them "as is".

![fhc fh-bunyan logs](/res/img/posts/2016-05-25-pretty-print-logs-using-fhc/Screen%20Shot%202016-05-25%20at%2008.28.55.png)

The solution? Use the bunyan CLI to pretty-print them! Run the following
command and marvel at the beauty of your logs:

```bash
# You might need sudo to global install - using nvm can get around this
npm i -g bunyan
fhc app logs tail --app=$CLOUD_APP_ID --env=$CLOUD_APP_ENV | bunyan
```

![fhc fh-bunyan pretty logs](/res/img/posts/2016-05-25-pretty-print-logs-using-fhc/Screen%20Shot%202016-05-25%20at%2008.29.05.png)

This solution becomes even more powerful if you need to find specific logs
since you can pipe to _grep_ and perform a realtime filter like so.

```bash
# View all delete calls
fhc app logs tail --app=$CLOUD_APP_ID --env=$CLOUD_APP_ENV | bunyan | grep -i "delete"
```

![fhc fh-bunyan pretty logs](/res/img/posts/2016-05-25-pretty-print-logs-using-fhc/Screen%20Shot%202016-05-25%20at%2008.34.10.png)
