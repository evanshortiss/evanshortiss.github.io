---
published: true
title: Monitor GitHub Files using Lambda or Kubernetes & SendGrid
layout: post
published: false
---

Recently, I needed to be notified when changes were made to a specific file in
a repository GitHub. Using Google I quickly found [github-file-watcher.com](https://app.github-file-watcher.com/).
Obviously that isn't the end of the blogpost though, since I ended up DIY'ing
this task. I needed to add multiple people to the notification list using their
corporate email addresses, and I only required one notification per day instead
of an email any time the file changed.

Being honest, I also wanted to have a little fun and make it a stateless
function that I could run anywhere, anytime. However, it's worth mentioning
that [github-file-watcher.com](https://app.github-file-watcher.com) is
excellent, and probably exactly what you need!

## Requirements List

The requirements the generic file watcher I developed were:
 
1. Notify multiple people of commits made to the file via email.
1. Include a commit list in the email.
1. Check for changes on a fixed interval.
1. Relatively simple to deploy and manage.
1. Cheap, or better yet, free to run.

In the next few paragraphs I'll break down how each requirement has been
fulfilled.


## Fetching Commit Data and Detecting Changes

Naturally, I was going to use the GitHub API to find a commit log for the file
being watched. I knew that this had to be possible via their API, and found that
the following endpoint provides recent commit data for a file -
`https://api.github.com/repos/$USER/$REPO/commits?path=$FILEPATH`

Here's a sample URL that fetches commits for the Node.js project README:

```bash
curl "https://api.github.com/repos/nodejs/node/commits?path=README.md"
```

This returns an array of objects that contain a bunch of data. Here's a slimmed
down example that includes only the fields that I needed:

```json
[
  {
    "sha": "9a46cfc33796a5effade4a0c6de95807f1fca7ad",
    "node_id": "MDY6Q29tbWl0MjcxOTM3Nzk6OWE0NmNmYzMzNzk2YTVlZmZhZGU0YTBjNmRlOTU4MDdmMWZjYTdhZA==",
    "commit": {
      "author": {
        "name": "Nick Schonning",
        "email": "nschonni@gmail.com",
        "date": "2019-08-29T13:04:34Z"
      },
      "committer": {
        "name": "Rich Trott",
        "email": "rtrott@gmail.com",
        "date": "2019-08-31T22:27:58Z"
      }
    }
  }
]
```

Since I only need to check for changes every 24 hours, i.e my function only
runs once every 24 hours, I can grab the `commit.committer.date` and check if
the commit timestamp is less than 24 hours old. If the timestamp is within that
range, I keep the commit, otherwise it's filtered out. This logic is performed
for each commit returned from the GitHub API.

The code below is the actual implementation of that filter. `INVOCATION_TIME`
is the current time, and `TIME_PERIOD` is `24 * 60 * 1000`, i.e the number of
milliseconds in 24 hours.

It's worth noting that `TIME_PERIOD` is configurable using an environment
variable, since if you'd like to check the target file for changes more
frequently than once every 24 hours you need to make sure the `TIME_PERIOD`
aligns with the frequency at which you run this function. For example, if you
plan to run the function every 2 hours the `TIME_PERIOD` variable should be set
to `2 hours` which will be translated to `2 * 60 * 1000` milliseconds.

```js
/**
 * Each commit Object is passed to this via Array.prototype.filter
 * e.g const getCommits().then(commits => commits.filter(commitTimeFilter))
 */
function commitTimeFilter (commit) {
  const commitDate = new Date(commit.commit.author.date)
  const timeDifference = INVOCATION_TIME.getTime() - commitDate.getTime()

  const isNewCommit = timeDifference < TIME_PERIOD

  return isNewCommit
}

const newCommits = githubApiResponse.filter(commitTimeFilter)

if (newCommits.length !== 0) {
  sendEmail(newCommits)
}
```

## Notification Emails

Sending an email programmatically in Node.js, and basically any language is a topic
that you'll find no shortage of content for so I'm not going to cover it in much
detail, but the following are popular options for Node.js if you have an SMTP
server available:

* [nodemailer](https://www.npmjs.com/package/nodemailer)
* [emailjs](https://github.com/eleith/emailjs)

If you don't have an SMTP server lying around [SendGrid](https://sendgrid.com)
and [Mailgun](https://www.mailgun.com/) are great options, and they have free
tiers available. Having past experience using SendGrid meant I chose their API
for this task. 

Very little code is required when using the SendGrid Node.js module:

```js
const text = generateEmailText(newCommits)

mailer.send({
  to: getEmailRecipients(),
  from: process.env.SENDGRID_SENDER,
  subject: `GitHub File Watcher: ${subject}`,
  text
})
  .then(() => log('Email sent successfully to: ', getEmailRecipients()))
  .catch((e) => {
    log('Error sending email')
    log(e)
  })
```

To avoid hard-coding the SendGrid details and email recipients I used some
environment variables to inject those paramaters:

* SENDGRID_API_KEY - The SendGrid API key
* SENDGRID_SENDER - The email address recipients will see in the "from" field
* SENDGRID_RECIPIENT - The recipient who should receive notifications
* SENDGRID_RECIPIENTS - A list of recipients. Overrides `SENDGRID_RECIPIENT` if set

## Deployment on AWS

Running this function at approximately the same time everyday using my own
laptop wasn't a viable option since I might be on PTO, traveling, eating lunch,
etc.

A long running server also isn't necessary for this task. A
Function-as-a-Service (FaaS) model such as
[AWS Lambda](https://aws.amazon.com/lambda/) was perfect for my requirements if
I could find a way to trigger it at an interval.

When I determined that it was
possible to invoke a Lambda using [AWS CloudWatch Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html)
it became a no-brainer to use AWS. Also, running a trivial function like this
once per day is effectively free on AWS so that covers requirements 3, 4, and
5.

Initially I set out using the AWS CLI to manage this, but I realised that using
the [Serverless Framework](https://serverless.com) made it significantly
easier since I could configure the deployment in a YAML file and let the
Serverless Framework handle the rest. The YAML specifies the function to
invoke, and the interval using [crontab](https://en.wikipedia.org/wiki/Cron)
syntax:

```yaml
service: serverless-github-file-watch

frameworkVersion: "=1.43.0"

provider:
  name: aws
  runtime: nodejs10.x
  profile: serverless
  stage: dev
  region: us-west-1

functions:
  checkForNewCommits:
    handler: handler.checkForNewCommits
    events:
      - http:
          path: /
          method: get
      # invoke the function at midnight each night
      - schedule: cron(0 0 * * ? *)
```

Deployment was then taken care of using the `serverless deploy` command. I
opened the AWS Lambda dashboard after the deployment was complete, and set the
required environment variables. The screenshot below contains an example of the
environment variables:

![](/res/img/posts/2019-09-16-github-file-watcher/aws-lambda.png)

And that's it! The function will execute each night at midnight (UTC), and if
any commits landed within the past 24 hours an email notification is sent to
the recipients defined in the `SENDGRID_RECIPIENT` or `SENDGRID_RECIPIENTS`
environment variable.

## Deployment on Kubernetes

Running a CronJob on Kubernetes is incredibly straightforward since the
platform has native support for them since version 1.8. Like everything in
Kubernetes this involves creating a YAML definition. I've included one in
the [repository](https://github.com/evanshortiss/stateless-github-file-watch/blob/master/cronjob.yml) for this blogpost - it looks like this:

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: stateless-github-file-watch
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: stateless-github-file-watch
            image: quay.io/evanshortiss/stateless-github-file-watch:latest
            env:
            - name: TIME_PERIOD
              value: "24 hours"
            - name: FILE_URL
              value: "https://github.com/nodejs/node/blob/master/README.md"
            - name: SENDGRID_RECIPIENT
              value: "someone@acme.com"
            - name: SENDGRID_API_KEY
              value: "api-key-goes-here"
            - name: SENDGRID_SENDER
              value: "github-file-watcher@acme.com"
          restartPolicy: OnFailure
```

This tells Kubernetes to create a CronJob that runs at midnight each night.
When it executes it will start a container specified by the `containers.image`
key. This container executes the code discussed in the previous sections of
this post to check the Node.js *README.md* for changes made in the past 24
hours.

Creating this in a namespace on your cluster is easy - just update the
variables and run `kubectl apply -f cronjob.yml -n $TARGET_NS`.