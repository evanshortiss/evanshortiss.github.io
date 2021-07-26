---
published: false
title: How I fixed my AMD Driver Timeout Issues and retained 10 Bits per Color
layout: post
categories: windows amd drivers
---

## The Issue

Does the following image increase your blood pressure? It sure as shit had an
affect on mine this past year. The idea of having a dud GPU in the current
market is a nightmare.

![AMD Driver Timeout Dialog](https://i.imgur.com/vMw4VvX.png)

I'm the proud owner of an MSI "Radeon RX 5600 XT GAMING X". It's a pretty great
GPU for the price. Well, that's assuming you can get it for a reasonable price
these days. Unfortunately it has a major fucking issue when gaming in Windows
on my 144Hz 1440p monitor: the driver crashes. And yes, based on my testing
it doesn't happen gaming in Fedora Linux 34.  

This GPU had a [bit of rough a launch](https://www.techspot.com/news/85113-amd-recommends-radeon-rx-5600-xt-owners-update.html),
so I cut it some slack when this issue started occurring. After multiple driver
updates I started to lose hope though. I realised was in a rough situation
due to the GPU market. Returning it would result in credit/cash towards a
new, massively overpriced, GPU. After trying ~basically~ literally every fix
posted online, I eventually found that I could "fix" the problem by lowering my
refresh rate to 120Hz.

Is this the definition first world problem? Yes, but on the otherhand I paid
for the Hz and the colours, so [I'm gonna use all the Hz and the colours](https://www.reddit.com/r/pcmasterrace/search?q=use%20all%20the&restrict_sr=1)!

## The Solution

*NOTE: I'm pretty sure this is harmless, but it's on you to look into what's appropriate and safe for your hardware/software. I'm not responsible for any damage, fucks ups, annoyance, or whatever else happens if this doesn't work out for you.*

When AMD released the 21.7.1 drivers I decided to see if the crashing issue
was fixed, but I wasn't surprised when I found out that the issue persists.
I noticed something during my testing though; my VRAM was constantly locked at
1750Mz at 144Hz. As soon as I changed my refresh rate to 120Hz it was able to
fluctuate, and generally stayed around 200Mhz at idle.

Using the VRAM clock speed clue leads us to this fantastic solution by
[Hard Reset on YouTube](https://www.youtube.com/watch?v=Td3mBgE1Dsc). Creating
a Custom Resolution solution solved my issue, but after a few days I noticed
that image quality seemed amiss. Areas of uniform colour seemed to have a
"noise" or "grain" and gradients weren't as smooth as I'd become accustomed to
on my monitor.

This was because the custom resolution was limiting colour depth. You can see
that I was limited to 6-bit colour in Radeon Software!

![Radeon Software Limited to 6 bpc](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/radeon-6bpc.png)

I spent a few minutes tweaking the custom resolution settings to try get it to
expand the colour depth while retaining a high-refresh rate, but was ultimately
unsuccessful. 

This is where [CRU (Custom Resolution Utility)](https://www.techspot.com/downloads/7345-custom-resolution-utility.html)
saved the day. 

I downloaded CRU and started it up. Here's how I got a high-refresh rate that
supported the full colour depth of my monitor: 

. Took note of the **Pixel Clock** listed beside the default entry for 144Hz.
. Added a new entry under the **Detailed Resolutions**.
. Selected *Automatic (PC)* from the **Timing** dropdown.
. Set the **Refresh Rate** to a value that results in a **Pixel Clock** that's equal to, or less than, the default value.
. Click the OK button to save the custom resolution.

At this point you will have two resolutions listed like so:

![CRU Listing](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/cru-list.png)

*NOTE: The **Pixel Clock** does not have to match perfectly. I managed to get them to align at 568.72Mhz somehow, but can't get it to happen when I try again!*

Click the "OK" button to close CRU. Run the **restart64** executable included
with CRU. Your screen will go blank for a moment, then come back to life. Now,
head over to the Windows display settings.

![Windows display settings](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/display-settings.png)

Verify that the selected resolution matches that one you created in CRU, then sroll down and select
**Advanced display settings**.

![Windows display settings before](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/advanced-display-settings-before.png)

Take note of the listed **Bit depth**. Select your custom refresh rate. The
screen will go blank briefly, and should return to normal after a second. If
everything seems fine click the **Keep changes** button. If the screen remains
blank don't panic! It just means your custom refresh and resolution isn't
supported, but Windows will revert the change after 30 seconds.

If you were able to select **Keep changes**, verify that the listed
**Bit depth** matches what it was previously. You should also verify that your
monitor is displaying the refresh rate correctly. Many monitors include an
overlay that displays the active refresh rate, such as th example at
[this link](https://www.tftcentral.co.uk/images/lg_27gl850/P1210225.JPG).

You can see that my custom refresh rate of 136Hz retains 10 bit depth. It
might be possible to push the refresh rate higher and maintain the 10 bits, but
I'm happy with initial discovery.

![Windows display settings after](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/advanced-display-settings-after.png)