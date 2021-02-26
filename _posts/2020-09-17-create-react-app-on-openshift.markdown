---
published: true
title: Create React App on OpenShift
layout: post
categories: development openshift react
redirect_from:
  - /development/openshift/react/2020/09/17/create-react-app-on-openshift.html
---

[Create React App](https://create-react-app.dev/) is described as "the best way
to start building a new single-page application in React" on 
[reactjs.org](https://reactjs.org/docs/create-a-new-react-app.html). It's
no surprise so many developers start building a modern web application using
this toolchain.

This post outlines a straightforward method to deploy an NGINX container that
serves the contents of a [Create React App](https://create-react-app.dev/)
application, or any modern web application, on an OpenShift cluster.

A custom [source-to-image (s2i)](https://github.com/openshift/source-to-image#source-to-image-s2i)
builder that I maintain at [github.com/evanshortiss/s2i-nodejs-nginx](https://github.com/evanshortiss/s2i-nodejs-nginx)
is part of the magic that simplifies this process. This builder image was
originally authored by the folks at [jshmrtn](https://github.com/jshmrtn/s2i-nodejs-nginx)!

Using this method allows you to deploy code that's stored in a Git repository,
or to upload your local codebase directly to a [Build](https://docs.openshift.com/container-platform/3.9/dev_guide/builds/index.html)
running on OpenShift. Both approaches are covered below.

## Requirements

* An OpenShift 4.x cluster
* OpenShift CLI v4.x
* Node.js v12 or newer
* GitHub Account (optional)

## Git Method

_Note: I'm going to assume you're starting from scratch, but if you have an existing
repository with a React application that was generated via Create React App or
that uses `react-scripts` you can skip to the bash script below._

To use this approach head over to [github.com/new](https://github.com/new)
and do the following:

* Enter a **Repository name**
* Skip the **Initialize this repository with** section
* Click the **Create repository** button.

Once the repository is created, execute the commands below to get your
web application hosted by NGINX on OpenShift.

```bash
# The builder image is used by OpenShift to build the application
export BUILDER_IMAGE="quay.io/evanshortiss/s2i-nodejs-nginx"
export APPLICATION_NAME="cra-application"

# REPLACE THIS WITH YOUR REPOSITORY URL
export GIT_REPO="https://github.com/yourusername/your-react-app-repo"

# Create a React application template using CRA, and push to
# the empty GitHub repository you created
npx create-react-app $APPLICATION_NAME
cd $APPLICATION_NAME
git remote add origin $GIT_REPO
git push origin master

# Create an OpenShift project
oc new-project webapp

# Create a new BuildConfig on OpenShift based on the Node.js & NGINX builder
# --to:         The name for the output container image
# --build-env   Environment variable indicating where the builder image should
#               look for the bundled web application. This is the result of
#               running "npm build" or "yarn build". For CRA this is a "build"
#               directory, but for a custom webpack config it might be "dist"
oc new-app $BUILDER_IMAGE~$GIT_REPO \
--name $APPLICATION_NAME \
--build-env BUILD_OUTPUT_DIR=build

# Expose the service via HTTP using an OpenShift Route
oc expose svc/$APPLICATION_NAME

# Check the build status
oc get builds

# Print the application URL (Route)
echo "http://$(oc get route/$APPLICATION_NAME -o jsonpath='{.spec.host}')"
```

To deploy future changes run oc `oc start-build $APPLICATION_NAME`. You could
also set up a [Build Trigger](https://docs.openshift.com/container-platform/4.5/builds/triggering-builds-build-hooks.html#builds-triggers_triggering-builds-build-hooks)
to automate builds using a Git hook.

## Git-less Method

If you'd rather develop locally and upload the application assets for builds,
then give this binary build solution a try.

```bash
export APPLICATION_NAME="cra-application"
export BUILDER_IMAGE="quay.io/evanshortiss/s2i-nodejs-nginx"

# Create a React application template using CRA
npx create-react-app $APPLICATION_NAME

# Create an OpenShift project
oc new-project webapp

# Create a new BuildConfig on OpenShift based on the Node.js & NGINX builder
# --binary:     Indicates we'll trigger builds by uploading assets directly
# --to:         The name for the output container image
# --build-env   Environment variable indicating where the builder image should
#               look for the bundled web application. This is the result of
#               running "npm build" or "yarn build". For CRA this is a "build"
#               directory, but for a custom webpack config it might be "dist"
oc new-build $BUILDER_IMAGE --binary \
--to $APPLICATION_NAME \
--build-env BUILD_OUTPUT_DIR=build

# Start a Build based on the BuildConfig. Pass the local CRA project folder
# as the build input
oc start-build $APPLICATION_NAME --from-dir=$APPLICATION_NAME

# Create a Deployment using the latest tag on the application ImageStream
oc new-app --image-stream $APPLICATION_NAME

# Expose the service via HTTP using an OpenShift Route
oc expose svc/$APPLICATION_NAME

# Check the build status
oc get builds

# Print the application URL (Route)
echo "http://$(oc get route/$APPLICATION_NAME -o jsonpath='{.spec.host}')"
```

To push new changes from your local machine to OpenShift and trigger a build
run the `oc start-build $APPLICATION_NAME --from-dir=$APPLICATION_NAME` command
again.

## Result

Once `oc get builds` shows that the build is complete, you'll be able to view
the application in your web browser. The URL is obtained via the `echo` command
above.

![Create React App in Firefox](/res/img/posts/2020-09-14-create-react-app-on-openshift/cra-on-crc.png)