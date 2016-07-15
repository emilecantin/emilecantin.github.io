---
layout: post
title: "Arduino and SignalK - Part 3: Making it \"Production-ready\""
categories: web sysadmin iot javascript sailing
---

_This is part 3 of a multi-part series on outfitting my boat with some basic instrumentation, using open-source software and off-the-shelf electronics._

In the previous part of this series, we got our Arduino reading sensor values, we wrote a bit of code to convert the values coming from the Arduino into the SignalK format, and we configured our SignalK server to read that data.

Now, this is pretty good, but we don't want to have to SSH into our Raspberry Pi while we're out sailing, do we? To make our system truly "plug and play", there's a few things we can (and will!) do.

## udev

As you might have noticed while working on the code which talks to the Arduino in the previous part of this series, the device path we use (`/dev/ttyUSB0` in our example) is not really constant. You might have had to change it, and you might have noticed that it changes when reconnecting the USB cable between the Pi and the Arduino. We definitely don't want to be editing configuration files while we're enjoying a breezy day on the water.

*udev* is Linux's device manager. It's this piece of software that decides how to name the devices on the system. Luckily, it's configurable, so we'll write a _rule_ to tell it to always give the same name to our Arduino. I followed the steps and explanation [here](http://rolfblijleven.blogspot.ca/2015/02/howto-persistent-device-names-on.html), but the short version was that I watched `dmesg | grep usb` for USB detection events, and I got the vendor and product ID from that, and then I wrote `/etc/udev/rules.d/99-usb-serial.rules`:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", SYMLINK+="arduino_sensors"
```

To make this effective, either reboot or run `udevadm trigger` as root.

We can now change the device path from `/dev/ttyUSB0` to `/dev/arduino_sensors` in our "driver", as the Arduino will always have that device path from now on.

## systemd

Our Arduino has stopped changing name so we don't have to edit files as often anymore, but we still don't want to have to log into our Pi and run commands to get our sailing instruments running. The way to have a program start on boot on our Raspberry Pi is by configuring a program called *systemd*.

*systemd* is a relatively new init system that's becoming more and more popular in the Linux world. It replaces the old *init* program, which was originally written in the 70s. It sometimes is controversial (there was quite a bit of drama when Debian adopted it as their default init system, for example), but having written both regular old init scripts and systemd unit files, let me say that the latter is much more convenient.
