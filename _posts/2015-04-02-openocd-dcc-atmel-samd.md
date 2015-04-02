---
layout: post
title: Using the DCC as a debug console on Atmel SAMD MCUs
---

I found a nice piece of debug hardware on Atmel's SAMD line of Cortex M0+ microcontrollers.  It's a simple implementation of a Debug Communication Channel (DCC) as part of Atmel's Device Service Unit (DSU).  They provide two 32-bit DCCs which makes it easy to implement a bidirectional debug "console" without using UART or other hardware.  This has a few advantages:

- no UART wiring or pins used, just SWD which you're already using for debugging.
- the DCC implements basic flow control like a polled UART
- the DCC is available even when the part is secured so you can carefully provide some basic debug functionality even on a shipping product.

See section 12.11.4 in the Atmel SAMD20 datasheet for more details.

I submitted [an implementation for openocd](http://openocd.zylin.com/#/c/2692/) that provides a TCP socket server to "host" the DCC and control of the two channels.  This should be enough to implement a simple UART-style console in your SAMD firmware though you can use it for tracing and other functionality as well.

If you'd like to try this out, apply that patch and then decide on a scheme for using the DCC.  Here's what I did for example:

- DCC0 is the MCU's receiver (the debugger writes there)
- DCC1 is the MCU's transmitter (the debugger reads from there)
- we deal with bytes (characters) even though each DCC is 32 bits wide.

In openocd, I then issue:

    at91samd dcc server enable
    at91samd dcc channel 0 write
    at91samd dcc channel 1 read

And this starts a TCP server on port 22000, polls and writes to DCC0, and polls and reads from DCC1.  I then telnet to that port and I have my "console":

    telnet localhost 22000

On the firmware side, we need to check the DCC "dirty" bits in the DSU STATUSB
register.  The idea is:

- For our read channel (DCC0) we should read from the DCC when the dirty bit is set.  This means that the debugger has written data and we haven't read it out yet (the bit is cleared on read).
- For our write channel (DCC1), we can write when the dirty bit is clear.  This means that the debugger has read the previous data and we won't clobber anything (again, the bit is cleared on read, this time by the debugger reading).

Here's a snippet for "read",

    #include "dsu.h" /* part of the ASF */

    if (DSU->STATUSB.bit.DCCD0) {
        uint32_t data = DSU->DCC[0].reg;
        /* We have the character the debugger wrote in 'data' now. */
    }

From there the character in `data' could be handled to a console parser or placed in some kind of receive FIFO.  You could also implement the standard C library
`_read()' stub to pull from that FIFO.

And here's your "write":

    #include "dsu.h" /* part of the ASF */

    /* We are waiting to write and the DCC is ready */
    if (have_data_to_write() && !DSU->STATUSB.bit.DCCD1) {
        DSU->DCC[1].reg = data_to_write();
    }

The dummy `have_data_to_write' and `data_to_write' methods can be implemented, for example as part of a simple FIFO that buffers up outgoing characters and pulls them from the FIFO when the DCC is available.  Your standard C library `_write()' stub implementation would then simply place characters in the FIFO and let a DCC task (or background/idle hook) do the above IO transfer.  This would enable you to simply `printf()' into the DCC.
