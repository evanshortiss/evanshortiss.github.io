---
published: true
title: Excluding files from local Source-to-Image (s2i CLI) Builds
layout: post
categories: openshift nodejs
---

TLDR; Use the flag `--exclude "(^|/)\.git|node_modules|some_other_folder(/|$)"`

[Source-to-Image (s2i)](https://docs.openshift.com/container-platform/4.7/openshift_images/using_images/using-s21-images.html)
provides a convenient solution for developers looking to easily
create OpenShift-ready container images from their code.

A simple way to build and deploy an application on OpenShift using s2i is via
the `oc new-app` command.

```bash
export BUILDER=registry.access.redhat.com/ubi8/nodejs-14
export REPO=https://github.com/sclorg/nodejs-ex.git

oc new-app $BUILDER~$REPO
```

It's also possible to use s2i to build code in your development environment
using the [s2i CLI](https://github.com/openshift/source-to-image/releases).

When building a Node.js application locally you might have `.env`,
*node_modules/*, and other files that you'd rather not pass to the s2i build.
These might be files that are typically included in your *.gitignore* or a
*.dockerignore*.

I recently learned that s2i supports an `--exclude` flag to support this
behaviour. The help output for this flag is surprisingly, well, helpful:

```
Regular expression for selecting files from the source tree to exclude
from the build, where the default excludes the '.git' directory
(see https://golang.org/pkg/regexp for syntax, but note that ""
will be interpreted as allow all files and exclude no files)
(default "(^|/)\.git(/|$)")
```

Now, I don't know about you, but I immediately assumed this would be pain in
the ass to configure. 99% of the time I write RegEx I do so in JavaScript, and
I use the Node.js REPL to experiment and get it right.

![I'm tired of pretending it's not meme for Regex](https://i.imgflip.com/5exkyb.jpg)

Much to my surprise though, I got lucky with this one on the first try.
Seriously. I know the RegEx ninjas out there think I'm worthless but
sometimes I get lucky! 🤷‍♂️

Here's how you can use `--exclude` to exclude *node_modules* and two other
directories:

```bash
# Output image name
export IMAGE_NAME=quay.io/evanshortiss/rhosak-nodejs-sbo-example

# Base image used to create the build
export BUILDER_IMAGE=registry.access.redhat.com/ubi8/nodejs-14

# Build the local code into a container image, but do not pass
# the .git, node_modules, or .bindingroot directory to the build
s2i build -c . --exclude "(^|/)\.git|.bindingroot|node_modules(/|$)" \
$BUILDER_IMAGE $IMAGE_NAME
```

