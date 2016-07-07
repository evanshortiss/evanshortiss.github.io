---
published: false
title: Multiple Environments and Unified Push Server
layout: post
categories: development rhmap
---

Recently on a customer engagement I need to send push notifications to users of
a mobile application. In the past I have used Amazon SNS, but this gave me an
opportunity to use the AeroGear Unified Push Server. This blogpost will cover
how you can effectively utilise the Unified Push Server with your Red Hat
Mobile instance to deal with multiple deployment environments.

## AeroGear Unified Push Server
The AeroGear Unified Push Server is an open source push notification service
that can be deployed on-premise, on OpenShift, and is also included with Red
Hat Mobile Application Platform. It supports sending notifications to Android,
iOS, and Windows Phone.
