---
published: true
title: Battling with the Android Download Manager
layout: post
categories: development javascript
---

As part of a project I was working on recently we needed to get a Cordova application to download PDF files generated provided by a Node.js backend to the Android, iOS, and Desktop Browser frontend applications. For iOS and Desktop this was trivial, but on Android it turned out to be bit of a struggle.

## Download Process & Structure
Due to some stringent security requirements we needed to generate a one-time download token for the PDF. This resulted in URLs with the following structure _/pdf/download/:unique-token_ which would invoke code that looked loosely like this:

{% highlight javascript %}
function getPdfRouteHandler (req, res, next) {
  pdfService.getPdf(req.params.token, function (err, data) {
    if (err) {
      next(err);
    } else {
      var buf = new Buffer(data, 'base64');

      // This is how we sent the PDF to the client
      res.header(
        'Content-Length',
        Buffer.byteLength(buf, 'base64').toString()
      );
      res.header('Content-Type', 'application/pdf');
      res.write(buf);
      res.end();
    }
  });
}
{% endhighlight %}

We ensured the PDF downloads were working for both iOS and Desktop first since those were a higher priority. Using the above code on the Node.js service, both iOS and Chrome were able to display the PDF successfully, but when Android's time came to shine things got pretty weird.

## Android Issues
When we opened the PDF download URL on Android the browser would wait while the Node.js service initiated the request, performed authentication etc. and then began streaming back the PDF data. Once the PDF data began to be streamed back the Android browser would hand the download to the Download Manager. This background download would consistently fail and this was causing a major headache.

## Angles of Investigation

#### HTTPS vs HTTP
Initially I found many results online claiming that Android PDF downloads over HTTPS were consistently failing. Considering I was testing with Android 4.4 I found it hard to believe this was still an issue. After performing a quick test over HTTP I confirmed this wasn't the cause of issue since HTTP and HTTPS both produced the same result.

#### Download Headers
Since the protocol was ruled out at this point I thought maybe the way Android's was interpreting headers was at fault. Investigations online indicated that this was an issue as per this [blog post](http://www.digiblog.de/2011/04/android-and-the-download-file-headers/). Trying all manner of header combinations continued to produce the same result.

Here are some examples, naturally we only ever tried one content type at a time:

{% highlight javascript %}
res.header('Content-Type' 'application/octet-stream');
res.header('Content-Type' 'application/pdf');
res.header('Content-Type' 'application/force-download');
res.header('Content-Disposition' 'attachment; filename="filename.PDF"');
{% endhighlight %}

## The Solution
After much frustration I thought that maybe none of the problems were with headers or protocols, but instead was something related to the Android Download manager. Looking at the download URL made me think "Well, the URL doesn't contain a file extension, so maybe when the Download Manager receives this it fails to recognise a filename". I added the .pdf to the URL and added support for this in the Node.js code and to my surprise this fixed the issue.

So, given the below example URLs only the second works:

1. _/pdf/download/5e0c4346-e228-49c2-99a6-78475052eb9e_
2. _/pdf/download/5e0c4346-e228-49c2-99a6-78475052eb9e.pdf_

Oddly enough, I couldn't recreate this issue locally using a basic express HTTP server running the same code snippet which leads me to believe it may somehow be linked an error in the Android Download Manager process when downloading over HTTPS or running code behind a proxy or load balancer. I might update this in the future with more thorough results and check for logs from logcat when I reproduce the failing downloads in a sandbox running on the same host.
