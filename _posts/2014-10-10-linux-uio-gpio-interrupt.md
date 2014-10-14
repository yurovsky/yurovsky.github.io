---
layout: post
title: Handling GPIO interrupts in userspace on Linux with UIO
---

The Linux kernel provides a userspace IO subsystem (UIO) which enables some
types of drivers to be written almost entirely in userspace (see [basic documentation here](https://www.kernel.org/doc/htmldocs/uio-howto/). This is done by
via a character device that the user program can open, memory map, and
perform IO operations with. This device can also be used to block for
interrupts.

## Driver

This provides a nice and fairly low-latency interface for handling a GPIO
interrupt in userspace. In my case, I needed a userspace program to talk to
SPI (via the `spidev` module) and handle interrupts as well, so UIO seemed more
fitting than, say, a gpio-keys input events approach.

The kernel provides a `uio_pdrv_genirq` module which uses the core `uio`
framework to handle generic IRQs. Build your kernel with:

    CONFIG_UIO=m
    CONFIG_UIO_PDRV_GENIRQ=m

## Device Tree

There is not much device tree documentation on this topic but here is what I
have figured out. Please correct me if any of this is the wrong approach!

At a minimum, your device tree will need a node for your UIO device. It will
have to match `generic-uio` and `ui_pdrv` in order to be used with the generic
IRQ driver. I created a node named `user_io` and threw in my own name, say,
`mydevice` into the compabible string as well:

    user_io@0 {
        compatible = "mydevice,generic-uio,ui_pdrv";
        status = "okay";
    };

### GPIO Details

That said, for a GPIO interrupt we need an interrupt parent (GPIO port) and
pin. In this example I used an Atmel SAM9 SoC and its Port A pin 13. My
device asserts an active-high interrupt:

    user_io@0 {
        compatible = "mydevice,generic-uio,ui_pdrv";
        status = "okay";
        interrupt-parent = <&pioA>;
        interrupts = <13 IRQ_TYPE_EDGE_RISING>;
    };

in the above, the interrupt parent is the GPIO port node in question and the
interrupt configuration tells it to use pin 13 on that port.  The interrupt
level definitions can be found in the kernel tree in
`include/dt-bindings/interrupt-controller/irq.h` if you need more detail.
Typically you need a level interrupt (ex: `IRQ_TYPE_LEVEL_HIGH`) when your
device pulses the pin rather than just changing its level.  An edge trigger is
more common, in this case the device just changes the pin level and holds it
at the new level until the interrupt is acknowledged or otherwise cleared.

This UIO device can also be tired to pinmux control, for example to set (or
unset) a pull device. In my case I needed pin A13 to be pulled low so I added a
`pinctrl` node for it with SAM9-specific initialization:

    ahb {
        apb {
            pinctrl@fffff400 {
                user_io {
                    pinctrl_user_io: user_io-0 {
                        atmel,pins = <AT91_PIOA 13 AT91_GPIO AT91_PINCTRL_PULL_DOWN>;
                    };
                };
            };
        };
    };

I then referenced it in the `user_io` node:

    user_io@0 {
        compatible = "mydevice,generic-uio,ui_pdrv";
        status = "okay";
        interrupt-parent = <&pioA>;
        interrupts = <13 IRQ_TYPE_EDGE_RISING>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_user_io>;
    };

Take a look in `/sys/kernel/debug/pinctrl` if you need to troubleshoot pin
control issues.

## Userspace

With the above device tree compiled and given to the kernel, load the generic
IRQ driver with the following parameter:

    modprobe uio_pdrv_genirq of_id="mydevice,generic-uio,ui_pdrv"

This tells UI to match the `compatible` string given by the `of_id` so adjust
yours accordingly. You should see `/dev/uio0` created.

A userspace program can use the UI device node as follows:

1. `open()` the device node in read-write (`O_RDWR`)
2. `write()` to the device to unmask the interrupt
3. `read()` from the device to block until an interrupt arrives. You can also use `select()` or `poll()` or whatever blocking method you prefer. Go back to step 2 to handle the next interrupt.  A `select()` or `poll()` call is much more efficient than a simple `read()` due to the way the UIO event handling is implemented internally.
4. `close()` the device to mask off the interrupt and clean up.

All reads and writes are handled by transferring exactly a 32-bit unsigned
integer (that is, 4 bytes).

Keep in mind that you can look at `/proc/interrupts` to check the number of
interrupt counts for the GPIO line in question.  This lets you know if the
interrupt is happening at all or whether an IRQ storm has happened.  The
counter increments for every interrupt event.

### Example

The interrupt starts masked and the user must explicitly unmask it. This is
done by writing a 1 (again, four bytes) to the device. The interrupt is masked
once again after reading. For example, consider this very trivial program:

    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <stdint.h>

    int main(void)
    {
        int fd = open("/dev/uio0", O_RDWR);
        if (fd < 0) {
            perror("open");
            exit(EXIT_FAILURE);
        }

        while (1) { /* some condition here */
            uint32_t info = 1; /* unmask */

            ssize_t nb = write(fd, &info, sizeof(info));
            if (nb < sizeof(info)) {
                perror("write");
                close(fd);
                exit(EXIT_FAILURE);
            }

            /* Wait for interrupt */
            nb = read(fd, &info, sizeof(info));
            if (nb == sizeof(info)) {
                /* Do something in response to the interrupt. */
                printf("Interrupt #%u!\n", info);
            }
        }

        close(fd);
        exit(EXIT_SUCCESS);
    }

The value that you will read back is the interrupt count (a running counter of
the number of interrupts generated). You can of course open the device in
non-blocking mode (with `O_NONBLOCK`) if needed.

### Example with poll()

A better approach may be to use `poll(3)` on the file descriptor:

    #include <stdio.h>
    #include <stdint.h>
    #include <stdlib.h>
    #include <poll.h>
    #include <fcntl.h>
    #include <errno.h>

    int main(void)
    {
        int fd = open("/dev/uio0", O_RDWR);
        if (fd < 0) {
            perror("open");
            exit(EXIT_FAILURE);
        }

        while (1) {
            uint32_t info = 1; /* unmask */

            ssize_t nb = write(fd, &info, sizeof(info));
            if (nb < sizeof(info)) {
                perror("write");
                close(fd);
                exit(EXIT_FAILURE);
            }

            struct pollfd fds = {
                .fd = fd,
                .events = POLLIN,
            };

            int ret = poll(&fds, 1, -1);
            if (ret >= 1) {
                nb = read(fd, &info, sizeof(info));
                if (nb == sizeof(info)) {
                    /* Do something in response to the interrupt. */
                    printf("Interrupt #%u!\n", info);
                }
            } else {
                perror("poll()");
                close(fd);
                exit(EXIT_FAILURE);
            }
        }

        close(fd);
        exit(EXIT_SUCCESS);
    }

passing -1 as the third argument to `poll()` requests polling without a timeout
(`poll()` would return 0 in a timeout case otherwise). The return value is
then either the number of descriptors ready or -1 to indicate an error (in
which case `errno` is set). One potential error case is that `poll()` was
interrupted by a signal, in this case `errno` is set to `EINTR` and this may or
may not need to be treated as an actual error depending on how your program is
designed.
