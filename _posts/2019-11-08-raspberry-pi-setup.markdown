---
published: true
title: Raspberry Pi Quick Setup
layout: post
categories: raspberry-pi
redirect_from:
  - /raspberry-pi/2019/11/08/raspberry-pi-setup.html
---

This blog post is more so a list of reminders for myself than anything else,
but it will be helpful for anyone setting up their Raspberry Pi too. I did all
of this on a Raspberry Pi 2 Model B.

_Note: This blog post assumes you'll be using a variant of the Raspbian OS. If
you're using Fedora, Ubuntu, or another distro then some of these instructions
aren't valid (`raspi-config` will not be available for example) so YMMV._

## Flashing an OS Image

1. Download a Raspbian image from [raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/).
1. Download [balenaEtcher](https://www.balena.io/etcher/).
1. Use balenaEtcher to load the image onto a microSD.
1. Wait until balenaEtcheris finished.
1. Eject the microSD from your PC or Laptop.
1. Plug the microSD into the Raspberry Pi and boot it up with a keyboard and
monitor attached.

_Note: balenaEtcher is not strictly necessary, but it's a nice and easy way to perform the flash._


## Initial Login and Password Change

I might revisit this with steps to create a new user and delete the default
`pi` user, but will leave it as is for now.

1. Wait until the Pi finishes booting and prompts for a login.
1. Enter the username `pi`.
1. Enter the password `raspberry`.
1. Run the `passwd` command once logged in and follow the prompts to create a new password.


## Configure Internationalisation and Keyboard Layout

1. Enter the `sudo raspi-config` command.
1. Choose _Localisation Options => Change Locale_ and set as required.
1. Choose _Localisation Options => Change Keyboard_ and choose your keyboard.

If your keyboard was not listed try this instead:

1. Exit `raspi-config` using the `Esc` key.
1. Enter `sudo vi /etc/default/keyboard`.
1. Change the value of `XKBLAYOUT` to a valid code, e.g `us`.

## Configure WiFi

I used a WiFi dongle to perform initial software installs and configuration
since it's easier. The WiFi adapter I use is the 
[OURLiNK from adafruit](https://www.adafruit.com/product/1012)) - it's slow,
but it gets the job done when you need a wireless option.

1. Enter the `sudo raspi-config` command.
1. Choose _Network Options_.
1. Select the _N2 Wi-fi_ option.
1. Enter the network SSID.
1. Enter the network password.
1. Exit `raspi-config` using the `Esc` key.
1. Test connectivity by issuing a command, e.g `curl https://whatsmyip.dev/api/ip`.

## Enable SSH Access

1. Enter the `sudo raspi-config` command.
1. Select _Interfacing Options => P2 SSH_.
1. Choose `Yes` when asked if you'd like SSH server to be enabled.

## Add a Little Extra Security

1. Run `sudo apt-get update` and `sudo apt-get install fail2ban`.
1. Enter `y` when prompted to perform the install.
1. Run `sudo vi /etc/fail2ban/jail.local` and paste content similar to the
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

1. Open a terminal.
1. Enter `ssh pi@192.168.x.x`.
1. When prompted to continue connecting enter `yes`.
1. Enter the password for the `pi` user when prompted.
1. Run the commands `cat /var/log/auth.log` and `cat /var/log/fail2ban.log`.
1. Verify that the output from the `cat` commands above show your SSH activity.
1. Type `exit` to quit the SSH session.
1. Enter `ssh pi@192.168.x.x`, but enter the wrong password a few times. You'll get banned!

Switch to your Pi and check `/var/log/fail2ban.log` to verify there's an
`[ssh] Ban 192.168.x.x` entry. You can unban your PC/Laptop from the Pi
directly by issuing a `sudo fail2ban-client unban --all` command.

## Relax

You're now ready to have some fun with your Pi.