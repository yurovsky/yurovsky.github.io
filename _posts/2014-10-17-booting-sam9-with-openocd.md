---
layout: post
title: Bringing up and booting an Atmel SAM9 with openocd
---

The [boot process](http://www.at91.com/linux4sam/bin/view/Linux4SAM/GettingStarted#Boot_sequence) on this SoC family is generally:

1. ROM bootstrap loader runs and tries to find a suitable bootstrap program to
   load into internal SRAM.  If that fails (ex: there is no bootstrap
   programmed yet), the ROM bootstrap sits in a monitor program, intended to
   be used with the SAM-BA tool from Atmel.
2. The bootstrap program is executed from SRAM.  It sets up various hardware,
   including external DDR RAM.  It then copies the next application (ex: u-boot
   or the Linux kernel itself) into DDR RAM and jumps there.

A fresh board can be set up by running u-boot from RAM and in turn using that
to transfer over the Linux kernel and other components and then write them to
the NAND Flash (or whatever your boot source is).  The `at91bootstrap` program,
however, must run first in order to set up DDR and other hardware.

This can be achieved with openocd by:

1. Resetting and halting the target.
2. Loading and executing the bootstrap program.  This would normally load
   u-boot from Flash but the Flash is erased.
3. Stopping the bootstrap program right before it jumps to the application
   (with DDR now set up).
4. Loading u-boot into the right address.
5. Letting the target continue and thereby jump in to u-boot.

This process can be automated but here is an interactive way to run through it.

I assume that you have a serial interface connected to the `DBGU` port on the
SoC, that is where serial output from bootloaders is typically sent.  The
configuration for this is the typical 115200 8N1.

## at91bootstrap

First, you will need to build the `at91bootstrap` program for your specific
target (or whatever development kit is closest to your production hardware).

Grab the bootstrap program and configure it, for example:

    git clone git://github.com/linux4sam/at91bootstrap.git
    cd at91bootstrap
    make mrproper

    make at91sam9x5eknf_uboot_defconfig
    
    make CROSS_COMPILE=arm-none-linux-gnueabi-

The output will be found in `binaries/`, specifically the ELF file can be used
with GDB and the bin file can be written to Flash.  Find the appropriate
`_defconfig` file in `board/<name>/` and note that `nf` means "NAND Flash" and
the `uboot` configs try to load u-boot whereas the the image configs try to
load a Linux kernel directly.  I used a configuration for the sam9x5 evaluation
kits with NAND Flash and u-boot.  You can customize further, after setting the
defconfig by running:

    make menuconfig

and then building the binaries.

## openocd

Tell openocd what JTAG interface you are using and what board you are
targetting.  In my case the `at91sam9rl-ek` configuration was close enough to
the one I am actually using and my debugger is an Atmel-branded J-Link, my
`openocd.cfg` file looks like:

    source [find interface/jlink.cfg]
    source [find board/atmel_at91sam9rl-ek.cfg]

Make sure you know where your SRAM is located and how large it is (this varies
between the various SAM9 sub-families) and verify that the openocd board and
target scripts match up.  You may have to edit or create your own scripts if
things do not look right for your particular chip.

Start openocd with your configuration file and make sure that it is able to
find the CPU and report the number of breakpoints.  Make sure you see something
like:

    Info : Embedded ICE version 6
    Info : at91sam9rl.cpu: hardware has 2 breakpoint/watchpoint units

in your openocd output before proceeding.

## GDB

We can now use GDB to start the bootstrap program.  Tell GDB to connect and
immediately halt the target and let it know what ELF file you would like to
use:

    arm-none-linux-gnueabi-gdb -ex "target remote localhost:3333" -ex "mon reset halt" binaries/at91sam9x5ek-nandflashboot-uboot-3.7.elf

From the GDB console, we can enable DCC downloads:

    mon mon arm7_9 fast_memory_access enable

And now tell GDB to load the program into RAM:

    load

We need a breakpoint right at the end of `main()`, check main.c for the line
number, for example:

    b main.c:170

Now let the program run and hit the breakpoint:

    c

At this point the bootstrap has completed hardware setup and is ready to jump
to the u-boot address, which in my case is `0x26f00000`.

Now we just need `u-boot.bin` loaded at `0x26f00000`.  We can use `load_image`
in openocd to do this:

    mon load_image /tftpboot/u-boot.bin 0x26f00000 bin

and then continue:

    c

The u-boot console should now appear on `DBGU`, press Enter to start
interactive mode.

## u-boot

From here you can use u-boot to transfer files into RAM (via Ethernet or USB or
MMC or even the serial port).  You could also continue to use JTAG to transfer
images into RAM (openocd `load_image`) and then use u-boot to write them to
Flash.

For example, we can interrupt the target in GDB by pressing control+c and then
load a uImage into RAM at `0x22000000`:

    mon load_image /tftpboot/uImage 0x22000000 bin
    c

Then tell u-boot to run this via its console:

    bootm 0x22000000

or, similarly, you could have u-boot write the image to NAND Flash or whatever
is needed.  See [the example memory map](http://www.at91.com/linux4sam/bin/view/Linux4SAM/GettingStarted#Linux4SAM_NandFlash_demo_Memory) for ideas on where to
put things in NAND Flash.  Given the typical [boot sequence](http://www.at91.com/linux4sam/bin/view/Linux4SAM/GettingStarted#Boot_sequence), I would start by
writing the appropriate `at91bootstrap` program into Flash, writing u-boot into
Flash as well, and then seeing if a reset of the board gets you the expected
results.
