---
layout: post
title: Sitara AM335x EDMA regression fix for 3.13, 3.14, 3.15 Linux kernels
---

I had a frustrating problem with the TI Sitara AM335x at work that first
cropped up as non-working wl12xx WiFi on kernels newer than 3.12 (incidentally,
3.12 is the last kernel on which TI "supports" the wl12xx, they are now telling
customers to use wl18xx, though in this case it will not make a difference).

I hooked up the platform data glue for the wlcore SDIO module in the new way
introduced in 3.13 -- add your board to the table in `arch/arm/mach-omap2/pdata-quirks.c` and have your routine call `legacy_init_wl12xx()` with the
appropriate clock and IRQ pin setting.  See, for example, the `omap3_evm_legacy_init()` routine.  With that and the right information in our device tree (DTS)
file, the wlcore driver found our card and began to initialize it.

Unfortunately SDIO then stalled, and it did so on its first attempt to use
DMA -- turning on `CONFIG_MMC_DEBUG` showed that `CMD52` worked fine (single
register accesses) but then it came time to use `CMD53` to perform a block
transfer and this never returned because the DMA callback never triggered.

As it turned out our board places the WiFi interface on the MMC2 bus (the
third controller on Sitara, called `mmc3` in Device Tree as they are numbered
starting with 1 rather than 0).  `mmc1` and `mmc2` have direct DMA channels
but `mmc3` requires the use of a cross-bar or "xbar" switch.  In our case we
mapped Event 1 and Event 2 to channels 12 and 13 like this:

    &edma {
            ti,edma-xbar-event-map = <1 12
                                      2 13>;
    };

The DMA linkage in `&mmc3` then looks like:

    dmas = <&edma 12
            &edma 13>;
    dma-names = "tx", "rx";

The code responsible for looking at `ti,edma-xbar-event-map` and actually
configuring these mappings is found in `arch/arm/common/edma.c` and this is
where I finally found a problem.  The function
`edma_of_read_u32_to_s16_array()` is supposed to read out the device tree data
and it does so in the 3.12 kernel, terminating the entries with a `-1` to
indicate the end of the list.  In 3.13 someone cleaned up this admittedly
sloppy-looking routine and unfortunately terminated the list at the zeroeth
entry unconditionally.

The solution is as simple as replacing the implementation of  `edma_of_read_u32_to_s16_array()` with the one from the 3.12 kernel.  However I noticed that
the (at the time of writing this) in-development 3.16 kernel contains a [more
thorough and cleaner fix](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/arch/arm/common/edma.c?id=cf7eb979116c2568e8bc3b6a7269c7a359864ace) by Thomas Gleixner so users of 3.16 and beyond will not have to worry
about this.

In the end we have wl12xx (or wl18xx for that matter) WiFi working on 3.13
through 3.15 kernels now, and it should continue to work beyond that as well.
