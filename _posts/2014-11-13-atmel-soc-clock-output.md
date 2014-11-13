---
layout: post
title: Programmable clock output on Atmel SoCs in Linux
---

Atmel ARM SoCs like the SAM9 and SAMA5D3 have programmable clocks that can be
output on particular pins.  These are often used to provide clock sources to
peripherals like audio codecs.  The following things need to happen in order to
get one of these clocks routed somewhere on your board:

1. Configure the pinmux accordingly.
2. Reference the corresponding clock and pinmux entry.
3. Request and enable that clock.

For example, I needed PCK0 on the SAM9G25 SoC.  This can be routed to a few
places, for example Port C pin 15, peripheral function C.  In device tree, the
pinmux entry for this can look like:

    pck0 {
        pinctrl_pck0: pck-0 {
            atmel,pins = <AT91_PIOC 15 AT91_PERIPH_C AT91_PINCTRL_NONE>;
        };
    };

Now we need a device tree node that will reference that clock.  Atmel provides
the `pck0` definition already so we can create a custom node and then a trivial
driver that will match its `compatible` string:

    wclock {
                compatible = "wclock";
                clocks = <&pck0>;
                clock-names = "slowclock";
                pinctrl-0 = <&pinctrl_pck0>;
                pinctrl-names = "default";
    };

This node references `pck0` as well as our `pinctrl_pck0` pinmux configuration.
Now we just need someone to match the `wclock` compatible string and ask for
and then enable the clock.  A trivial kernel module that implements this can
look like:

    #include <linux/kernel.h>
    #include <linux/platform_device.h>
    #include <linux/of.h>
    #include <linux/of_platform.h>
    #include <linux/module.h>
    #include <linux/clk.h>

    static struct clk *slowclock;

    static int wclock_probe(struct platform_device *pdev)
    {
        slowclock = devm_clk_get(&pdev->dev, "slowclock");
        if (IS_ERR(slowclock)) {
            dev_err(&pdev->dev, "Failed to get slowclock\n");
            return PTR_ERR(slowclock);
        }

        clk_prepare_enable(slowclock);

        return 0;
    }

    static int wclock_remove(struct platform_device *pdev)
    {
        return 0;
    }

    static const struct of_device_id of_wclock_match[] = {
        { .compatible = "wclock", },
        {},
    };

    MODULE_DEVICE_TABLE(of, of_wclock_match);

    static struct platform_driver wclock_driver = {
        .probe	= wclock_probe,
        .remove	= wclock_remove,
        .driver	= {
            .name		= "wclock",
            .owner		= THIS_MODULE,
            .of_match_table = of_match_ptr(of_wclock_match),
        },
    };

    module_platform_driver(wclock_driver);

    MODULE_AUTHOR("Andrey Yurovsky");
    MODULE_DESCRIPTION("programmable clock dummy driver");
    MODULE_LICENSE("GPL");

When loaded, the driver will ask for the `slowclock` clock and then will tell
the clock subsystem to enable it.  The pinmux configuration will then have
taken effect and your `pinmux-pins` file hook in debugfs should have a line
like:

    pin 79 (pioC15): wclock (GPIO UNCLAIMED) function pck0 group pck-0

reflecting the change.  You can also check the clock enable count and status
in debugfs, see `/sys/kernel/debug/clk/pck0` for the PCK0 status.
