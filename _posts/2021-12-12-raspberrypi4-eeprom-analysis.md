---
layout: posts
title:  "Raspberry Pi4 VC4 Bootloader Analysis"
date:   2021-12-12 00:00:00 -0600
categories: raspberrypi videocore4 bootloader
author: Nicholas Starke
---

I recently took a look at reverse engineering the Raspberry Pi4 bootloader. This is the proprietary binary blob that is flashed to EEPROM on Raspberry Pi 4.  It controls the initial boot process for the Raspberry Pi4, before the ARM processor starts executing.  The binary blob is code written for the VideoCoreIV (henceforth referred to as "VC4") processor, which is the GPU for the Raspberry Pi4.  

## tl;dr
It is a mystery: The functions for the network protocols and capabilities have been removed in the latest version of the Raspberry Pi4 firmware, but when this version of firmware is flashed to a Pi4, those capabilities appear to still be available when run.  

## Binary Analysis

The binary blob files for the bootloader are stored in a [repository on Github](https://github.com/raspberrypi/rpi-eeprom). I used Ghidra and Bindiff for most of the analysis.  There is no official VC4 Ghidra Language Definition, so I used one proposed in a [Github Pull Request that was never merged](https://github.com/NationalSecurityAgency/ghidra/pull/1147).  I compared the output from this Ghidra Language Def with [my disassembly](https://github.com/nstarke/raspberrypi4-bootloader-analysis) from [vc4-toolchain](https://github.com/itszor/vc4-toolchain) and they were the same in the instances I checked.

![Ghidra VC4 settings](/images/12122021/ghidra-vc4-settings.png "Ghidra VC4 Settings Screenshot")

The proper load address is `0x80000000` and the file offset is `0x8`.

In performing this analysis I was keenly interested in two things:

* Network Capabilities and protocol implementations (IPv4, ARP, DHCP, TFTP, etc)
* The recently introduced Secure Boot implementation

This blog post will focus on what I learned while looking at the network capabilities.  I will return to the Secure Boot implementation in future blog posts.

## Entropy Analysis

Entropy Analysis for `pieeprom-2021-07-06.bin`

![pieeprom-2021-07-06.bin](/images/12122021/pieeprom-2021-07-06.bin.png "pieeprom-2021-07-06.bin entropy screenshot")

Entropy Analysis for `pieeprom-2021-11-22.bin`

![pieeprom-2021-11-22.bin](/images/12122021/pieeprom-2021-11-22.bin.png "pieeprom-2021-11-22.bin entropy screenshot")

These images look similar enough to suggest there are no enormous structural differences between the two images.  Specifically what I was concerned about was whether or not some of or all of the firmware files were encrypted.  For reference, this is what random data (output from `/dev/urandom`) looks like from an entropy analysis perspective:

![test.bin](/images/12122021/test.png "test.bin random data entropy analysis")

## Setting up UART

On the test Raspberry Pi4, I added the following two lines to `/boot/config.txt` in order to enable UART during ARM-core based code execution:

```
enable_uart=1
uart_2ndstage=1
```

Then I created a `bootconf.txt` file that had the following contents:

```
[all]
BOOT_UART=1
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0
BOOT_ORDER=0x2
```

`BOOT_ORDER=0x2` enables TFTP boot with a fall back to the primary eMMC boot.
`BOOT_UART=1` enables debug console output during this first stage vc4-based code exection.

I applied this `bootconf.txt` file to a copy of `pieeprom-2021-11-22.bin` using `rpi-eeprom-config`, then flashed the resulting eeprom binary file to the EEPROM using the `rpi-eeprom-update`.  Then I connected a UART cable to the UART GPIO pins and rebooted.  I then had debug output via UART when booting up:

```
RPi: BOOTLOADER release VERSION:3c912e10 DATE: 2021/11/22 TIME: 11:23:32 BOOTMODE: 0x00000006 part: 0 BUILD_TIMESTAMP=1637580212 0x6e87a417 0x00b03111 0x00050f94
PM_RSTS: 0x00001020
part 00000000 reset_info 00000000
uSD voltage 3.3V
Initialising SDRAM 'Micron' 16Gb x1 total-size: 16 Gbit 3200

XHCI-STOP
xHC ver: 256 HCS: 05000420 fc000031 00e70004 HCC: 002841eb
USBSTS 11
xHC ver: 256 HCS: 05000420 fc000031 00e70004 HCC: 002841eb
xHC ports 5 slots 32 intrs 4
Reset USB port-power 1000 ms
xhci_set_port_power 1 0
xhci_set_port_power 2 0
xhci_set_port_power 3 0
xhci_set_port_power 4 0
xhci_set_port_power 5 0
xhci_set_port_power 1 1
xhci_set_port_power 2 1
xhci_set_port_power 3 1
xhci_set_port_power 4 1
xhci_set_port_power 5 1
XHCI-STOP
xHC ver: 256 HCS: 05000420 fc000031 00e70004 HCC: 002841eb
USBSTS 18
XHCI-STOP
xHC ver: 256 HCS: 05000420 fc000031 00e70004 HCC: 002841eb
USBSTS 19
xHC ver: 256 HCS: 05000420 fc000031 00e70004 HCC: 002841eb
xHC ports 5 slots 32 intrs 4
Boot mode: NETWORK (02) order 0
GENET: RESET_PHY
PHY ID 600d 84a2
NET_BOOT: dc:a6:32:09:f8:c4 wait for link TFTP: 0.0.0.0
Stopping network
RX: 0 IP: 0 IPV4: 0 MAC: 0 UDP: 0 UDP RECV: 0 IP_CSUM_ERR: 0 UDP_CSUM_ERR: 0
GENET STOP: 0
NETBOOT CANCEL
NETBOOT init failed
Boot mode: SD (00) order 0
Insert SD-CARD
[...]
```

We can verify that we flashed the right version of `pieeprom` by checking for the string `VERSION:3c912e10`:

```
$ grep "VERSION:3c912e10" pieeprom-2021-11-22.bin
Binary file pieeprom-2021-11-22.bin matches
```

Further, the date from the log output match up with the date in the filename.

A little more tinkering with my DHCP options and I had the test RPI4 attempting to netboot over TFTP.

But that isn't where this story gets interesting...

## Bindiff Analysis

I did two bindiff comparisons:

* `pieeprom-2021-07-06.bin` vs `pieeprom-2021-11-22.bin`
* `pieeprom-2021-04-29.bin` vs `pieeprom-2021-07-06.bin`

![Bindiff Overview](/images/12122021/bindiff-overview.png "Bindiff Overview Screenshot")

As you can see from the above screenshot, `pieeprom-2021-07-06.bin` identified 798 functions and `pieeprom-2021-11-22.bin` only identified 254.  This is quite a discrepancy when `pieeprom-2021-04-29.bin` identified 686 functions.  Why are there many less functions in `pieeprom-2021-11-22.bin` than the other two versions?

I identified a few functions in `pieeprom-2021-07-06.bin` related to TFTP (function naming mine):

* `tftp_get`: 0x80015b18
* `tftp_recv`: 0x80015c68
* `tftp_timeout`: 0x80015ab0

![Bindiff Primary](/images/12122021/bindiff-primary.png "Bindiff Primary Screenshot")

It turns out that the functions responsible for handling network capabilities do not exist in `pieeprom-2021-11-22.bin`.  I confirmed this using Bindiff's Primary Unmatched functions for functions in `pieeprom-2021-07-06.bin` that were not in `pieeprom-2021-11-22.bin` (see above screenshot). A quick spot check on this result can be achieved by runnings strings for `tftp` against both versions of `pieeprom`:

```
$ strings pieeprom-2021-07-06.bin | grep -i tftp
   tftp: %i %e
NET %i %i gw %i tftp %i
NET_BOOT: %e wait for link TFTP: %i
TFTP%d
TFTP: disconnect: timeouts %d
TFTP_GET: %e %i %s
TFTP %d: %s
TFTP: complete %u
TFTP_FILE_TIMEOUT
TFTP_PREFIX
TFTP_PREFIX_STR
TFTP_IP
```

```
$ strings pieeprom-2021-11-22.bin | grep -i tftp
TFTP_FILE_TIMEOUT
TFTP_PREFIX
TFTP_PREFIX_STR
TFTP_IP
0tftp
nk TFTP6
TFTP%d)
```

Some of the logging messages, such as `NET_BOOT: %e wait for link TFTP: %i` are not present in the `pieeprom-2021-11-22.bin` file.  However, during the boot cycle I can definitely see this error message for tftpboot:

```
[...]
Boot mode: NETWORK (02) order 0
GENET: RESET_PHY
PHY ID 600d 84a2
NET_BOOT: dc:a6:32:09:f8:c4 wait for link TFTP: 0.0.0.0
LINK STATUS: speed: 1000 full duplex
Link ready
GENET START: 64 16 32
GENET: UMAC_START 0xdca63209 0xf8c40000
RX: 0 IP: 0 IPV4: 0 MAC: 0 UDP: 0 UDP RECV: 0 IP_CSUM_ERR: 0 UDP_CSUM_ERR: 0
DHCP src: ac:1f:6b:66:01:83 192.168.20.1
YI_ADDR 192.168.20.100
OPTIONS:-
        op: 53 len:   1 DHCP recv OFFER (2) expect OFFER
        op: 54 len:   4 192.168.20.1
        op: 51 len:   4
        op:  1 len:   4 255.255.255.0
        op:  3 len:   4 192.168.20.1
        op: 66 len:  14 192.168.20.105[66]: 192.168.20.105

DHCP src: ac:1f:6b:66:01:83 192.168.20.1
YI_ADDR 192.168.20.100
OPTIONS:-
        op: 53 len:   1 DHCP recv ACK (5) expect ACK
        op: 54 len:   4 192.168.20.1
        op: 51 len:   4
        op:  1 len:   4 255.255.255.0
        op:  3 len:   4 192.168.20.1
        op:  6 len:   4
        op: 66 len:  14 192.168.20.105[66]: 192.168.20.105

        op: 15 len:  23
NET 192.168.20.100 255.255.255.0 gw 0.0.0.0 tftp 192.168.20.105
ARP 192.168.20.105 dc:a6:32:1b:1d:c6
NET 192.168.20.100 255.255.255.0 gw 0.0.0.0 tftp 192.168.20.105
RX: 13 IP: 0 IPV4: 4 MAC: 4 UDP: 3 UDP RECV: 2 IP_CSUM_ERR: 0 UDP_CSUM_ERR: 0
TFTP_GET: dc:a6:32:1b:1d:c6 192.168.20.105 6e87a417/start4.elf
[...]
```

This leaves me **very** confused: if those functions/strings are not present in the `pieeprom-2021-11-22.bin` firmware, where is the Raspberry Pi4 retrieving that code from during boot?  

If you have any ideas, please send me a DM on [twitter](https://twitter.com/nstarke).