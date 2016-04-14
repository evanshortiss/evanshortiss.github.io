---
published: true
title: Android Application - Exposure Calculator
layout: post
categories: mobile ios android angular angularjs ionic
---

After much time thinking it over I've finally went ahead and finished writing a mobile applicaiton for iOS and Android - [Exposure Calculator](https://play.google.com/store/apps/details?id=com.evanshortiss.exposurecalculator). It's thoroughly unoriginal but it's a nice start! I felt it was somewhat required of me as a Software Engineer, who largely works in the capacity of writing mobile applications, to personally publish an app.

The application is pretty straightforward in its functionality. Given base exposure settings a photographer is satisfied with it will then calculate an equivlent exposure with a particular variable from the [exposure triangle](https://www.google.ie/#q=exposure+triangle) modified by the photographer.


## The Technologies
The application was written using [AngularJS](https://angularjs.org/) and the new [Ionic Framework](http://ionicframework.com/). It is contained in a native Android application wrapper built using [Cordova](https://cordova.apache.org/) via the Cordova CLI available on [npmjs.org](https://www.npmjs.org/package/cordova).

AngularJS is awesome and offers so much functionality for web application developers. Two-Way data binding is an out of the box feature of Angular, and is an instant win for any developer. It saves time working with DOM manipulation and allows a developer to simply focus on their application logic. Angular also pretty much forces the use of structured design patterns via it's modules, services, factories, controllers etc. No more spaghetti code!

Ionic extends on AngularJS and adds sugar for developing applications that are intended for use on mobile devices as Cordova applications. Besides that it also provides excellent out of the box directives. Directives are an AngularJS concept used to enable developers to extend HTML and create automatic hooks to run when certain HTML classes, attributes or tags are detected in the DOM. Ionic leverages on this ability to create excellent out of the box UI elements and styles.

## Photographic Calculations
Some may consider the technical implementation of exposure calculations to be of interest. As someone with a number of years photographic experience (I've been a keen photography hobbyist since the tender age of sixteen) I'm fortunate enough to understand photographic exposure.

For an application of this type it's not a requirement to continually calculate the resultant equivalent exposure based on the variables of shutter speed, aperture and ISO. Instead it is possible to generate photographic exposure tables beforehand and only do actual calculations for scenarios that go beyond the common exposure ranges. This approach is useful as it requires simple array index lookups based on offsets and therefore doesn't require exposure calculations. It's also far easier to take this approach from a user interface perspective as using calculated numbers would introduce the need to round figures and this may not always correlate with what is shown on a camera and may introduce confusion.

In my application, when a user requires a shutter speed greater than 30 seconds (represented as 30") the application will perform the neccessary photographic calculations to determine the required shutter speed as anything greater than 30 seconds is not in the pre-generated tables. I used the code below to calculate the shutter speed. The *requiredIndex* variable represents how many stops difference there is between the base shutter speed and the required shutter speed accounting for full stops, half stops or third stops.

```javascript
function calcShutterSpeed(requiredIndex) {
  // Get value of 1/X as a decimal i.e 1/3 -> 0.33333
  var stops = (1 / parseInt($scope.increments.split('/')[1]));
  // This is the scaling for the shutter speed
  var factor = Math.pow(2, stops);
  // X seconds
  var baseShutterSpeed = $scope.baseShutterSpeed;
  // Difference in speeds
  var offset = Math.abs(SHUTTER_SPEEDS[increments()].indexOf(baseShutterSpeed) - requiredIndex);
  // Get shutter speed as fraction of a second
  var res = Math.round(Math.pow(factor, offset) * shutterNotationToDecimal(baseShutterSpeed));

  if(res <= 60) {
    return res.toString() + '"';
  } else {
    return seondsToFormattedTime(res);
  }
}
```


## Wait. It's hybrid! What about iOS?
You might be wondering why I've only published to the [Play Store](https://play.google.com/store) for Android if it's a hybrid application. My main reasoning for not yet publishing on iOS is one of design. I started out primarily targeting iOS but as a daily Android user I switched bases.

## Android Design
Personally my design skills are pretty weak, but then again I never trained to be a graphic designer so go figure! Despite my lack of design knowledge, I do have an appreciation for pleasing design and consider it to be an essential part of all applications. If an application isn't pleasant to look at it generally isn't pleasant to use. Either way, I'm not planning to go into a UI/UX rant here, but it is important and, in my opinion, many Android applications don't cut the mustard when it comes to design.

For the Android version of this application I checked out the [design section](https://developer.android.com/design/index.html) of the Android Developers site which I first checked out over two years ago while working as an intern. It has some nice simple colour schemes, links to the gorgeous Roboto font TTF files and default Android iconography, and an overview of Android's default themes amongst other things. If you haven't already checked it out then do!

To override the default iOS friendly styles of Ionic I simply injected a small CSS file like so into my index.html based on the device User-Agent. Naturally the Cordova device plugin is also an acceptable solution for device detection. Seemples!

```javascript
var ua = navigator.userAgent;

if (ua.match(/Android/)) {
  var tag = document.createElement('link');
  tag.rel = 'stylesheet';
  tag.href = 'css/android.css';

  document.head.appendChild(tag);
}
```

Exposure Calculator doesn't make too much use of colour as I didn't think it appropriate. The only real use of colour is to inform users that entered settings are out of the standard exposure ranges generally possible by changing the Equivalent Exposure field red and a blue accent on a slider. The screenshot below shows the application running on my Galaxy S3 with an exposure that requires a shutter speed greater than 1/8000th of a second.

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2014/May/Screenshot_2014_05_11_16_07_28.png)

## What's next?
iOS! Now that the Android version of this application is in a condition I'm satisfied with I'm ready to tweak it for iOS and push it out ASAP.
