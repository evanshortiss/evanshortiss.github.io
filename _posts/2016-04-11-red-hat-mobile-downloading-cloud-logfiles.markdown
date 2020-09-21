---
published: true
title: Red Hat Mobile - Downloading Cloud Application Logs
layout: post
categories: development rhmap
redirect_from:
  - /development/rhmap/2016/04/11/red-hat-mobile-downloading-cloud-logfiles.html
---
Recently I found that I needed to fetch all the logs for a Node.js application running on the Red Hat Mobile Application platform. Fetching logs is easy via our Studio UI, but fetching all files automatically requires some extra effort. Thankfully we have fhc, a command line interface that can be used to interact with your instance of Red Hat Mobile.

You can install fhc by running:

```bash
npm install fh-fhc@latest-2
```

After fhc is installed, use the following to target your domain and login:

```bash
fhc target $YOUR_DOMAIN_URL
fhc login
```

Following this you might want to verify the name of the environment you're downloading logs from; it won't always match the UI! The command below will allow you to find the ID for the cloud environment you'd like to download logs from:

```bash
fhc admin environments list
```

Once you're setup you can use the following to download all logs for a given cloud application or mBaaS Service.

```bash
for log in `fhc app logs list --app=$YOUR_APP_ID --env=$ENV --json \
| grep -i "stdout" | sed 's/.*"\(.*\)"[^"]*$/\1/'`; \
do fhc app logs get --logname="$log" --app=$YOUR_APP_ID --env=$ENV --verbose; \
done > output.txt
```

This will download each stdout log file in series and write them to a file named _output.txt_. You can change the pattern passed to _grep_ to get stderr logs, or to get files for a specific date.
