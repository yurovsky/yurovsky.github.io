---
layout: post
title: Using spidev with the Linux kernel device tree
---

I recently hit a `WARN_ON` when adding a spidev device to a device tree, the
warning is:

    buggy DT: spidev listed directly in DT

and it was introduced by [this patch](http://www.spinics.net/lists/linux-spi/msg03301.html). This is quite unfortunate but easy to work around, though I would
argue that it's counter to the point of the `spidev` driver to begin with: after
all we're describing hardware that we're talking to from user space and there's
no need to describe it in the device tree, this just creates extra work.

## Solution 1: the proper way

To avoid this warning you will need to add your device to the `spidev_dt_ids`
table in `drivers/spi/spidev.c` and then use that device string (and not literally `spidev` in your device tree `compatible`).  For example you could add
a device named `foocorp,modem` to the end of that table:

	--- a/drivers/spi/spidev.c
	+++ b/drivers/spi/spidev.c
	@@ -695,6 +695,7 @@
	 static const struct of_device_id spidev_dt_ids[] = {
		{ .compatible = "rohm,dh2228fv" },
		{ .compatible = "lineartechnology,ltc2488" },
	+	{ .compatible = "foocorp,modem" },
		{},
	 };
	 MODULE_DEVICE_TABLE(of, spidev_dt_ids);

And then your device tree entry would contain a node with the corresponding
compatible string:

	compatible = "foocorp,modem";

When the `spidev` module is probed, things should match up and you will see
a corresponding character device as before.

### Problems with this

This has some obvious problems including:

 * We now have to patch the kernel to add a string to some code for no apparent
   reason other than "correctness" of the device tree.
 * This patch is probably not going upstream because it's unlikely that anyone
   else has this hardware (though in some cases it may go upstream).
 * Maintaining this patch is not going to be pleasant: you're adding an entry
   to a table and rebasing will take some work once people add more things to
   that table.

## Solution 2: quick and dirty

You can forgo patching the driver and simply claim one of the already included
devices from the table. It's not going to make any difference: `spidev` will
match that string and move on.  So for example I don't have a "rohm,dh2228fv"
on my board so I could have:

	comptabile = "rohm,dh2228fv"; /* actually my foocorp modem */

And things will work fine and I won't see the nasty `WARN_ON` anymore.

A nicer future change would be to move this `spidev_dt_ids` table out to the
device tree itself and avoid this altogether but no one has done that yet (at
least as of the 4.8 kernel).  A cursory glance through `arm/boot/dts` shows
many users of `spidev` that will now be seeing the `WARN_ON` message.
