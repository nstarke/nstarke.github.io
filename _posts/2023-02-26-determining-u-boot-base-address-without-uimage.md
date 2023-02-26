---
layout: posts
title: "Determining U-Boot Base Address without uImage"
date: 2023-02-26 00:00:00 -0600
categories: u-boot ghidra
author: Nicholas Starke
---

## Introduction

I recently pulled a `u-boot.bin` U-Boot image off of an embedded device.  The file had no uImage header; it was just a raw binary file. Many times with loading a raw binary file into Ghidra, figuring out the base address is a challenge if it is not provided in a header or configuration file.  That was exactly the case in this instance; all I had to go off of was the `u-boot.bin` file.  The base address was not listed or documented anywhere else.  

The following is a little guide on how to figure out the Base Address to load U-Boot at in Ghidra in order to resolve symbols properly for ease of reverse engineering.

![](/images/02262023/ghidra.png)

In this specific firmware file, the U-Boot Base Address is `0x49fb0000`.  We are able to determine that by looking at the double word at offset `0x40` into the binary file.  

```
00000000: ea00 0014 e59f f014 e59f f014 e59f f014  ................
00000010: e59f f014 e59f f014 e59f f014 e59f f014  ................
00000020: 49fb 01c0 49fb 0220 49fb 02a0 49fb 0320  I...I.. I...I..
00000030: 49fb 03a0 49fb 0420 49fb 04a0 1234 5678  I...I.. I....4Vx
00000040: 49fb 0000 49fb 0000 0000 0000 0000 0000  I...I...........
00000050: 0002 afac 0003 8594 e3a0 2c1f e3a0 30ff  ..........,...0.
00000060: e582 3000 e10f 0000 e3c0 001f e380 00d3  ..0.............
```

This can be determined from looking at `include/image.h` in the [U-Boot Source](https://source.denx.de/u-boot/u-boot/-/blob/master/include/image.h#L316):

```c
/*
 * Legacy format image header,
 * all data in network byte order (aka natural aka bigendian).
 */
struct legacy_img_hdr {
	uint32_t	ih_magic;	/* Image Header Magic Number	*/
	uint32_t	ih_hcrc;	/* Image Header CRC Checksum	*/
	uint32_t	ih_time;	/* Image Creation Timestamp	*/
	uint32_t	ih_size;	/* Image Data Size		*/
	uint32_t	ih_load;	/* Data	 Load  Address		*/
	uint32_t	ih_ep;		/* Entry Point Address		*/
	uint32_t	ih_dcrc;	/* Image Data CRC Checksum	*/
	uint8_t		ih_os;		/* Operating System		*/
	uint8_t		ih_arch;	/* CPU architecture		*/
	uint8_t		ih_type;	/* Image Type			*/
	uint8_t		ih_comp;	/* Compression Type		*/
	uint8_t		ih_name[IH_NMLEN];	/* Image Name		*/
};

struct image_info {
	ulong		start, end;		/* start/end of blob */
	ulong		image_start, image_len; /* start of image within blob, len of image */
	ulong		load;			/* load addr for the image */
	uint8_t		comp, type, os;		/* compression, type of image, os type */
	uint8_t		arch;			/* CPU architecture */
};
```

Looking at the ghidra screen shot, we can see that `image_len` is `0x2afac` (starting at offset 0x50), and thus by working backwards we can see that `image_start` and `end` are both the value 0 and that `start` is `0x49fb0000`.

So by looking at either offset 0x44 or 0x40 we can reliably determine the u-boot load address. 

