---
layout: post
title: "Arduino and SignalK - Part 1: Running the server"
categories: web sysadmin iot javascript sailing
---

_This is part 1 of a multi-part series on outfitting my boat with some basic instrumentation, using open-source software and off-the-shelf electronics._

If you know me personally, you know I've been into sailing for a bit now. I've taken lessons on the lake near my house, sailing Lasers for two years, and this summer I finally bit the bullet and bought my own boat. She's not the latest and greatest, and she's almost twice my age, but at least she floats.

One thing about old boats, though, is that they don't come with much in the way of electronics. I've been planning on upgrading and improving some of it, but marine electronics are _pricey_.

While hanging out on various sailing-related forums and subreddits, I stumbled upon [SignalK](http://signalk.org/), an open-source data format for everything boat-related. Their demo had some nice web-based instruments, and their code was on [Github](https://github.com/SignalK/signalk-server-node). Being a Web Developer, this was perfect for me.

## Basic hardware

I already had a Raspberry Pi model B (the original one) sitting around, collecting dust since I've bought a Chromecast (it was my media center before), so I figured I'd use it for this project.

I also had some _very_ cheap Arduino clones I got from [AliExpress](http://www.aliexpress.com/item/Best-prices-UNO-R3-MEGA328P-for-Arduino-Compatible-Free-Shipping-Dropshipping/32213964945.html), so I had all I needed to get started.

## Setting up the Raspberry Pi

This was pretty easy; I used [raspbian-ua-netinst](https://github.com/debian-pi/raspbian-ua-netinst/), as I always do. This image is awesome: flash it on your SD card, pop it in your Pi, power it on, and about 15 minutes later you can SSH into it. My Raspberry Pi has a reserved IP address on my router so it's always the same, but you can look at the DHCP leases on your router to get your Pi's IP.

## Setting up the SignalK server

There are two SignalK server implementations, one in Java and the other in Node.js. I chose the Node.js one, because I know Node.js very well.

To install Node.js, I've used [NVM](https://github.com/creationix/nvm); it always works well for me. Simply follow the installation instructions (with the install script), and then actually install a Node.js version:

{% highlight bash %}
nvm install latest

nvm alias default latest
{% endhighlight %}

At this point, typing `node -v` should return a pretty recent version of node. If it doesn't, stop and debug this before continuing.

So now, we've got all the prerequisites ready, and we can actually install the SignalK server. The [instructions](https://github.com/SignalK/signalk-server-node#get-up-and-running) are pretty standard in the Node.js world:

* Clone the repo (https://github.com/SignalK/signalk-server-node)
* Install the dependencies (`npm install`)
* Start the server (`bin/nmea-from-file`)

You should then be able to open this and see a bunch of data: [http://raspberry:3000/signalk/v1/api/](http://raspberry:3000/signalk/v1/api/) (substitute your Pi's IP address)

That's it! The SignalK server is now running on your Pi. In the next part, we'll look at getting our own data into the server.
