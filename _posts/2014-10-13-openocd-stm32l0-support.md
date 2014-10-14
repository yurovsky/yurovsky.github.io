---
layout: post
title: Support for STM32L0 in openocd
---

openocd supports the new STM32L0 family of very low-power Cortex-M0+ parts from
ST.  The common dev kits for them are the Discovery and Nucleo boards which can
both use the `stm32l0discovery.cfg` file introduced in [this commit](http://openocd.zylin.com/#/c/2317/).  Your `openocd.cfg` can just be:

    source [find board/stm32l0discovery.cfg]

This pulls in the STLink-v2.1 driver as well the the relevant target settings.

I refactored the STM32Lx flash driver a while back -- [this commit](http://openocd.zylin.com/#/c/2200/) so you can reflash your STM32L0 in openocd like you
would expect.  The STM32L0 flash controller is very similar to the STM32L1 but
has a few geometry differences.
