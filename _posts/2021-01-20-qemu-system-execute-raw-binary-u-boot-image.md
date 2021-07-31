---
layout: post
title:  "How to Execute Raw U-Boot Bootloader Binary with QEMU"
date:   2021-01-20 00:00:00 -0600
categories: qemu u-boot bootloader
author: Nicholas Starke
---

When working with U-Boot images, it might be desireable to execute an image in QEMU.  If you have compiled U-Boot yourself, you will notice there are two different "types" of u-boot images compiled:

* `u-boot` - Has an elf header
* `u-boot.bin` - Is a raw binary without an elf header.

The first file (`u-boot`) is fairly easy to virtualize with QEMU since QEMU has excellent support for ELF binaries (like the Linux kernel). What if you wanted to execute the second file (`u-boot.bin`) though?

First thing you must be sure of is that QEMU has support for the board that your U Boot image is built to run on.  For testing purposes, I selected `vexpress_ca9x4_defconfig` from the `configs` directory and built a U Boot image for that board type. Once I had the image compiled, I successfully virtualized the raw binary version of the U Boot image using this QEMU command:

```
$ qemu-system-arm -M vexpress-a9 -serial stdio -device loader,file=u-boot.bin,addr=0x60800000,cpu-num=0,force-raw=on
```

Console output:

```
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument


U-Boot 2015.01-dirty (Jan 24 2021 - 14:48:15)

DRAM:  128 MiB
WARNING: Caches not enabled
Flash: 128 MiB
MMC:   MMC: 0
*** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Net:   smc911x-0
Warning: smc911x-0 using MAC address from net device

Warning: Your board does not use generic board. Please read
doc/README.generic-board and take action. Boards not
upgraded by the late 2014 may break or be removed.
Hit any key to stop autoboot:  0 
Wrong Image Format for bootm command
ERROR: can't get kernel image!
VExpress# 
```

From there you can add NICs using `-nic` or adjust the memory size with `-m`.