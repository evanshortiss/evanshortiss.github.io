---
published: true
title: Automatic npm Registry Switching (npmrc daemon)
layout: post
categories: nodejs
---

*TLDR; My [npmrcd](https://github.com/evanshortiss/npmrc-daemon) package allows
you to automatically switch from the public registry to a private registry.
Switching occurs when the private registry can be resolved via DNS or your
machine connects to a specific network.*

Recently at work I had to switch from directly using the public [npmjs.com](https://www.npmjs.com/)
registry to an internal mirror/proxy server we're using. Doing this is pretty
straightforward since it can be achieved with just one or two commands as shown
below:

```bash
# Make your npm CLI use the registry at $CORPORATE_REGISTRY_URL
npm config set registry $CORPORATE_REGISTRY_URL

# Optional, but might be necessary depending on your org setup
npm config set cafile $CA_FILEPATH
```

Seems easy enough, but as someone who often travels for work, frequently works
from home, and is generally a forgetful guy, I knew this would quickly become
annoying. It was inevitable that I'd forget to switch between the corporate
registry at work and public registry as I moved between networks, projects,
and jumped on and off VPNs.

![](/res/img/posts/2019-12-08-npmrc-daemon/forgetful-pikachu.jpg)

In my quest to conquer my forgetfulness, and this problem worthy of
[/r/mildlyinfuriating](https://old.reddit.com/r/mildlyinfuriating/), I
decided to make switching registries an automated process. The conditions were
simple enough, as demonstrated by the pseudocode below:

```js
/**
 * Only use the public npm registry if:
 *  - the corporate registry cannot be resolved
 *  - and we're not on corporate WiFi
 */
if (dnsResolvesUrl(CORPORATE_NPM_REGISTRY_URL) || isConnectedToCorporateWifi()) {
  switchToCorporateRegistry()
} else {
  switchToPublicRegistry()
}
```

Effectively what I was looking for was a way to have a "work" and "personal"
profile for my npm configuration, and a mechanism that would automatically
determine which to use.

## Have no fear, npmrc is here!

The npm CLI manages your configuration by saving it in a file named *.npmrc*.
This file is typically stored in your home directory on MacOS/Linux. Creating
multiple *.npmrc* files and rotating them would allow me to achieve what I
needed.

I figured this had to be a solved problem, and sure enough I found the
[npmrc package](https://www.npmjs.com/package/npmrc) on npm. Using this package
I could easily create a new profile using `npmrc -c [profile-name]`, for
example I could do the following:

```bash
# Create the work profile
npmrc -c work

# Switch to the work profile
npmrc work

# Configure settings for this profile
npm config set registry $CORPORATE_REGISTRY_URL
npm config set cafile $CA_FILEPATH

# Switch back to the default profile
npmrc default
```

The npmrc package worked perfectly despite not being updated in 5 years, and
it has zero external dependencies!

![](/res/img/posts/2019-12-08-npmrc-daemon/perfection-meme.jpg)

Finding an existing tool to manage the profiles saved me some time and meant
I could focus purely on the automation work.

## Agents and Daemons

Automating the switch between profiles by running a script on a regular
interval seemed like a reasonable solution, so I pursued that avenue. Since
my primary development machine is a MacBook Pro, my search for setting up
a daemon on macOS lead me to [launchd.info](https://www.launchd.info/). The
website summarises how to create a daemon/agent as follows:

*The behavior of a daemon/agent is specified in a special XML file called a property list (plist). Depending on where it is stored it will be treated as a daemon or an agent.*

This seemed straightforward enough, until it wasn't! I wrote a script that
would use the *npmrc* CLI to switch profiles, created a *plist* and tried to
load it using the *launchctl* CLI. I spent a while placing my *plist* in
*/Library/LaunchAgents* and */Library/LaunchDaemons*, running variations of
`sudo launchctl load -w my-plist.plist`, and changing various properties in the
*plist* to get around issues such as:

* "why the hell can't it find the `node` executable? It's on my `PATH`!"
* "why doesn't the daemon run at all?"
* "oh I needed to specify the user and group it should run as"
* "wait, it's still not working"

Eventually when I moved the *plist* into the folder *~/Library/LaunchAgents/*
and ran `launchctl load -w my-plist.plist` it worked! This confusion is
probably my own fault since I just looked at the summary on the homepage of
[launchd.info](https://www.launchd.info/) and didn't really
[RTFM](https://en.wikipedia.org/wiki/RTFM). Basically, I needed to place it in
the Agents folder for my specific user.

The final *plist* was similar to this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.npmrcd</string>
    <key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>my-path-string</string>
      <key>HOME</key>
      <string>/Users/my-username</string>
    </dict>
    <key>Program</key>
    <string>my-daemon-script.js</string>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>120</integer>
    <key>StandardOutPath</key>
    <string>/tmp/npmrcd.stdout</string>
    <key>StandardErrorPath</key>
    <string>/tmp/npmrcd.stderr</string>
  </dict>
</plist>
```

This *plist* tells the OS to run *my-daemon-script.js* once every 120 seconds.
So, my Mac will check verify it's connected to the correct registry every two
minutes. Perfect!

## Putting it all Together

Once I had the Agent, lookup, and npmrc pieces figured out, I needed to wire
them together. To this end, I created a small Node.js script that would be
executed by the macOS Agent every couple of minutes to:

* Perform a DNS lookup to verify if the corporate registry could be resolved.
* Check the WiFi SSID that my Mac is connected to.
* Based on the DNS/WiFi results invoke the `npmrc` command to change registry.
* Notify that the change has occurred.

I wasn't sure how best to notify myself of the change since this would be a
process running in the background, but then I recalled that various Node.js
modules I've used trigger native desktop notifications. Research lead me to
the conclusion that the `node-notifier` module seems to be the most common
solution used by other modules for triggering a native notifications, and it
worked perfectly for me too - just look at that glorious notification in the
screenshot below.

The entire solution is wrapped together as a neat little module called `npmrcd`
so others can easily use it. If you frequently switch between registries give
it a try like so:

```bash
# create a work profile with the following settings
npx npmrcd --registry=https://repository.acme.com/nexus/repository/registry.npmjs.org
```

If you need to use a CA file that option is supported. The CA file can point to
a local file or HTTP/HTTPS endpoint that the file can be fetched from:

```bash
# create a work profile with the following settings
npx npmrcd \
--cafile=~/REGISTRY-CA.crt
--registry=https://repository.acme.com/nexus/repository/registry.npmjs.org
```

Finally, you can specify that certain WiFi SSIDs should trigger the switch, even
if the registry is not accessible:

```bash
# even if the registry url does not resolve, switch if connected to
# the "Red Hat Guest" WiFi to prevent using the public registry
npx npmrcd \
--triggerssid='Red Hat Guest'
--cafile=~/REGISTRY-CA.crt
--registry=https://repository.acme.com/nexus/repository/registry.npmjs.org
```

Here's a screenshot showing my registry switch when I first opened my laptop
in the office at 10:25AM - hey, I was on calls at 7AM at home before getting
there so give me a break on getting there at 10:25AM! I hadn't plugged into
the network via an Ethernet cable yet hence the "currently not accessible"
message. You can see that when I opened my laptop at home later that day it
switched back to the public npm registry.

![](/res/img/posts/2019-12-08-npmrc-daemon/npmrcd-notification.png)

I hope this post and module are helpful, or even mildly interesting!