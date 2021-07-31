---
layout: posts
title:  "AT51 Libfind Results - Keil Toolchain"
date:   2021-07-31 00:00:00 -0600
categories: 8051 at51 linux-firmware cpu_rec
author: Nicholas Starke
---

# Background

The [linux-firmware package](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git) contains a number of binary firmware images that are based on 8051-chipsets. I ran [cpu_rec](https://github.com/airbus-seclab/cpu_rec) against all the binaries in `linux-firmware` and posted a list of results [here](https://gist.github.com/nstarke/771f76801e92e5c46508a9a61888920d).  I took this list and ran [at51 libfind](https://github.com/8051Enthusiast/at51) against the 8051-reported results.  I used the Keil 8051 toolchain files with `at51 libfind` to develop [these results](https://github.com/nstarke/at51-libfind-linux-firmware-results).  

# Results

This results in an interesting observation: some 8051-based firmware blobs do not have any results when matched against the Keil toolchain, meaning they were most likely compiled with an alternative toolchain.  The list of these 8051 blob files in linux-firmware are:

```
./rt3071.bin
./rt73.bin
./usbdux_firmware.bin
./usbduxfast_firmware.bin
./keyspan_pda/xircom_pgs.fw
./dabusb/firmware.fw
./hfi1_dc8051.fw
./qlogic/sd7220.fw
./microchip/mscc_vsc8574_revb_int8051_29e8.bin
./microchip/mscc_vsc8584_revb_int8051_fb48.bin
./rt2870.bin
./ene-ub6250/ms_rdwr.bin
./ene-ub6250/sd_rdwr.bin
./ene-ub6250/sd_init1.bin
./ene-ub6250/msp_rdwr.bin
./ene-ub6250/ms_init.bin
./ene-ub6250/sd_init2.bin
./usbduxsigma_firmware.bin
./edgeport/down2.fw
./edgeport/down.fw
```