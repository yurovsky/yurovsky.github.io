---
layout: post
title: ThinkPad X1 Carbon second generation with Linux
---

I recently received a ThinkPad X1 Carbon (second generation, type 20A7) at work
and this is my quick review with some notes about using it as a Linux machine.
As an executive summary: yes, this laptop is great for running Linux, pretty
much everything works out of the box including the "adaptive" keyboard.

I made an Ubuntu 14.04 64-bit USB disk and booted off of it (hit F12 at the
BIOS screen to select a boot device, or press Enter to interrupt the boot
process and then F12) and installed Ubuntu like normal, erasing the entire
hard disk.

I must admit that, given the relative newness of the laptop at the time of
writing, I went ahead and installed the 3.15 kernel right away.  It may be that
driver support (for example for the various new hotkeys on this laptop) has
improved from 3.13.  If you would like to do that too, just download the amd64
headers and image packages from the [Ubuntu mainline kernel PPA](http://kernel.ubuntu.com/~kernel-ppa/mainline/) and install them by passing both files to
`sudo dpkg -i` and then reboot.

## Issues

There is one minor issue that needs to be addressed once you boot up -- your
machine likely needs a BIOS update in order for it to suspend and resume.
Lenovo have addressed a bug in the BIOS and the updated BIOS ensures that
suspend/resume works in Linux, the fix is in version 1.13 so you will need that
version or newer.

One major issue for me is this laptop's keyboard and touchapd.  I love the
ThinkPad TouchPoint device (the little red nub) and Lenovo have effectively
ruined it with this new design by removing the three TouchPoint physical
buttons and drawing them on the touchpad instead.  The 'nub' is useless without
the corresponding buttons.  I suspect that their driver for Windows solves this
by detecting presses on the top of the touchpad and generating the right
events.  There's nothing like that for xorg at this time and even if there was,
I doubt it would be intuitive to use, especially since the touchpad "clicks",
causing the cursor to jump.  I begrudgingly gave up on using the TouchPoint
with this laptop.

Finally, the keyboard: Lenovo have come up with a completely new keyboard for
this model and it's a bit jarring, especially for a software developer.  The
Backspace and Delete keys are now side by side (this causes me to hit Delete
when I meant Backspace), the Home and End keys are inexplicably moved to where
Caps Lock used to be, and the tilde key is now next to the right-hand Alt key,
which makes absolutely no sense (Escape is where it used to be).  There's
likely nothing really "wrong" with these decisions for the average user but
they are a bit annoying for a developer used to a more standard layout.  I will
try to adjust.

### Updating the BIOS to fix suspend/resume

To update the BIOS:

1. Go to the [Lenovo support site](http://support.lenovo.com/en_US/) and enter
   your machine type in the "quick path" search box.  Mine for example is
   "20A7", written on the bottom of the laptop.  Press "Drivers & Software"
   from the "Select Task" box on the right and then change "Category" to
   "BIOS".  This will let you download a "BIOS Update Bootable CD" -- grab the
   newest one offered.  At this time for my machine this was version 1.14.
2. You will need to use the `geteltorito.pl` perl script ([more information
   here](http://forums.lenovo.com/t5/Linux-Discussion/SUPPORT-REQUEST-X220-BIOS-UPDATE-INSTRUCTIONS-USB/td-p/532077)) to extract the UEFI
   bootable image from the ISO file you downloaded.  For example, if your file
   is called "foo.iso", you can:


        wget http://www.uni-koblenz.de/~krienke/ftp/noarch/geteltorito/geteltorito.pl
        perl geteltorito.pl foo.iso > biosimage.iso


   The resulting ISO image needs to be written to a USB disk.  Insert a disk
   that you can erase and unmount it.  Then use `dd` to copy the ISO
   image to it.  For example, if your USB disk is drive `/dev/sdb`,

        sudo umount /dev/sdb1
        sudo dd if=biosimage.img of=/dev/sdb
        sudo eject /dev/sdb

   should work.
3. Now reboot the laptop and use F12 to select the USB disk as the boot disk.
   You should see a gray screen with options to read about the update and to
   reflash the BIOS (option 2).  Make sure that power is plugged in and your
   battery is reasonably full (you do not want to lose power during the
   reflash) and proceed with option 2.  Follow the onscreen instructions and
   eventually the laptop will reboot a few times as it updates each firmware
   image.  You will then boot back in to Ubuntu and now suspend and resume
   should work.

## Interesting features

The "adaptive" keyboard on this laptop is quite simple, the top row (where the
F1 through F12 keys would be) is now a capacitive touch panel with OLED glyphs
in place of keys.  The "Fn" button on the top left switches between F1 through
F12 plus a few special keys and your volume and brightness adjustment buttons.
The implementation is quite good: it looks nice in person, the lighting is done
really well, and the touch sensitivity is good.  There are some special keys
which don't map to anything in Ubuntu but I really don't care (I suppose that
a working RF kill button would be nice though, unfortunately it doesn't do
anything at this time).  By the way, a double-tapping the left Shift turns it
in to a Caps Lock key, lighting up a green LED on that key to indicate Caps
Lock state.

One positive change in this new layout is the placement of the left Control
key.  It's now on the left corner of the keyboard as it should be and it's
quite large (older ThinkPads usually have the Fn key there and I'd press it
by accident when reaching for Control).

The keyboard is backlit rather than lit from the screen via a ThinkLight.  The
backlight level is controlled by the second-from-right capacitive button when
in F1 through F12 mode and works just fine at night.

The X1 Carbon has a gigabit Ethernet interface, the RJ45 jack is on a dongle
provided with the laptop.  It works just fine out of the box.

## Nice features

I ordered this laptop with the 2560x1440 IPS panel and it is excellent.  The
matte screen is pleasant to look at, there is no glare and I don't find my eyes
straining like the do with the MacBook Pro retina screen.  The Intel graphics
card of course has excellent Linux and xorg support.  The Intel WiFi and
Bluetooth work fine out of the box.

The laptop's performance in general is excellent: the SSD is very fast, boot
time feel very quick, and suspend/resume is nearly instantaneous.  It compiles
the Linux kernel and other large projects about as quickly as a MacBook Pro,
which is my main benchmark.

The built-in camera is quite good and the speakers are decent as well, a
welcome change from the ones they put in previous X series laptops.

The keyboard feel, to me, is quite good, especially compared with the MacBook
Pro keyboard.  It is comfortable to type on aside from the strange layout.
