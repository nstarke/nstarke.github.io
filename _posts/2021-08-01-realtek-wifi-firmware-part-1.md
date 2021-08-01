---
layout: posts
title:  "Realtek Wifi Firmware: Part 1"
date:   2021-08-01 00:00:00 -0600
categories: firmware realtek wifi linux kernel
author: Nicholas Starke
---

## Firmware Header

In [https://8051enthusiast.github.io/2021/07/05/002-wifi_fun.html](https://8051enthusiast.github.io/2021/07/05/002-wifi_fun.html), the author mentions this:

>It turns out there is a 0x20 byte header

I was curious how the author came to this conclusion.  We'll look at `rtl8188efw.bin` which is loaded and serviced by `rtl8188ee.ko`.  The source code for this driver can be found here: [https://github.com/torvalds/linux/tree/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee](https://github.com/torvalds/linux/tree/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee).  Particularly interesting is the c file that handles loading and unloading the firmware: [https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee/fw.c](https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee/fw.c).

Looking at the first 0x40 bytes of `rtl8188efw.bin` we see this:

```
00000000: e188 1000 0800 0000 1025 2156 b02b 0000  .........%!V.+..
00000010: a204 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0245 3500 0000 0000 0000 0000 0000 0000  .E5.............
00000030: 0000 00c1 5600 0000 0000 0000 0000 0000  ....V...........
```

The Ghidra disassembly for offset 0x20 looks like this:

```
//
// CODE 
// CODE:4000-CODE:6bcf
//
CODE:4000 02  45  35       LJMP       LAB_CODE_4535
```

Which looks like valid 8051 disassembly for the beginning of a binary image. So it _looks_ like this is the correct offset, But how do we _confirm_ 0x20 is the correct offset?

It turns out the realtek wifi firmware header structure is actually defined in the Linux kernel: [https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/wifi.h#L234](https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/wifi.h#L234)

```c++
struct rtlwifi_firmware_header {
	__le16 signature;
	u8 category;
	u8 function;
	__le16 version;
	u8 subversion;
	u8 rsvd1;
	u8 month;
	u8 date;
	u8 hour;
	u8 minute;
	__le16 ramcodesize;
	__le16 rsvd2;
	__le32 svnindex;
	__le32 rsvd3;
	__le32 rsvd4;
	__le32 rsvd5;
};
```

The first two bytes are a signature.  In the image we are considering, this is `0xe188`, which is the 16-bit little endian value `0x881e`.  This signature is verified by the linux kernel driver here: [https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee/fw.h#L15](https://github.com/torvalds/linux/blob/v5.13/drivers/net/wireless/realtek/rtlwifi/rtl8188ee/fw.h#L15).

Other field values include:
* `category` - 0x10 - 1 Byte
* `function` - 0x00 - 1 Byte
* `version` - 0x0008 - 2 Bytes
* `subversion` - 0x00 - 1 Byte
* `rsvd1` - 0x00 - 1 Byte
* `month` - 0x10 - 1 Byte
* `date` - 0x25 - 1 Byte
* `hour` - 0x21 - 1 Byte
* `minute` - 0x56 - 1 Byte
* `ramcodesize` - 0x2bb0 - 2 Bytes
* `rsvd2` - 0x0000 - 2 Bytes
* `svnindex` - 0x04a20000 - 4 Bytes
* `rsvd3` - 0x00000000 - 4 Bytes
* `rsvd4` - 0x00000000 - 4 Bytes
* `rsvd5` - 0x00000000 - 4 Bytes

Some interesting notes:

What `month` is the hex value 0x10? Perhaps this should be interpretted as decimal even though its embedded in binary format.  The same is true for the `minute` value 0x56 - this would be the decimal value 86 which is more than the number of minutes in an hour. Ditto for the `date` - 0x25 is way past the maximum month date value of 31.

`ramcodesize` matches the length of the firmware file minus the header size:

```
>>> hex(11216 - 32)
'0x2bb0'
```

However, to answer our original question, by adding up the size of each field's date type, we see that the size of this struct is 32 bytes or 0x20 in hex.  

```
>>> hex(2 + 1 + 1 + 2 + 1 + 1 + 1 + 1 + 1 + 1 + 2 + 2 + 4 + 4 + 4 + 4)
'0x20'
```

Also, I do not see anything in the header which looks like a checksum.

## at51 Results

The [at51](https://github.com/8051Enthusiast/at51) base results are as follows:

```
$ at51 base rtl8188efw.bin
Index by likeliness:
        1:  0x3fe0 with 139
        2:  0x2526 with 63
        3:  0x64f with 58
```

We take the first result and add 0x20 for the header size to it to find the base address of 0x4000.

The libfind results for this firmware file are located here: [https://github.com/nstarke/at51-libfind-linux-firmware-results/blob/main/results/rtlwifi/rtl8188efw.bin.results.txt](https://github.com/nstarke/at51-libfind-linux-firmware-results/blob/main/results/rtlwifi/rtl8188efw.bin.results.txt).  Specifically, `at51 libfind` reports the `?C_START` address as `0x593`. 

If we subtract 0x20 from 0x593, we get 0x573.  Add 0x4000 for the firmware base address and we receive 0x4573 which is the second jump in the binary:

![RTL Firmware Graph](/images/0057/top-graph.PNG "RTL Firmware Graph Screenshot")

## Taking it further

I wonder what would happen if we increased the `ramcodesize` field in the header on a host with one of these cards installed?  Fortunately, at least on Ubuntu, the `/lib/firmware` files are only writable as root:

```
$ ls -lah /lib/firmware/rtlwifi/rtl8188efw.bin
-rw-r--r-- 1 root root 11K Jun 11 15:19 /lib/firmware/rtlwifi/rtl8188efw.bin
```

I currently don't have access to the necessary hardware to perform this experiment, but would be interested to play around with it.