---
layout: post
title: Developing STM8 boot code with SDCC
---

I'm using the open source [SDCC](http://sdcc.sourceforge.net/) toolchain to
develop an application for the STM8
microcontroller and part of that requires a custom bootloader (what ST's
manuals refer to as User Boot Code or UBC) and application firmware. Here are
some notes on how to use SDCC and [stm8flash](https://github.com/vdudouyt/stm8flash) to develop and flash the bootloader and application.

The UBC concept itself is mostly a convention on STM8. The hardware does not
do much with it aside from treating the UBC area of flash as write-protected
(the idea is that boot code is not field-upgradeable in a typical product
whereas we may wish to reflash the application firmware).

# Boot process and interrupts

The STM8 uses option byte 1 to determine the size of the UBC (it's 0 by default
meaning there is no boot code). Setting this to a non-zero size reserves a
portion of flash (starting at `0x8000`) for the UBC. For example setting byte 1
to 4 reserves four 256 byte blocks or 1KB for the UBC.

It is up to the boot code to jump to the application. The STM8 CPU assumes that
the interrupt vector table is located at `0x8000` so this single table must be
shared between the UBC and the application.

## Interrupt vector table

In SDCC, interrupt vectors look like:

    void usart1_rx_irq(void) __interrupt(28)
    {
    }

Interrupt vectors must be implemented in the same translation unit (file) that
implements `main()` and the `__interrupt` attribute is used to specify their
IRQ number (which becomes an offset into the interrupt vector table).

The interrupt vector table is placed at the start of the program (`0x8000` by
default, or whatever is set by using the `--code-loc` option). The UBC will be
placed at `0x8000` along with its vector table but the application needs to be
placed after the UBC (starting with its own vector table). So for a 1KB UBC
(that is, option byte 1 is set to 4) we would build the application firmware
with `--code-loc=0x8400` and we know that the interrupt vector table for the
application is at `0x8400`.

The STM8 manual shows the offsets from the start of the vector table for each
interrupt handler. For the above interrupt 28, the offset is 0x78. Assuming the
boot code does not need to do anything with interrupt 28, we could simply
redirect to the application firmware's implementation of that interrupt handler.
That is, in the UBC we would have something like:

    void usart1_rx_irq(void) __interrupt(28)
    {
        __asm jpf 0x8478 __endasm;
    }

To connect the real interrupt 28 to the redirected handler in the application's
table. The application's implementation of interrupt 28 would do whatever it
is that is appropriate for handling that interrupt.

The UBC should redirect every single interrupt to the right location to provide
equivalent functionality in the application. The table itself contains 4-byte
entries:

* 0x00: the reset vector
* 0x04: trap handler
* 0x08: interrupt 0 (unused)
* 0x0C: interrupt 1 (FLASH)
* 0x10: interrupt 2 (DMA 0/1)

...and so on. As such, interrupt 28 is `4 * 28 + 8` or `120` which gives us the
offset `0x78` and, if the application starts at `0x8400` the redirected vector
is at `0x8478`.

We can calculate some offsets and addresses in code (or the preprocessor) to
make life easier.

# stm8flash

To flash the MCU:

* get the current option bytes content from the MCU
* modify that content to set the UBC size (and any other changes needed)
* write back the modified option bytes
* write the bootloader
* write the application

Using the STM8 discovery board (with STM8L151), we can read the option bytes
from the MCU,

    stm8flash -c stlink -p stm8l151?6 -s opt -r opt.bin

Then edit `opt.bin` as needed. Note that byte 0 must always be set to `0xAA` to
keep the SWIM protocol usable. To write it back:

    stm8flash -c stlink -p stm8l151?6 -s opt -w opt.bin

To write the bootloader, `boot.ihx` to the default location (0x8000):

    stm8flash -c stlink -p stm8l151?6 -w boot.ihx

And then to write the application to (for example) 0x8400:

    stm8flash -c stlink -p stm8l151?6 -s 0x8400 -w fw.ihx
