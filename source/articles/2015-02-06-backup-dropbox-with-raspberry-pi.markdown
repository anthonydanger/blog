---
layout: post
title: "Backup Dropbox with Raspberry Pi"
date: 2015-02-06
comments: false
tags: raspberry_pi, dropbox, aws, s3
---

With the recent release of the
[Raspberry Pi 2](http://www.raspberrypi.org/raspberry-pi-2-on-sale/) I thought
it would be good to document what I'm doing with my old one. I've tried to use
it as a media center or network backup drive. Neither of those really seemed
that useful. But I've finally found something that (for me) is really useful:
using the Pi to take nightly snapshots of my Dropbox, then archive that and
putting it on S3.

There are a few reasons why I want to do this:
1. Dropbox is a file syning tool, not an archiving tool. If you delete stuff 
  in your Dropbox, it will be deleated everywhere else you have it synced.
2. Because of number 1, just mirroring my Dropbox to S3 wouldn't work. If I 
accidently nuke my photos folder, I wanted it to be safely backed up somewhere
else.
3. The Glacier storage class is really cheap, so I can keep a huge number of 
Dropbox snapshots on S3 for very little money. 

##Set up the Pi
I have the following hardware:
* [Raspberry Pi Model B](http://www.raspberrypi.org/products/model-b/)
* USB charger from an old Android phone (be sure yours is the correct spec!)
* [Belkin powered 7-port USB hub](http://www.belkin.com/in/IWCatProductPage.process?Product_Id=534950)
* 1TB Western Digital USB drive (similar to MyPassport line)
* 8GB SanDisck ultra SD card

I am using a wired connection for the Pi. I
followed the excellent 
[NOOBS setup guide](http://www.raspberrypi.org/help/noobs-setup/) to get
Raspbian running.

Once you get your Pi up and running, you'll want to make a directory to mount
your USB drive. You'll probably want to modify your /etc/fstab to have the drive
auto-mount when the Pi boots up. 
[Here is a guide](http://www.techjawab.com/2013/06/how-to-setup-mount-auto-mount-usb-hard.html)

###Optional setup
You might want to set up Samba (many guides are available) on your Pi, and
configure it to have a static IP. I'm using an Apple Airport Extreme, and it's
really easy to reserve a local IP address based on the Pi's MAC address.

##Get files from Dropbox
There are a lot of ways to do this. Unfortunately the Linux Dropbox client will
not work on the Pi, as you must use a binary at some point and there are no ARM
ones available. I chose to use [rclone](http://rclone.org/), mostly because it
allows me to just pull down the *changes* to my Dropbox, not the whole thing. 



