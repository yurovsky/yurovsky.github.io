---
layout: post
title: Reflashing and debugging Atmel SAML21 with openocd
---

I submitted [this simple patch](http://openocd.zylin.com/#/c/2690/) adding support for the new super low power Atmel SAML21 Cortex M0+ MCU family.  The flash controller is nearly identical to the one in SAMD20 and related parts so it's a trivial change aside from having to fix a couple of bit shifting mistakes that I hadn't caught until now.

If you'd like to try this out, apply that patch and rebuild and install openocd and then plug in your SAML21 Xplained Pro board.  You can make an openocd.cfg file that contains:

    source [find board/atmel_saml21_xplained_pro.cfg]

And then fire up openocd.  You should then be able to debug normally with GDB, relfash the MCU, etc.  There are some NVM (flash) improvements on the L21 that I'll look at later, for one thing they've implemented hardware EEPROM emulation and that might need to be taken into account in the openocd driver.
