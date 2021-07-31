---
published: true
title: How I fixed my AMD Driver Timeout Issues and retained 10 Bits per Color
layout: post
categories: windows amd drivers
---

## The Issue

Does the following image increase your blood pressure? It sure as shit had an
affect on mine this past year. The idea of having a dud GPU in the current
market is a nightmare. For some people this dialog isn't displayed, but they
get a warning in their Windows logs that states "driver amdkmdag
stopped responding and has successfully recovered".

![AMD Driver Timeout Dialog](https://i.imgur.com/vMw4VvX.png)

There are many potential causes for this issue. Poor RAM compatibility,
unstable RAM timings, unstable overclock, and a weak PSU seem to be common
causes. In my case it appears to be a specific AMD Driver issue due to my GPU
and monitor combination.

I'm the proud owner of an MSI *Radeon RX 5600 XT GAMING X*. It's a fantastic
GPU for the price; assuming you can find one at MSRP these days. It's cooler is
an [absolute unit](https://knowyourmeme.com/memes/absolute-unit), so it has a
nice feature that allows it to disable the fans when not gaming. Unfortunately
the AMD drivers combined with my specific GPU and monitor have a major fucking
issue when gaming at 1440p 144Hz in Windows; the driver crashes occasionally.
And yes, based on my limited testing, Fedora Linux 34 does not suffer from this
issue when gaming at 1440p 144Hz.

This GPU had a [bit of rough a launch](https://www.techspot.com/news/85113-amd-recommends-radeon-rx-5600-xt-owners-update.html),
so I cut it some slack when this issue started occurring. After multiple driver
updates I started to lose hope though. The awful state of the GPU market made
matters even worse. Returning it would result in credit/cash towards a
new massively overpriced GPU. Worse again, I bought it from the UK so I wasn't
sure how the Brexit shenanigans might affect the return process. Another matter
of concern is that this appears to be a software issue when using certain
monitors and refresh rates, so the return might have been rejected.

After trying ~~almost~~ literally every fix posted online, I eventually
found that I could "fix" the problem by lowering my refresh rate to 120Hz. I
wasn't totally satisfied with this so I kept digging, as you'll see in this
post.

Is this the definition first world problem? Sure is. But on the other hand I
paid for the whole monitor and GPU, so
[I'm gonna use the whole monitor and GPU](https://www.reddit.com/r/pcmasterrace/search/?q=i%20paid%20for%20the&restrict_sr=1)!

## The Clue

When AMD released the 21.7.1 drivers I decided to see if the crashing issue
was fixed. Spoiler alert: it's not fixed, at least not for some of us. During
my latest test I noticed that my VRAM was constantly locked at 1750Mz when I
set my refresh rate to 144Hz. This is fine when gaming, but why would it be
locked so high when the PC is idle? As soon as I changed my refresh rate back
to 120Hz the VRAM was able to fluctuate between 200Mhz and 1750Mhz, depending
on workload. Why is it fine at 1750Mhz gaming at 120Hz and not when gaming at
144Hz? I still don't know, but I knew that I had a lead.

Using the VRAM clock speed clue leads us to a fantastic solution by
[Hard Reset on YouTube](https://www.youtube.com/watch?v=Td3mBgE1Dsc). Their
video demonstrates that creating a Custom Resolution solution solves the issue.
I created a 1440p custom resolution with a 142Hz refresh rate, and my crashing
problems were solved. So, is this it? Case closed? Sadly not!

Image quality seemed amiss when using my new custom resolution. It wasn't very
noticeable, but what has been seen cannot be unseen. Every time I opened a new
tab in my web browser I could see "grain" on the blank page. Areas of uniform
colour seemed to have a "noise" or "grain" and gradients weren't as smooth as
I'd become accustomed to on my monitor. This was because the custom resolution
was limiting colour depth. Radeon Software listed 8 and 10 bit colour options,
but selecting either would fail and revert back to 6 bpc.

![Radeon Software Limited to 6 bpc](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/radeon-6bpc.png)

## ~~The~~ My Solution

*NOTE: I'm pretty sure this is harmless, but it's on you to look into what's appropriate and safe for your hardware/software. I'm not responsible for any damage, fucks ups, annoyance, or whatever else happens if this doesn't work out for you.*

I'm not a video expert, but I assume the custom resolution configuration was
maxing out the available bandwidth on my DisplayPort. Now, I know for a fact
that I can run at 1440p 144Hz with 10 bit colour via DisplayPort on my hardware
since the non-custom resolutions support this, albeit with a once daily crash
when gaming. Knowing this, I spent a few minutes tweaking the custom resolution
settings in Radeon Software to try get it to expand the colour depth while
retaining a high-refresh rate. After 5 minutes I accepted defeat 😢

This is when I turned to [CRU (Custom Resolution Utility)](https://www.techspot.com/downloads/7345-custom-resolution-utility.html).
It was mentioned in one of the threads I found online when researching this
issue, so I figured it was worth a try.

Here's how I used CRU to create a high-refresh rate that supported the full
colour depth of my monitor:

1. Start the CRU program.
1. Take note of the **Pixel Clock** listed beside the default entry for 144Hz (or whatever resolution/refresh rate combination is causing trouble for you).
1. Add a new entry under the **Detailed Resolutions**.
1. Double click this new entry to open the Edit dialog.
1. Select *Automatic (PC)* from the **Timing** dropdown.
1. Set the **Refresh Rate** to a value that results in a **Pixel Clock** that's equal to, or less than, the default value.
1. Click the OK button to save the custom resolution.

At this point you will have two resolutions listed like so. The **Pixel Clock**
of the new resolution does not have to match the original perfectly, just be
make sure it's **equal or lower**.

![CRU Listing](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/cru-list.png)

I managed to get them to align at 568.72Mhz while keeping a 136Hz refresh rate
somehow by setting the timing back to **Manual** and editing the
**Pixel Clock**, but can't get it to happen when I try again when testing it
for this blog! I'm too lazy to spend more time trying to squeeze the refresh
rate higher, and the difference between 144Hz and 136Hz is less than half a
millisecond so who cares. Wait, couldn't you make a similar argument for 120Hz
vs 136Hz and that I shouldn't have wasted my time on this first world problem? 🤔

<table style="text-align: center;margin: auto;padding: 1.5em 1em;">
  <thead>
    <tr>
      <th style="padding: 0 1em;">Refresh Rate</th>
      <th style="padding: 0 1em;">Frametime</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>144Hz</td>
      <td>6.944ms</td>
    </tr>
    <tr>
      <td>136Hz</td>
      <td>7.353ms</td>
    </tr>
    <tr>
      <td>120Hz</td>
      <td>8.333ms</td>
    </tr>
    <tr>
      <td>100Hz</td>
      <td>10.000ms</td>
    </tr>
    <tr>
      <td>60Hz</td>
      <td>16.666ms</td>
    </tr>
  </tbody>
</table>

No, this effort is for "science"! Anyways, click the "OK" button to close CRU.
Run the **restart64** executable included with your CRU download. Your screen
will go blank for a moment, then come back to life. Now, head over to the
Windows display settings.

![Windows display settings](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/display-settings.png)

Verify that the selected resolution matches that one you created in CRU, then sroll down and select
**Advanced display settings**.

![Windows display settings before](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/advanced-display-settings-before.png)

Take note of the listed **Bit depth**, e.g 10-bit in my case. Select your
custom refresh rate. The screen will go blank briefly, and should return
to normal after a second. If everything seems fine click the
**Keep changes** button. If the screen remains blank don't panic! It just
means your custom refresh and resolution isn't supported, but Windows will
revert the change after 30 seconds.

If you were able to select **Keep changes**, verify that the listed
**Bit depth** matches what it was previously. You should also verify that your
monitor is displaying the refresh rate correctly. Many monitors include an
overlay that displays the active refresh rate. For example, my monitor has it
built into the settings overlay [like this](https://www.tftcentral.co.uk/images/lg_27gl850/P1210225.JPG).
This verification is important, since I found that Windows might accept the
custom resolution and refresh rate, but in reality your monitor will only
output half of the refresh rate. For example, I had a 140Hz refresh rate
configured, but my monitor was only displaying at 70Hz which was pretty
noticeably slower than 140Hz, and also easily confirmed via the monitor
settings!

You can see that my custom refresh rate of 136Hz retains 10 bit depth. It
might be possible to push the refresh rate higher and maintain the 10 bits, but
I've been unable to find a setting that does so with my monitor and GPU. A few
tutorials state that increasing the blanking on the default resolution profile
does the trick too, so it might be worth trying.

![Windows display settings after](/res/img/posts/2021-07-26-amd-driver-memory-lock-timeout/advanced-display-settings-after.png)
