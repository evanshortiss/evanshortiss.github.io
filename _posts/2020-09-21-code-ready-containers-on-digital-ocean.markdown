---
published: true
title: Hosting CodeReady Containers on DigitalOcean
layout: post
categories: development openshift
---

# Hosting CodeReady Containers on DigitalOcean

This is a slightly customised version of Jason Dobies' post [on the Official OpenShift blog](https://www.openshift.com/blog/accessing-codeready-containers-on-a-remote-server/). That blog was inspired by Trevor McKay's GitHub Gist [found here](https://gist.github.com/tmckayus/8e843f90c44ac841d0673434c7de0c6a). That means I'm being incredibly derivative with this one, but I had fun and I hope you find it interesting!

Like Jason, I sometimes want admin access to a full, but lightweight OpenShift development environment. Running [CodeReady Containers](https://developers.redhat.com/products/codeready-containers/overview) on my Macbook works well, but it's not fun when Firefox, Chrome, Visual Studio Code are also in competition for my 16 gigs of memory. I'd also rather my Macbook's fans didn't spin at 100% speed all the time.

I do have a gaming PC with plenty of power to run CodeReady Containers, but it only has a single 1TB drive with Windows 10 installed. Partitioning my drive, or running CodeReady Containers on Windows didn't sound like exciting prospects, and waiting for an SSD delivery would require patience, so I figured I'd try something else.

This post outlines how I setup a DigitalOcean Droplet to run OpenShift 4 via CodeReady Containers, and how I configured my Macbook's DNS to communicate with it.

## DigitalOcean Droplet Setup

I'm going to assume you have a DigitalOcean account with a credit card attached to it. If you don't, go ahead and set that up before reading any further. They're not going to let you run a Droplet with 4 vCPU and over 16GB of RAM for free!

Get started by creating a new project on DigitalOcean. Name it something boring, like "CRC OpenShift 4", and give it a similarly sensible description.

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/do-setup-00.png)

{:.caption}
Creating a project for CodeReady Containers on DigitalOcean.

Next, create a **Fedroa 32** Droplet in that project you just created. You'll want something beefy since the [CodeReady Containers documentation](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/1.0/html/getting_started_guide/getting-started-with-codeready-containers_gsg#minimum-system-requirements_gsg) states that it requires:

* 4 vCPUs
* 8GB RAM (I've found it uses closer to 9GB)
* 35 GB of disk space

A Basic Droplet type with 8 vCPUs and 16GB RAM does the trick. This costs approximately 12 USD cents an hour, or 80 USD/month. Even though this is a throwaway single node OpenShift cluster, make sure you configure an SSH Key for authentication, because [professionals have standards](https://youtu.be/9NZDwZbyDus?t=68) (warning: cartoon violence)!

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/do-setup-01.png)

{:.caption}
The Droplet I chose has 8vCPUs and 16GB RAM. Noice!

While the Droplet is being created, head to the *Networking* section and click *Create Firewall* in the *Firewalls* section. Add the following rules:

* SSH
* HTTP
* HTTPS
* TCP 6443 (use the *Custom* option in the *New Rule* dropdown)

Your rules should look similar to the screenshot below. Don't forget to actually apply the rules to your new Fedora 32 Droplet!

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/do-setup-04.png)

## Software Versions and Other Notes

The bash commands throughout this guide reference a `$DROPLET_IP` variable. As you may have guessed, this is the IPv4 address that's listed alongside your shiny new Fedora 32 Droplet in the DigitalOcean UI. Anywhere you see `$DROPLET_IP`, replace it with your Droplet's IPv4 address.

Also, take note of this list. If you encounter issues recreating my setup, it might due to a version mismatch* with these: 

* Fedora 32
* CodeReady Containers version: 1.16.0+bf72d3a
* HA-Proxy version 2.1.7 2020/06/09
* NetworkManager 1.22.10-1.fc32
* firewalld 0.8.3
* libvirtd 6.1.0

_* and hopefully not an error in my instructions!_

## User Setup

Once your Droplet finishes can SSH into it. Read this [DigitalOcean How-To](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/) for SSH if you're having trouble.

CodeReady Containers should not be run as the root user, so the first thing to do is create another user, e.g `crc-user`.

1. `ssh root@$DROPLET_IP`
1. `useradd crc-user` 
1. `passwd crc-user`
1. `sudo usermod -aG wheel crc-user`
1. `exit` (the next command is run from your development/client machine)
1. `ssh-copy-id crc-user@$DROPLET_IP`

There's more you can do to secure your Droplet, but that's outside the scope of this guide. Step 2 in [this guide](https://www.digitalocean.com/community/tutorials/initial-setup-of-a-fedora-22-server) for Fedora is a start!

## Install Dependencies and Confgure firewalld

1. `ssh crc-user@$DROPLET_IP` (note that it's `crc-user` this time!)
1. `sudo dnf -y install NetworkManager haproxy firewalld policycoreutils-python-utils`
1. `sudo systemctl start libvirtd`
1. `sudo systemctl start firewalld`
1. `sudo firewall-cmd --add-port=80/tcp --permanent`
1. `sudo firewall-cmd --add-port=6443/tcp --permanent`
1. `sudo firewall-cmd --add-port=443/tcp --permanent`
1. `sudo systemctl restart firewalld`
1. `sudo semanage port -a -t http_port_t -p tcp 6443`

Before moving on, verify that a valid `nameserver` entry is set in `/etc/resolv.conf`. The primary nameserver needs to be set to `127.0.0.1` so OpenShift Routes can be resolved. I used `1.1.1.1` as my secondary nameserver. The updated `/etc/resolv.conf` will look like this:

```
# Generated by NetworkManager
nameserver 127.0.0.1
nameserver 1.1.1.1
```

## Download and Run CodeReady Containers

I've inclued a URL in the first step here for completeness, but if it fails you can check [cloud.redhat.com/openshift/install/crc/installer-provisioned](https://cloud.redhat.com/openshift/install/crc/installer-provisioned) to find a working URL.

1. `curl https://mirror.openshift.com/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz > crc-linux-amd64.tar.xz`
1. `tar -xf crc-linux-amd64.tar.xz`
1. `sudo mv crc-linux-1.16.0-amd64/crc /usr/local/bin/`
1. Use `crc version` to verify the binary is on the `PATH`. If it's working correctly version info should be printed to the console.
1. `crc setup`.

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-pull-secret.png)

{:.caption}
Obtain the Pull Secret under the cluster creation UI. You can also find the CodeReady Containers download URL here by right-clicking the "Download Code-Ready Containers" button and choosing "Copy Link Location".

To start CodeReady Containers you need a "pull secret". This is found on [cloud.redhat.com/openshift](https://cloud.redhat.com/openshift/):

1. Login to [cloud.redhat.com/openshift](https://cloud.redhat.com/openshift/).
1. Select *Clusters* from the side menu.
1. Click the *Create* button on the clusters listing page.
1. Choose *Red Hat OpenShift Container Platform* as the cluster type.
1. Choose the *Run on Laptop* option.
1. Click the *Copy Pull Secret* button.
1. Return to your SSH session.
1. Paste your pull secret into a file, e.g `vi ~/crc-pull-secret.json` and paste the contents.
1. `crc start -p ~/crc-pull-secret.json`

CodeReady Containers takes a minute or two to start, so this is a good time to mindlessly browse Reddit or Twitter.

## Configure HAProxy

HAProxy will route HTTP and HTTPS traffic to CodeReady Containers:

1. Backup the default HAProxy configuration: `sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.original`
1. Obtain the CodeReady Containers VM IP using the `crc ip` command.
1. Paste the following configuration into `/etc/haproxy/haproxy.cfg`, being sure to **replace the `CRC_IP`** strings with the IP obtained using `crc ip`:

```
global
        debug

defaults
        log global
        mode    http
        timeout connect 5000
        timeout client 5000
        timeout server 5000

frontend apps
    bind 0.0.0.0:80
    bind 0.0.0.0:443
    option tcplog
    mode tcp
    default_backend apps

backend apps
    mode tcp
    balance roundrobin
    option tcp-check
    server webserver1 CRC_IP check port 80

frontend api
    bind 0.0.0.0:6443
    option tcplog
    mode tcp
    default_backend api

backend api
    mode tcp
    balance roundrobin
    option tcp-check
    server webserver1 CRC_IP:6443 check port 6443
```

4. Start HAProxy `sudo systemctl start haproxy` (if this fails use `haproxy -f /etc/haproxy/haproxy.cfg -db` to view logs and diagnose the issue)

Note that the **CodeReady Container VM IP can change** on reboots/restarts. If the IP changes you'll need to update the `haproxy.cfg` with the new IP and run `sudo systemctl restart haproxy`!

## Configure DNS on your Development Machine

As Jason said in his post, there's no right or wrong way to do this, but you'll need to tell your development machine how to resolve the CodeReady Container URLs, i.e to resolve URLs with the `testing` suffix to your `$DROPLET_IP`.

This is pretty easy to configure for the static OpenShift routes via `/etc/hosts` on Linux and macOS. Add the following line to your `/etc/hosts` file: 

```
# CodeReady Containers (running on DigitalOcean)
# REMEMEBER: you need to replace $DROPLET_IP with your actual droplet ip
$DROPLET_IP api.crc.testing oauth-openshift.apps-crc.testing console-openshift-console.apps-crc.testing default-route-openshift-image-registry.apps-crc.testing
```

DNS resolution for deployed application Routes is a little less straightforward. This is because they are wildcard routes, i.e `*.apps-crc.testing`. I had hoped my [PiHole](https://pi-hole.net/) would help with this, but it doesn't seem to support wildcards. Instead used **dnsmsaq**. 

These commands will configure **dnsmasq** to resolve your CodeReady Containers installation on DigitalOcean on macOS:

```bash
brew install dnsmasq

# Commands based on: https://asciithoughts.com/posts/2014/02/23/setting-up-a-wildcard-dns-domain-on-mac-os-x/
echo "address=/.testing/$DROPLET_IP" > /usr/local/etc/dnsmasq.conf

# Copy the daemon config plist
sudo cp -fv /usr/local/opt/dnsmasq/*.plist /Library/LaunchDaemons
sudo chown root /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

# Approve the request to "accept incoming" connections if asked
sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

# Redirect DNS queries for "testing" to localhost
sudo mkdir -p /etc/resolver
sudo sh -c 'echo "nameserver 127.0.0.1" > /etc/resolver/testing'
```

Finally, in the Network settings for your device make `127.0.0.1` your first nameserver, then follow it up with another, e.g `1.1.1.1` and/or `8.8.8.8`. 

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-dns-setup.png)

{:.caption}
macOS Network settings to resolve the `testing` domain via dnsmasq, and 1.1.1.1 for everything else.

## Revel in Your Success

Once you have DNS configured, navigate to [console-openshift-console.apps-crc.testing](https://console-openshift-console.apps-crc.testing/) in your web browser. You'll receive some very exciting SSL warnings. Yes, these are exciting because it means you've been successful!

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-ssl-warning.png)

{:.caption}
The most exciting self-signed SSL certificate error you'll see this week.

Add an exception for the self-signed SSL certificates, and voila! You'll be brought to the OpenShift login page. You can login using credentials found by running `crc console --credentials` on your Droplet - hell yeah!

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-openshift-console.png)

{:.caption}
The OpenShift Projects list. You can see I created a Project to test the basic cluster functionality.

Create an OpenShift Project, and deploy a template. I went with the Node.js application because I'm a *bit* of a JavaScript fanboi. You can see my Node.js application running in the Developer Topology view in the screenshot below.

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-javascript-app-00.png)

{:.caption}
A Node.js application running in my Project.

Clicking the application node presents a details pane on the right. Open the URL under Routes and briefly marvel at the wonders of cloud technology.

![](/res/img/posts/2020-09-21-code-ready-containers-on-digital-ocean/crc-javascript-app-01.png)

## Configure OpenShift User Logins

Last but not least, you'll want to change the default login details. Those are the ones that can be obtained via `crc console --credentials`. The `developer` user it provides has a default password that should be changed.

Here's how you can update it:

```bash
ssh crc-user@$DROPLET_IP

# This will make the OpenShift CLI (oc) available
eval $(crc oc-env)

# Installs htpasswd, among other things
sudo yum install httpd-tools

# Get the kubeadmin credentials 
crc console --credentials

# Login as kubeadmin (you can omit the -p flag so it prompts you for the password)
oc login -u kubeadmin -p $KUBEADMIN_PASSWORD https://api.crc.testing:6443

# Save the current htpasswd file that OpenShift is using to a local user.htpasswd file
oc get secrets/htpass-secret -n openshift-config -o jsonpath="{.data.htpasswd}" | base64 -d > users.htpasswd

# Update the password for the "developer" user. You can also use htpasswd
# to add more users if you want
export DEVELOPER_PASSWORD="newsafepassword"
htpasswd -c -B -b users.htpasswd developer $DEVELOPER_PASSWORD

# Output the htpasswd in base64 format
cat users.htpasswd | base64
```

Visit [console-openshift-console.apps-crc.testing/k8s/ns/openshift-config/secrets/htpass-secret/yaml](https://console-openshift-console.apps-crc.testing/k8s/ns/openshift-config/secrets/htpass-secret/yaml) and replace the existing htpasswd value in the YAML with the new one, then click *Save*. You can also do this via `oc patch` or `oc edit` commands.

Verify that the new password is working by logging in as the `developer` user:

```bash
oc login -u developer
```

## That's a Wrap

Don't forget to shutdown the Droplet when you're not using it!

![Mission Accomplished GIF](https://media2.giphy.com/media/8UF0EXzsc0Ckg/giphy.gif)