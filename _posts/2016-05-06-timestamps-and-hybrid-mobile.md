---
published: false
title: Timezones and Mobile Applications
layout: post
categories: development
---

TLDR; Always use UTC or Epoch timestamps. Ensure servers are set to UTC time.

With software becoming more mobile focused it can roam freely around our
[global village](TODO) on high powered mobile devices. This makes our users
_really_ happy, but is not without development challenges. This post will focus
on one of those challenges in particular - managing timestamps across
applications running in multiple timezones.

## Traditional Web Applications
Managing timestamps sounds like a trivial issue. A user performs some
action that calls a HTTP endpoint on your application server, the server
performs some tasks and in the process notes the time the operation was
performed at. Simple, right? This problem is trivial in the example we've
described; the server is responsible for recording the times that actions
occurred so you always have consistent timestamps regardless of the timezone
your user is in. If you're rendering a page for the user you can store their
timezone in a cookie and pass that to the server to correctly render the time
in a HTML template that will be returned to the user.

## Mobile Applications
When working with mobile a paradaigm shift occurs since you typically won't
perform server-side rendering, and the application has to work offline. Think
about some of your favourite mobile applications and how they work - offline
productivity is a must! Most of them achieve this by recording data to device
storage and then uploading it to a RESTful API at some later point in time.

Recording activity offline means we need to capture timestamps on the device
and send those to our backend. It's critical that this data is recorded
correctly, particularly in enterprise applications. Sending these local device
timestamps to the server could lead to problems if you don't process them
correctly, in particular you'll want to be careful with regard to timezone data.

## Real World Example
Let's take Google Photos as an example, and assume you have auto backup
turned on:

1. Head out to your favourite hiking trail miles from civilisation and any cell
towers.
2. Hike a little.
3. Take some pretty photos of nature.
4. Hike a little more.
5. Return home and take a nap. You deserve it.

When you wake up from your well deserved nap you might have a notification from
Google Photos saying "We've backed up your photos!". Let's take a look at my
Google Photos for an example of what it shows for an upload:

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2016/May/Screen%20Shot%202016-05-06%20at%2010.05.36.png)

In the above picture the main thing we care about for the purposes of this
article is the date and time on the details tab. You can see that it
states April 23rd at 4:44PM. This is not the time I got home and my Google
Photos began to Sync over WiFi, it's the time the photo was taken.
This means my device captured the time, sent it up to Google with the
upload, and Google is using that timestamp instead of the time it actually
received the image when it renders the page. This sounds simple enough, but
what about the timezone differences between the Google servers and my mobile
device? What if the timestamp my device upload is EST format and the server
stores it in some other format or performs some operations on the timestamp?

#### RESTful APIs and Timezones
Let's assume Google has a JSON API
