---
published: true
title: SNS Push Notifications using Node.js
layout: post
categories: development mobile
redirect_from:
  - /development/mobile/2014/02/22/sns-push-notifications-using-nodejs.html
---
# Introduction
Recently I had to use Amazon's SNS service to send Push Notifications to a mobile application being developed for iOS and Android.

[SNS](http://aws.amazon.com/sns/) according to Amazon:

*SNS makes it simple and cost-effective to push to mobile devices such as iPhone, iPad, Android, Kindle Fire, and Internet connected smart devices, as well as pushing to other distributed services.*

Android push notifications are sent from your server to devices via the Google Cloud Messaging for Android (GCM). Similarly iOS notifications are sent to devices from your server side application via Apple Push Notification Service (APNS). SNS acts like a proxy between your Node.js application and the GCM / APNS services that allows you to use a generic API for sending messages to all device types.

This post will focus on how to get up and running with Amazon SNS Push Notifications for iOS and Android via a Node.js application. It will not cover the Android and iOS code required just yet, but that may be added later or be done in a separate blog post. I have not tested this tutorial on a Windows machine but it should work fine with the only exception being around the manner in which environment variables are set. If on Windows you could use [Cygwin](http://www.cygwin.com/) as a work around for accessing a Unix like terminal.

# Basic Terms
There are a few terms you'll need to be familiar with before getting started with SNS. If you don't know these then you'll feel a bit like poor Chris O'Dowd here when you read some of the later sections.

![](http://www.reactiongifs.com/r/wtfih.gif)


#### APNS (Apple Push Notification Service)
Apple's service for sending push notifications.

#### GCM (Google Cloud Messaging)
Google's service for sending push notifications.

#### PlatformApplication(ARN)
A PlatformApplication is a resource created in SNS that is used to send push notifications for an application. To create an PlatformApplication for an iOS application you will need APNS credentials, these are your private key and certificate from the [Developer Centre](https://developer.apple.com/membercenter). Creating a Platform application for an Android application requires a Server Key from the [Google Cloud Console](https://code.google.com/apis/console/)

Each Platform Application has a PlatformApplicationArn with the format: *arn:aws:sns:eu-west-1:1234567890:app/Service/AppName*. The ARN is used to refer to the application in API calls.

#### Endpoint(ARN)
An Endpoint refers to a created endpoint. Each created Enpoint has an EndpointArn that can be used to refer to it in API calls. These ARNs have the format *arn:aws:sns:eu-west-1:1234567890:endpoint/APNS/MyApp/5e2068b1-7d74-39ec-946e-a5fcca3ad737*.

# Setting Up an Amazon User
Before you can start making calls to the SNS API you'll need to setup a user with the correct permissions to get access credentials.

During this process you'll be prompted to download a .CSV file containing your access credentials. That's gonna come in handy later so hold on to it.

1. Login to your AWS account and navigate to the IAM section.
2. Go to the Users tab and click Create New Users.
3. Enter a user name and make sure "Generate an access key for each User" is checked.
4. Take note of the generated Secret Key and Access Key as you'll need these!
5. Finally select your user from the list go to the Permissions tab for that user and click Attach User Policy.
6. Enter the policy posted below.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccessToSNS",
      "Effect": "Allow",
      "Action": [
        "sns:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
*Disclaimer: I'm not an AWS ninja so this configuration may not be optimal!*

# Creating a GCM Platform Application (Android)
To use Google's Cloud Messaging for Android service you need to enable it from the [Google Cloud Console](https://cloud.google.com/console) and then get your Server Key so the SNS server can send messages via GCM.

Login to the Cloud Console and activate Google Cloud Messaging for Android from the APIs & Auth -> APIs tab.

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2013/Dec/Screen_Shot_2013_12_18_at_01_14_58.png)

Once this is done go to the Credentials section and scroll down to the Public API access section and copy your Server Key if one is present. If no key is present create one and copy it.

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2013/Dec/Screen_Shot_2013_12_18_at_01_14_58.png)

Login to AWS and go to the SNS section. Here click the Create and Add button. In the popup enter a sensible name. A name that follows an application bundle identifier, com.myname.appname, for example is probably a good idea as you can easily link which PlatformApplication is linked to an on device application. Select GCM as the Push Platform and finally paste in the Server Key form earlier.

![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2013/Dec/Screen_Shot_2013_12_18_at_01_14_58.png)

Now you're ready to send Push Notifications to your Android application via SNS!

# Creating an APNS Platform Application
Creating an SNS APNS application is a little more difficult than creating an SNS GCM PlatformApplication as it requires the generation of certificates and some configuration in the iOS Member Center.

First you need to create a Certificate in the iOS Developer Centre. The kind of Certificate you'll be using for now is an *Apple Push Notification service SSL (Sandbox)* as I assume you'll be testing with a development application. Select the application ID that you'll be using from your list of identifiers and click the Edit button. On the next screen scroll down to the Push Notifications section and create a *Development SSL Certificate* as shown in the screenshot.

![](/content/images/2014/Feb/Screen_Shot_2014_02_22_at_12_33_13.png)

Once you have generated the required certificate you need to convert it to a format that is usable by SNS. The steps required to accomplish this are outlined by Amazon themselves [here](http://docs.aws.amazon.com/sns/latest/dg/mobile-push-apns.html) so follow those steps until you reach the *To obtain a device token from APNS for your app* section.

Once you have the certs in correct format you can setup the PlatformApplication on SNS. Make sure to select APNS_SANDBOX from the creation dialog box and then paste in the contents of the files you converted earlier into the appropriate text areas.

![](/content/images/2014/Feb/Screen_Shot_2014_02_22_at_15_06_43.png)

# The Node.js Code
Now for your Push Notification server. I'm assuming you have node.js and npm installed on your machine. The basic process I've used is as follows:

1. Get device token (done on device)
2. Send device token to your Push Server.
3. Register this device with SNS.
4. Pester your users!?
5. Profit!

To do this we'll use an [expressjs](http://expressjs.com/) server and the wrapper I've written for Amazons AWS.SNS module, [sns-mobile](https://www.npmjs.org/package/sns-mobile).

## The Server
Create a new directory in your workspaces folder. Now navigate to that folder and type the following:

```language-bash
npm init
```

Follow the prompts and don't worry too much about what you enter as it isn't really relevant for the purpose of this tutorial but it will provide some scaffolding that we need. Once finished the setup process type the following commands to install the required dependencies.

```language-bash
npm install sns-mobile@0.0.0 --save
npm install express@3.2.0 --save
```

## Preparation
Remember the CSV file / credentials you got from Amazon earlier? Time to break that out again. Open the file and also open a terminal.

In the terminal type one of the following. If you prefer vi/vim use those instead of these commands.

```language-bash
open ~/.bash_profile
```
or ...
```language-bash
nano ~/.bash_profile
```

This should open your bash_profile for editing. We're doing this to setup some environment variables that will be used by our server. The reason we'll use these is so that our code can reference these variables. The keys access key and key ID Amazon provided in that file should be kept secret so you don't want them as plaintext in code unless you're 100% sure the code won't be pushed to a public repository.

Add the following lines to your bash_profile but replace the values below with the ones applicable to yourself:

```language-bash
# Amazon SNS variables
export SNS_KEY_ID="{Access Key Id}"
export SNS_KEY_ID="{Secret Access Key}"
export ANDROID_ARN="{The PlatformApplicationArn}"
```

Once completed save the file. If you used nano, save the file by pressing CTRL and X together then pressing enter. If you get a permission error do it as sudo.

To get the updated bash_profile to take effect you'll need to close all active terminal sessions and restart them or issue the following command:

```language-bash
source ~/.bash_profile
```

## Device Registration
As this is a tutorial we'll use a very lightweight example. Place the following code in a file called index.js in your root directory.

```language-javascript
// Require our dependencies
// Express is resolved from node_modules
// Push will be a local file we create
var express = require('express'),
  push = require('./push.js');

var app = express();

// Body parser extracts POST data from incoming
// requests for our use
app.use(express.bodyParser());

// Path used to register for push notifications
// push.register is the handler function
app.post('/register', push.register);

app.listen(3000, function(error) {
  if(error) {
    // Something unexpected happened
    throw err;
  }

  console.log('Listening on 127.0.0.1:3000');
});
```

So. What's happening here? We've setup a basic http server that has a single route */register* defined. Now we need to implement the handler for that route in a file named *push.js*.

```language-javascript
// We need to use the sns-mobile module
var SNS = require('sns-mobile');

// Some environment variables configured
// You don't want your keys in the codebase
// in plaintext
var SNS_KEY_ID = process.env['SNS_KEY_ID'],
  SNS_ACCESS_KEY = process.env['SNS_ACCESS_KEY'];

// This is an application we created via the
// SNS web interface in the section
// Creating a GCM Platform Application (Android)
var ANDROID_ARN = process.env['SNS_ANDROID_ARN'];

// Object to represent the PlatformApplication
// we're interacting with
var myApp = new SNS({
  platform: 'android',
  // If using iOS change uncomment the line below
  // and comment out the 'android' one above
  // platform: 'ios',
  region: 'eu-west-1',
  apiVersion: '2010-03-31',
  accessKeyId: SNS_ACCESS_KEY,
  secretAccessKey: SNS_KEY_ID,
  platformApplicationArn: ANDROID_ARN
});

// Handle user added events
myApp.on('userAdded', function(endpointArn, deviceId) {
  console.log('\nSuccessfully added device with deviceId: ' + deviceId + '.\nEndpointArn for user is: ' + endpointArn);

  // Maybe do some other stuff...
});

// Publically exposed function
// Recieves request and response objects
// This is used by index.js
exports.register = function(req, res) {
  var deviceId = req.body['deviceId'];

  console.log('\nRegistering user with deviceId: ' + deviceId);

  // Add the user to SNS
  myApp.addUser(deviceId, null, function(err, endpointArn) {
    // SNS returned an error
    if(err) {
      console.log(err);
      return res.status(500).json({
        status: 'not ok'
      });
    }

  // Tell the user everything's ok
    res.status(200).json({
      status: 'ok'
    });
  });
};
```

That's it! Pretty straightforward right? So, now you probably want to test it's working, well that's easy via a curl request. Run the following commands.

Start the server (run this in the project root directory)

```language-bash
node index.js
```

And now in a different terminal send a fake device ID. Android has no strict checks on the ID so we can get away with this.

```language-bash
curl http://127.0.0.1:3000/register -d 'deviceId=abc123345andmorerandomstuff'
```

You should get the following response in your terminal if all goes well.

```language-bash
{
  "status": "ok"
}
```

Now navigate to the SNS interface in your browser and view your Android PlatformApplication. You should see the fake ID(s) you added under the *Token* column. Great success!

![](http://www.reactiongifs.com/wp-content/uploads/2013/11/stoked.gif)

Admittedly you'll also want to save the returned *endpointArn* from the *addUser* callback to a database as you'll need it to send messages to the device at a later point in time.

## Sending Messages to Devices
So you want to send messages to devices based on certain parameters? Well that's pretty easy too. The code sample below demonstrates how this can be accomplished using the *sendMessage(endpointArn, message, callback)* function.

```language-javascript
// Assume the myApp object from previous
// samples is still available

// Sample endpointArn
var endpointArn = 'arn:aws:sns:eu-west-1:1234567890:endpoint/GCM/AndroidTest/5d3954e1-7d68-365a-80c2-95ae98ae4336';

// Message to send
var message = 'Thanks for using Push Notifications';

myApp.sendMessage(endpointArn, message, function(err, messageId) {
  if(err) {
      console.log('An error occured sending message to device %s', endpointArn);
        console.log(err);
    } else {
      console.log('Successfully sent a message to device %s. MessageID was %s', endPointArn, messageId);
    }
});
```

# Well Done!
If you've made it this far, well done, we're finished! The code for this tutorial is available [here](https://github.com/evanshortiss/sns-mobile-example) if you'd like to clone it and take a look. It's also worth noting we've only covered a small subset of the functionality available from SNS. You can read more about the module [here](https://github.com/evanshortiss/sns-mobile).

Thanks to Máté Rácz, Keyang Xiang, Michael Hearne, and Naomi Moran for their work on the client side code for this project.
