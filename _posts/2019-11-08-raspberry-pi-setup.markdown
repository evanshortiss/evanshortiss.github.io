---
published: true
title: Raspberry Pi Quick Setup
layout: post
categories: raspberry-pi
---

This blog post is more so a list of reminders for myself than anything else,
but if you find it helpful then that's great too! 

Note that the blog post assumes you'll be using a variant of the Raspbian OS.
If you're using Fedora, Ubuntu, or another distro then most of these
instructions are not applicable.

## Flashing an OS Image

. Download a Raspbian image from [raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/).
. Download [balenaEtcher](https://www.balena.io/etcher/).
. Use balenaEtcher to load the image onto a microSD.
. Wait until balenaEtcheris finished.
. Eject the microSD from your PC or Laptop.
. Plug the microSD into the Raspberry Pi and boot it up.

_Note: balenaEtcher is not strictly necessary but it's nice and easy._


## Initial Login and Password Change

I might revisit this with steps to create a new user and delete the default
`pi` user, but will leave it as is for now.

. Wait until the Pi finishes booting and prompts for a login.
. Enter the username `pi`.
. Enter the password `raspberry`.
. Run the `passwd` command once logged in and follow the prompts to create a new password.


## Configure Internationalisation and Keyboard Layout

. Enter the `sudo raspi-config` command.
. Choose _Localisation Options => Change Locale_ and set as required.
. Choose _Localisation Options => Change Locale_:
. Find your keyboard and select it.
  . If your Keyboard was not listed exit `raspi-config` using the `Esc` key.
  . Enter `sudo vi /etc/default/keyboard`.
  . Change the `XKBLAYOUT` to a valid code, e.g `us` or `ie`.

## Configure WiFi

I use a WiFi dongle to perform initial software installs and configuration
since it's easier. The WiFi adapter I use is the 
[OURLiNK from adafruit](https://www.adafruit.com/product/1012)).

. Enter the `sudo raspi-config` command.
. Choose _Network Options_.
. Select _N2 Wi-fi_.
. Enter the network SSID.
. Enter the network password.
. Exit `raspi-config` using the `Esc` key.
. Test connectivity by issuing a command, e.g `curl https://whatsmyip.dev/api/ip`.

## Enable SSH Access

. Enter the `sudo raspi-config` command.
. Select _Interfacing Options => P2 SSH_.
. Choose `Yes` when asked if you'd like SSH server to be enabled.

## Add a Little Security

. Run `sudo apt-get update` and `sudo apt-get install fail2ban`.
. Enter `y` when prompted to perform the install.
. Run `sudo vi /etc/fail2ban/jail.local` and paste content similar to the
example below then run `sudo service fail2ban restart`:

```
[ssh]

enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
banaction = iptables-allports
bantime = 24h
maxretry = 3
findtime = 30m
```

## Verify SSH Access

Get the IP address of your Pi by issuing the `ifconfig` command on the Pi. If
you're using WiFi then the IP  will be listed next to the `wlan0` interface
with a value such as `192.168.x.x`.

Using a Laptop/PC on the same network as your Pi try the following:

. Open a terminal.
. Enter `ssh pi@192.168.x.x`.
. When prompted to continue connecting enter `yes`.
. Enter the password for the `pi` user when prompted.
. Run the commands `cat /var/log/auth.log` and `cat /var/log/fail2ban.log`.
. Verify that the output from the `cat` commands above show your SSH activity.
. Type `exit` to quit the SSH session.
. Enter `ssh pi@192.168.x.x`, but enter the wrong password a few times. You'll get banned!

Switch to your Pi and check `/var/log/fail2ban.log` to verify there's an
`[ssh] Ban 192.168.x.x` entry. You can unban your PC/Laptop from the Pi
directly by issuing a `sudo fail2ban-client unban --all` command.

## Relax

You're now ready to have some fun with your Pi.