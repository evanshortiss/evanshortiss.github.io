---
published: true
title: BlueJeans Primetime & AirPods Play/Pause
layout: post
categories: tech-tips
---

This a quick, oddly specific, tip for those of you that find yourself using
Apple AirPods during a BlueJeans Primetime call.

I have the "Automatic Ear Detection" setting enabled on my AirPods. This means
that my AirPods automatically pause audio/video playback on my iPhone or
MacBook when I remove one of them from my ears. It also restarts the audio/video
playback when I place the AirPod back in my ear. Neat!

The "Automatic Ear Detection" feature is fantastic, but it's not ideal when
combined with BlueJeans Primetime. Despite BlueJeans Primetime being a "live"
stream, it will pause when you remove an AirPod from your ear! BlueJeans
Primetime doesn't expose playback controls because it's live. Placing an
AirPod back in your ear is the only way to resume playback! Or is it?

A quick fix for this, without refreshing your BlueJeans Primetime tab/browser
after removing the AirPod is to open up the developer tools in your web browser
(usually using `Ctrl + Shift + I` on Windows and Linux, or `Cmd + Opt + I` on
macOS) and run this JavaScript in the console.

```js
document.querySelector('video').play()
```

I assume it might also be possible to use media controls if your
keyboard/device is equipped with them, but I haven't tested that.