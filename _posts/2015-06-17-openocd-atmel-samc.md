---
layout: post
title: openocd support for Atmel SAMC 5.0V family
---

Atmel have released a [5.0V MCU family called SAMC](http://www.atmel.com/products/microcontrollers/ARM/SAM-C.aspx) which is equivalent to the
normal [SAMD family](http://www.atmel.com/products/microcontrollers/arm/sam-d.aspx) (there are SAMC20 and SAMC21 sets of parts). The Flash
controller and other pertinent components are nearly identical so I submitted
[this patch for openocd](http://openocd.zylin.com/#/c/2809/) which should add
SAMC support to the existing SAMD driver.

This also adds config files for the
SAMC Xplained Pro kits, `atmel_samc20_xplained_pro.cfg` and
`atmel_samc21_xplained_pro.cfg`.  These can be used directly or adapted as
needed for your application. Speaking of which, what is the application for a
5.0V I/O family these days? Is it something legacy-compatible or perhaps industrial?  I would love to know what folks are using these for outside of the hobby market.

Please give this patch a try if SAMC is interesting to you (I do not have the
hardware to test with) and let me know if things work.  This openocd Flash
driver should support all of the M0+ parts including SAMD, SAMR, SAML, and
SAMC now.
