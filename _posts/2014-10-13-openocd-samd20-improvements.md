---
layout: post
title: Improvements and new commands for Atmel SAMD20 in openocd
---

The openocd folks have merged a couple of my patches that deal with Atmel
SAMD2x series MCUs (SAMD20, SAMD21, the cheaper SAMD10 and SAMD11, etc.), they
are:

1. [fix protect, add EEPROM and boot commands](http://openocd.zylin.com/#/c/2326/)
2. [add erase/secure commands](http://openocd.zylin.com/#/c/2239/), this one also ensures that the NVM cache is disabled when we perform NVM operations in openocd.

You can now issue a full chip erase of your SAMD MCU, even without halting (as
I mentioned, openocd sometimes cannot halt the target and this is one way to
recover).  To do that, issue:

    at91samd chip-erase

This will not erase the User Page or the bootloader section (if any).  You can
also enable flash security on your MCU.  Please only do this if you know what
to expect and be aware that at this time openocd will not be able to un-secure
the chip by erasing it.  The reason is that even the DAP `IDCODE` will not be
readable via SWD once a chip is secured and openocd cannot handle that (Atmel
provides a tool in Atmel Studio that can do this).  If you really want to try,
run:

    at91samd set-security

and follow the instructions in the error message.  The security setting takes
effect on MCU reset.

The Flash protect feature does not persist on reset unless it is also set in
the User Page and furthermore I had a bug in the flash driver where the wrong
sector was protected or unprotected -- that is all fixed now.  You can also
modify the EEPROM size and bootloader size setting in the User Page (please see
the datasheet for details).

For example, to check the EEPROM size:

    at91samd eeprom

Provide a valid EEPROM size in bytes to set it, for example:

    at91samd eeprom 1024

or set it to 0 to disable EEPROM emulation.  Similarly the bootloader section
size (called the BOOTPROT region in the datasheet) can be inspected:

    at91samd bootloader

and set, for example:

    at91samd bootloader 16384

or disabled by setting a size of 0.  The EEPROM and bootloader settings take
effect on MCU reset and the target must be halted in order to write them.

The User Page is technically 256 bytes and only the first 64 bits of it are
used by the MCU.  Also those bits can be used to configure the watchdog timer
hardware on reset as well as the brownout detector (I did not add support for
that but it is easy to plumb in if needed).  Support for writing custom things
to the remaining bits in the page could be added as well, for example that is a
good place to store your device serial number and other manufacturing
information.
