---
layout: post
title: Dell XPS 13 WiFi and Linux
---

I recently bought a [Dell XPS 13 "developer edition"](http://www.dell.com/us/business/p/xps-13-linux/pd) (basically a 2015 XPS 13
that comes with Ubuntu pre-loaded).  The laptop works quite well with the
exception of WiFi, for some reason Dell chose to include a [Broadcom BCM4352](https://wiki.archlinux.org/index.php/Dell_XPS_13_%282015%29#WiFi) adapter which
in turn is not supported by the mainline Linux kernel drivers. This decision to
include a Broadcom WiFi device when Intel ones are available really confuses me.

## The Problem

Broadcom does provide their own out of tree driver but this means that you will
always need a build of that driver to match your kernel.  As a result I had
broken WiFi right
after updating Ubuntu and when I switched to Fedora 22, there was not yet a way
to build the Broadcom driver for the 4.0.1 and above kernel, and I did not want
to patch their code to match up with internal API changes.

## The Fix

The solution is quite simple:

1. Purchase an Intel-based PCIE adapter.  These laptops use a PCIE mini card and any Intel 802.11AC dual-band WiFi/Bluetooth adapter will plug in.  There are two antennas on the laptop side.
2. Purchase or borrow a T5 screwdriver and a small (size 0 or so) Philips screw driver and a guitar pick (or similar plastic tool) and read the first part of [this excellent iFixit teardown](https://www.ifixit.com/Teardown/Dell+XPS+13+Teardown/36157)
3. Replace the Broadcom card with the Intel card.  I found it helpful to use tweezers to plug the antennas back in.

...and then everything will work fine.  The replacement process took about ten minutes and this laptop is not difficult to take apart.  I now have an Intel PCIE card rather than a Broadcom USB card (PCIE mini card connector brings out both per the standard).

![Replacement card installed](/assets/dell-wifi.jpg)

For reference, the card I purchased on eBay (sub-$30) was advertised as a Dell part and described as `Intel 7260NGW Wireless-AC Dual band 7260 802.11ac/a/b/g/n WiFi + BT 4.0 Card` with Dell part numbers `KTTYN 0KTTYN M7X42 0M7X42` however pretty much anything that matches the pictures of the card and number of antennas should work.
