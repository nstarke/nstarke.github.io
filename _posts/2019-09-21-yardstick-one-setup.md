---
layout: posts
title:  "Yardstick One Setup"
date:   2019-09-21 00:00:00 -0600
categories: sdr yardstick-one
author: Nicholas Starke
---

A few years ago I bought a YardStick One from Great Scott Gadgets ([https://greatscottgadgets.com/yardstickone/](https://greatscottgadgets.com/yardstickone/)).

YardStick One works with a software suite called `rfcat` ([https://github.com/atlas0fd00m/rfcat](https://github.com/atlas0fd00m/rfcat)). I needed to update the bootloader firmware for my YardStick One to work with recent versions of `rfcat`.

Because of a compiler issue between `sdcc` 3.6.0.0 and 3.8.0.0 (latest as of this writing), when I attempted to flash the bootloader firmware, I received an `Invalid IOCTL` warning:

```
Could not configure port: (25, 'Inappropriate ioctl for device')
```

I had to use a custom build of the firmware, compiled against `sdcc` 3.6.0.0, in order for the YardStick One to work properly.

You can compile sdcc yourself at version 3.6.0.0 and then compile the bootloader in the `rfcat` source, or you can use the firmware here: [https://gist.github.com/mossmann/7b816680df2ac513df3835f3cb9eaa1b](https://gist.github.com/mossmann/7b816680df2ac513df3835f3cb9eaa1b)

Hopefully this saves someone some time.