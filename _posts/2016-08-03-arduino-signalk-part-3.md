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

*systemd* is a relatively new init system that's becoming more and more popular in the Linux world. It replaces the old *init* program, which was originally written in the 70s. It sometimes is controversial (there was quite a bit of drama when Debian adopted it as their default init system, for example), but having written both regular old init scripts and systemd unit files, let me say that the latter is much more convenient. So, let's get to it!

A systemd unit lives in `/etc/systemd/system/` and looks like this:

```
[Unit]
Description=Some service
Requires=dependency.target
After=dependency.target

[Service]
ExecStart=/path/to/file/to/run.sh

[Install]
WantedBy=dependent.target
```

Looking at this, we understand that systemd works with _dependencies_. That means, we can specify other services, or _targets_ (a target is a bundle of services, if I understand correctly) which we need to be running for our service to run correctly. We can also specify the service or target that triggers the starting of our service. If you've worked with _run levels_ before, it's kind of similar. Let's add some explanations to our example unit file:

```
[Unit] # This section describes our service
Description=Some service # a human-readable description for our service
Requires=dependency.target # service(s) or target(s) that need to be running alongside our service
After=dependency.target # service(s) or target(s) that need to be running before our service starts

[Service] # This section describe what is actually being run
ExecStart=/path/to/file/to/run.sh # The executable / script to run. It must not fork.

[Install] # This section describes what happens when we enable our service
WantedBy=dependent.target # service(s) or target(s) that will pull our service when it starts.
```

In our case, we pretty much only need the network to be up for SignalK to work, so we'll specify that as our dependency. According to the doc, it's going to be started anyway, so we don't need `Requires`, only `After` is necessary.

The target which will pull our service is `multi-user.target`, as it's at this stage that we can login via SSH or console, which is pretty much what we need.

Now that the hardest parts are figured out, we can write our unit file:

```
# /etc/systemd/system/signalk.service
[Unit]
Description=SignalK server
After=network.target

[Service]
ExecStart=/root/start-signalk.sh

[Install]
WantedBy=multi-user.target
```

To make systemd aware of our new unit, we can run `systemctl daemon-reload`, and then we can enable it: `systemctl enable signalk.service`, which will make it start when we boot. If we don't want to reboot immediately, `systemctl start signalk.service` will start it immediately.

I used the [Arch Linux wiki](https://wiki.archlinux.org/index.php/Systemd#Writing_unit_files) and the [systemd man page](https://www.freedesktop.org/software/systemd/man/systemd.unit.html) to learn how to do this. They're great sources of information, go check them out!

At this point, we've got the SignalK server running automatically, it recognizes the Arduino when we plug it in, and we don't have to log in to our Raspberry Pi all the time. We're starting to have a solid setup! In the next part of the series, we'll add an instrument display, and we'll start customizing it.
