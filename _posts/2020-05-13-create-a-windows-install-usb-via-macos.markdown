---
published: true
title: Creating a Windows 10 USB Installer using macOS in 2020
layout: post
categories: windows
---

You'd think creating a USB-based installer for Windows 10 would be easy using
macOS, or any modern operating system, but my experience following the guides
that Google returns for macOS wasn't great. Most guides are either outdated or
contain what appear to be errors in the Terminal commands, so the steps below
are what I used to create a working Windows 10 64-Bit USB installer.

This guide is mostly based on the one by Quincy Larson on
[freecodecamp.org](https://www.freecodecamp.org/news/how-make-a-windows-10-usb-using-your-mac-build-a-bootable-iso-from-your-macs-terminal/)
, but it contains modifications that I found necessary in my experience.


### Open a Terminal
We'll be using the Terminal on your Mac to create the bootable USB. If you
don't know how to open the Terminal App, then [read this](https://support.apple.com/en-gb/guide/terminal/apd5265185d-f365-44cb-8b09-71a064a42125/mac) - it only takes a second!

### Install Homebrew & Wimlib
You might already have Homebrew installed, so check by running `which brew`. If
the command prints `not found` then you need to run the following command to
install it:

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

Once you have Homebrew installed, quit the Terminal application (the shortcut
is CMD + Q), then reopen it. Next install wimlib by running
`brew install wimlib`.

### Download Windows
Download a Windows 10 ISO. You can do so at this link on [microsoft.com](https://www.microsoft.com/en-us/software-download/windows10ISO).

### Get Tea or Coffee ☕
Since the Windows 10 ISO is ~5.5GB, maybe now is a good time to have a drink?
It'll take a few minutes to download, even on a reasonably fast internet
connection. If you're lucky enough to have gigabit internet, then I guess you
can just wait the 50-ish seconds it'll take?

### Plug the USB Drive into your Mac
OK. I think you can figure this one out, but if you're using a new Macbook Pro
like me you probably need a USB-C to USB-A adapter, and that sucks.

### Identify the USB Drive
Run `diskutil list` in Termainal and a list attached disk drives will be
displayed. Your USB will probably be `/dev/disk2` or `/dev/disk3`. We'll
store that in a variable for use later by running the command
`DRIVE_MOUNT=/dev/disk2`.

I used `/dev/disk2` in the example `DRIVE_MOUNT`, but make sure you change it to the correct path for your USB drive. The next step involves formatting the drive. That means it deletes everything on the drive, so you want to be sure it's the right one!

### Format the USB Drive
Format the drive using the following commands:

_Note: The lines beginning with `#` are comments explaining what the command on the line below does. You don't need to paste them since they do nothing._

```bash
# create a variable for the drive name - it'll be handy later
DRIVE_NAME="WINSTALL"

# format the disk so we can boot the Windows installer from it
diskutil eraseDisk MS-DOS $DRIVE_NAME MBR $DRIVE_MOUNT
```

### Mount the ISO File
Enter `hdiutil mount` followed by a space character in the Terminal, then drag
the ISO file you downloaded into the Terminal. The end result will be a command
that looks similar to:

```
hdiutil mount /Users/your-username/Downloads/Windows.iso
```

Press enter to run the command. Once the ISO has mounted it will print the
mount path and volume name. The volume name will be similar to
`/Volumes/CCCOMA_X64FRE_EN-GB_DV9`, but varies depending on the version of
Windows and the language. Save it to a variable like so:

```bash
WIN_VOLUME="/Volumes/CCCOMA_X64FRE_EN-GB_DV9"
```

### Copy Files to the USB
Copy files to the USB by running the following command. Note that the
`install.wim` must be excluded since it's too large to copy, but we'll deal
with that next.

```bash
rsync -vha --exclude=sources/install.wim $WIN_VOLUME/* /Volumes/$DRIVE_NAME
```

### Copy install.wim to the USB
Use the following `wimlib` command to copy the final install file to the USB
drive: 

```bash
wimlib-imagex split $WIN_VOLUME/sources/install.wim /Volumes/$DRIVE_NAME/sources/install.swm 4000
```

### Unmount the USB & Install Windows
Eject the USB by running `diskutil unmount $DRIVE_MOUNT` and use it to install
Windows!