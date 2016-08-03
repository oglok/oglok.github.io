---
layout: post
title: Fedora in a MacBook Pro
subtitle: Tweaks & Tricks to make it work
---

Some months ago I got a MacBook Pro 13.3 for work, and since I'm much more comfortable in a Linux environment, I decided to install Fedora. I really like Apple's hardware, especially the retina display, but for me, Fedora was the right OS to work with.

So let's see what I did in order to install Fedora 24.

##Boot Media

Still running OS X, download Fedora 24 Workstation ISO file from the following [link](https://getfedora.org/workstation/download/). Then, follow these steps:

* Open terminal
* diskutil list
* Insert the USB pendrive that you want to make bootable
* diskutil list
* Take the new disk in the list. It should appear as /dev/diskX being X a certain number. (i.e. /dev/disk1)
* diskutil unmountDisk /dev/disk1
* sudo dd if=~/Downloads/Fedora-Live-Workstation-x86_64-24-10.iso of=/dev/rdisk1 bs=1m (see there is an 'r' in front of the disk name. This will make this process faster)
* Wait for some minutes to finish.

If you want to make a dual boot system (in order to keep Mac OS X), I suggest to open diskutil graphic application and make an empty partition of your disk.

##Installation

Insert the USB drive where Fedora 24 ISO has been located and restart the laptop. Make sure to press Alt when it's booting up. That way, you will be able to choose Fedora from the USB stick.

There are dozens of tutorials about how to install Fedora in general. Just follow the steps from the screen and configure date/timezone, users, network, partition table, etc.

Once the process has finished, remove the USB stick and reboot the system. Now, GRUB boot loader will show up and you will be able to boot Fedora 24!


