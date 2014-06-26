---
layout: post
title: Work around reset/halt issues on Cortex M0 and M0+ with openocd
---

I noticed that I could not halt an Atmel SAMD21 with openocd and recalled that
the part in question was running firmware that entered low power modes on the
MCU, leading to unreliable debugger operation.  I believe that the correct fix
is to work out how to implement the CPU Reset Extension feature and/or some
additional sequences to wake the debug block but I am not entirely sure.

I decided to work around this for now by simply erasing the entire Flash on
the MCU in order to remove the firmware that puts it into low power mode.  To
do this, start up openocd and connect to it with telnet:

    telnet localhost 4444

Write a 0x02 to the PAC1 register (PAC1 is Peripheral Access Control 1) to turn
off write protection for the Device Service Unit (DSU) registers:

    mww 0x41000000 2

Now we can use the DSU control register to set the Chip Erase (CE) bit:

    mwb 0x41002000 0x10

We can then wait for the DONE bit to be set in the ``STATUSA`` register in the
DSU but realistically the chip erase will finish pretty quickly and dumping the
start of Flash is a good enough check.  On SAMD21 that is at address zero so:

    mdw 0 16

Should show all 0xFFFFFFFF values indicating that the chip is erased.  I then
hot-plugged the board and fired up openocd again and confirmed that
``reset halt`` was working.  From there I could use GDB to load and debug a
new firmware.
