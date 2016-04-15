---
published: true
title: Creating Express Middleware for flickr
layout: post
categories: development javascript photography
---

_Note: My website is no longer running this since I've moved to Jekyll. Screenshots at the conclusion of this post show what can be achived by using this module._

TLDR; I created an Express middleware ([express-flickr-gallery](https://github.com/evanshortiss/express-flickr-gallery/)) that makes it easy to create a gallery in your Express app using your images from flickr.

I finally got the motivation to update my website over the past weekend. In doing so I had three primary goals:

1. Integrate my Ghost blog with the Express server already serving my website.
2. Redesign the UI to be cleaner and mobile friendly.
3. Incorporate a gallery component to showcase some of my photography.


### Integrating Ghost with Express
The first step was covered by Cian Clarke, a colleague of mine. I recalled that he posted a blog post previously about how he integrated Ghost with his own website. After a scan of his blog post and code it was clear how he achieved it so I shamelessly lifted his solution. Thanks Cian!

### Redesigning the UI
Redesigning the UI was tricky due to the fact my design skills aren't particularly strong. Using Bootstrap's default theme and some hacked together design I came up with a style that worked. I think...


### Deciding on a Gallery Implementation
The final, and arguably most arduous piece of work was to find a clean solution for exhibiting my photography. I could have hosted files myself and served them using Express, but this solution was too archaic for me. I could also have used the [node-gallery](https://github.com/cianclarke/node-gallery) module, which happens to be created by Cian, but having already used his Ghost solution I felt this might be too much of an ego boost for my fellow giner.

OK, all jokes aside, I actually thought [node-gallery](https://github.com/cianclarke/node-gallery) was a neat solution, but I wanted to reduce the management overhead as much as possible and upload my images to one place and have them hosted for me. I already had my photography hosted on flickr.com, and they expose a great API. The choice became very clear at this point; I'd need to somehow integrate my Express app with the flickr API. When I uploaded images to flickr they could instantly appear on my own website if it was calling the API without any extra administration work from my side. Perfect!

### Flickr Gallery Implementation
I knew the steps I'd need to take to make this solution possible.

1. Wrap the flickr API for Node.js
1. Fetch existing Sets (Albums) for a configured user.
2. Load images from said Sets into a template.

#### Gallery API Abstraction

[Mike Kamermans](https://github.com/Pomax) had already done a great job of creating a flickr API wrapper - this made writing my own gallery logic far easier, so thanks to Mike for the module.

The logic I needed to implement was pretty straightforward using Mike's API wrapper, but one component that I found missing was the ability to generate flickr CDN URLs from the returned image IDs without an extra query to their API. In a bid to reduce the requests to their API and also to allow me to dynamically create CDN URLs I created [flickr-urls](https://github.com/evanshortiss/flickr-url), a module that can generate the correct URL for an image at a specified size using the high-level data the flickr API returns for Sets.

With the URLs out of the way I finalised abstraction and had it expose three methods:

1. _init_ - Configure the API wrapper by providing the API Key, User ID and Secret.
2. _getAlbumList_ - Retrieve public Sets for the given user.
3. _getAlbum_ - Get the content (images) in a given Set.

The code is pretty concise and can be seen [here](https://github.com/evanshortiss/express-flickr-gallery/blob/master/lib/flickr.js). It also caches results for 5 minutes to improve page load times.

#### Middleware
Once the URL generation and gallery logic was out of the way I needed to decide on how users could drop this into their existing Express application. I wanted to avoid forcing users to configure a templating engine or be forced into a particular layout structure as much as possible. To achieve this I decided to have the user supply a _templates_ configuration option to the _middleware_ I created. Using this the user supplies the name of their template for the list of Sets (albums) and an Set (album) view into the middleware during initialisation. This allows the middleware to render them later with _res.render_ and inject an _expressFlickrHtml_ variable that the user can render wherever they like in their own template. The _expressFlickrHtml_ is generated internally using Jade templates.

Users can also supply a _renderer_ object to the middleware with a _classNames_ property. Doing so will allow custom classes be added for container, row and column element types for added layout flexibilty, and to support a responsive UI.

It might also be useful to expand the API in the future to allow it to return the HTML programmatically without defaulting to using the _res.render_ function on behalf of users, but for now it's working well on my own site at [evanshortiss.com/photography](http://evanshortiss.com/photography)

Here's an example of how the module can be added to an Express app. You can load the middleware synchrnously by not providing a callback, but it is not recommended since if the initialisation fails your gallery will not be served as expected.

```javascript
'use strict';

var express = require('express')
  , path = require('path')
  , app = express()
  , port = 8001
  , middleware = require('express-flickr-gallery').middleware;

// Use any view engine that you like...
app.set('view engine', 'jade');
app.set('views', path.join(__dirname, './views'));

loadFlickrMiddleware();

// Load the flickr middleware
function loadFlickrMiddleware () {
  middleware.init(express, {
    flickr: {
      // If you place albums IDs in the below array then only those will be
      // shown in the list
      // albums: []
      api_key: 'YOUR_API_KEY',
      secret: 'YOUR_FLICKR_SECRET',
      user_id: 'YOUR_FLICKR_USER_ID'
    },
    templates: {
      // The strings here should correspond to your views
      albumList: 'album-list-template-name',
      album: 'album-page-template-name'
    }
  }, onGalleryLoaded);
}

// Once the middleware loads start the app
function onGalleryLoaded (err, route) {
  if (err) {
    // Retry
    setTimeout(loadFlickrMiddleware, 5000);
  } else {
   	app.use('/gallery', route);
    app.listen(port, function (err) {
      if (err) {
        throw err;
      }

      console.log('Go to 127.0.0.1:%s%s', port, '/gallery');
    });
  }
}
```


If you'd like to check out either module here are links to their source. They can also be installed via npm:

* [express-flickr-gallery](https://github.com/evanshortiss/express-flickr-gallery)
* [flickr-urls](https://github.com/evanshortiss/flickr-urls)

### Examples
![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2015/06/Screen%20Shot%202016-04-13%20at%209.12.07%20PM.png)
![](https://dl.dropboxusercontent.com/u/4401092/blog/images/2015/06/Screen%20Shot%202016-04-13%20at%209.05.42%20PM.png)
