---
published: true
title: Crosswalk - The Answer to (most) Android WebView Woes
layout: post
categories: development mobile
redirect_from:
  - /development/mobile/2015/06/12/crosswalk-android-webview.html
---

Day to day at FeedHenry (by Red Hat) we bundle the vast majority of our applications using Cordova. This has some huge benefits for us, the obvious one being a single codebase for running an application on multiple platforms. Like any convenience however it has some drawbacks.

### Cordova in Practice
For most projects I've found that iOS behaves and performs reasonably well. The WebView is well featured and mostly consistent when you're dealing with iOS. Unfortunately the same can't be said for Android. Working on web applications for Android since version 2.2 has allowed me to observe that the varying WebView implementations often create a real minefield for developers. Despite the improvements to Android as a whole since the release of 4.0 we've seen some really strange stuff at FeedHenry. For example, select fields that don't seem to know their position when using a mixture of absolute and relative positioning in a specific layout (my mind still melts just thinking about this one), to magic input focus boxes that scroll when a page is scrolled while the original input stays in place! Then of course, there's the lacking features, and fluctuating performance caused due to a poor WebView and varying hardware configurations.

### ~~Improving~~ Replacing the Android WebView
In the past I had discussed with a colleague the possibility of somehow placing a Chromium WebView in place of the standard one, but in the world of a Professional Services environment deadlines reign supreme so nothing ever materialised from this idea. Little did I know that some other clever folks had already started implementing such an idea.

Fast-forward a little over a year and I stumble across the Crosswalk project. The Cordova/Chromium/Developer gods have answered my prayers it seems! Crosswalk remove the default WebView used by Cordova applications running on Android and replace it with Chromium. On devices running Android versions prior to 4.4 this is an absolute fantastic development since it means the following you get a standardised WebView regardless of manufacturer and Android version and far better performance. Android 4.4 and above defaults to Chrome, so if you're only supporting newer Android then maybe you don't need Crosswalk.

### Adding Crosswalk to a Project
The Crosswalk Project has an excellent guide on how to update your existing Android 4.0+ project to use the new WebView [here](https://crosswalk-project.org/documentation/cordova_3/migrate_an_application.html). Thankfully it's very straightforward, but has one drawback, it doesn't cover how to create a single binary that will run on both ARM and x86 devices. I wanted to do just that and figured I'd share/record the steps here.

#### Creating a Dual Architecture .apk
Creating a universal bundle is extremely easy once you figure it out. Creating this build is great if you plan to test on an Android emulator where running in x86 mode is the only choice you have unless you plan to waste away years of your life waiting on ARM emulation.  I've outlined the steps below:

1. Pick a release from the [stable downloads page](https://download.01.org/crosswalk/releases/crosswalk/android/stable/), unless of course you're feeling particularly brave.
2. In the root of the selected release page/folder download the file named _crosswalk-VERSION.zip_.
3. You also need to click either the ARM or x86 folder in the same directory and download the file named _crosswalk-cordova-VERSION-ARCH.zip_ within. You don't need to do this for both architectures as you'll see shortly.
4. Delete the contents of _platforms/android/CordovaLib_ in your project.
5. Unzip the first downloaded file (_crosswalk-VERSION.zip_) and copy the files in its _framework/xwalk_core_library_ folder to the _platforms/android/CordovaLib_ in your own project.
6. Copy the VERSION file from the root of the unzipped folder to _platforms/android_
7. Unzip the second download and copy _framework/xwalk_core_library/src_ into _platforms/android/CordovaLib/src_. This will ensure you have the Cordova framework files that are omitted from the first .zip you downloaded.
8. Run the following from your project root replacing the _--path_ and _--target_ as necessary: _android update project --subprojects --path platforms/android/ --target "android-21"_. I had to use _android-21_ for Crosswalk 12.
9. Run _ant debug_ in _platforms/android/CordovaLib_

That's it. You can now run _cordova build android_ to produce a single .apk that will run on x86 and ARM. Bear in mind this will make the base .apk size around 40MB before you include your assets.

#### The Quick and Easy Way
There is another way to add Crosswalk to your application - using a Cordova plugin! You need to be running at least version 5.0.0 of the Cordova CLI. I have tested this using 5.1.1 and it worked flawlessly. If you're not interested in installing 5.0.0 globally for now, and assuming your project contains a _package.json_, install it locally in your project and symlink it like so:

```
npm i cordova@5.1.1 --save-dev
ln -s node_modules/.bin/cordova cordova
```

Once that's done you can do the following in your project root:

```
./cordova platform remove android
./cordova platform add android
./cordova plugin add cordova-plugin-crosswalk-webview
./cordova prepare android
./cordova build android
```

This produces two separate .apk files, but is quick and easy if you'd like the most pain free method of getting up and running.
