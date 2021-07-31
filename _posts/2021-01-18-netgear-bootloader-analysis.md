---
layout: posts
title:  "Netgear Bootloader Analysis"
date:   2021-01-18 00:00:00 -0600
categories: netgear u-boot bootloader
author: Nicholas Starke
---

This is the first part in a two part blog post discussing the stage 1 and stage 2 bootloaders for a specific netgear device.  This first post will describe how I loaded the bootloader into ghidra, and the second post will document more extensively what I found.

I decided recently to look into the stage 1 and stage 2 boot loaders that the Netgear R9000 (x10) router utilizes during the boot process.  This bootloader binary does not come bundled with the upgrade firmware and seems to be burned onto SPI ROM at manufacture.  If you are interested in following along, the SPI flash can be dumped from an operating system shell using the following command:

```
$ cat /dev/mtd0 > /tmp/mtd0.bin
$ cd /tmp
$ tftp -p -l mtd0.bin $TFTP_SERVER_IP
```

Of course, getting an operating system shell is not a trivial task and is left as an exercize for the reader.  

My goal is to identify the binary code for the stage 2 bootloader, then load it into Ghidra.  Now that we have `mtd0.bin`, we run binwalk on the blob:

```

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
458824        0x70048         uImage header, header size: 64 bytes, header CRC: 0x6E51D086, created: 2016-07-22 05:32:10, image size: 77 bytes, Data Address: 0x0, Entry Point: 0x0, data CRC: 0x3379A803, OS: Firmware, CPU: ARM, image type: Script file, compression type: none, image name: "init_script"
524360        0x80048         Flattened device tree, size: 17996 bytes, version: 17
815700        0xC7254         CRC32 polynomial table, little endian
872762        0xD513A         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/iofic/al_hal_iofic.c
873806        0xD554E         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/udma/al_hal_udma_config.c
874549        0xD5835         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/udma/al_hal_udma_iofic.c
878344        0xD6708         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/include/udma/al_hal_udma.h
878920        0xD6948         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/eth/al_hal_eth_main.c
880620        0xD6FEC         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/eth/al_hal_eth_kr.c
881132        0xD71EC         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/ssm/al_hal_ssm_raid.c
883555        0xD7B63         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pcie/al_hal_pcie.c
886603        0xD874B         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pcie/al_hal_pcie_interrupts.c
886825        0xD8829         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/ddr/al_hal_ddr.c
887367        0xD8A47         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/ddr/al_hal_ddr_pmu.c
887611        0xD8B3B         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_muio_mux.c
888107        0xD8D2B         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_spi.c
888767        0xD8FBF         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_nand_dma.c
888932        0xD9064         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_bootstrap.c
889766        0xD93A6         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_i2c.c
890312        0xD95C8         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_addr_map.c
890612        0xD96F4         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/pbs/al_hal_tdm.c
890870        0xD97F6         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/ring/al_hal_pll.c
891316        0xD99B4         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/sys_services/al_hal_timer.c
891994        0xD9C5A         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/sys_fabric/al_hal_iommu.c
892486        0xD9E46         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/drivers/ring/al_hal_cmos.c
894653        0xDA6BD         Unix path: /home/ericwang/git-home/u-boot.git/tools/al_boot_v_1_65_1/src/HAL/services/pcie/al_init_pcie.c
911381        0xDE815         Copyright string: "copyright."
949496        0xE7CF8         Flattened device tree, size: 4445 bytes, version: 17

```

The first line of the binwalk output shows a uImage header with a type of "script" that is 77 bytes long.  Due to its short length, we can be sure this is not the bootloader we are interested in.  The additional binwalk results seem to show Unix paths that indicate u-boot is used as the stage 2 bootloader. 

Running `strings` on `mtd0.bin` reveals that the board is a Anna Purna Labs Alpine board as well as the version of U-Boot:

```
[...]
 ;Annapurna Labs Alpine Dev Board
[...]
U-Boot 2015.01-gd836bbb (Jul 22 2016 - 13:32:00)
[...]
```

This U-Boot version string seems to suggest the bootloader was compiled back in 2016, and not updated since.  This leads me to believe the bootloader is not ever, or rarely, updated in the field.  

I begin reviewing `mtd0.bin` using the `xxd` binary, and it seems like there are several components embedded in the blob that are separated by blocks of `0xff` bytes.  I adapted a script I wrote a while back to split binary data on null characters:

```python
#!/usr/bin/env python3

# The idea behind this file is that many times firmware images are constructed with zero'd out address regions as the delimiter.
# This script will split files in a firmware image when the embedded files are $MAX zero'd bytes in distance from each other.

import sys

FILE=sys.argv[1]
MAX=512

print ("Analyzing File: " + FILE)

with open(FILE, 'rb') as f:
    counter = 0
    data = f.read()
    start = 0
    tripped = False
    for byte in range(len(data)):
        if data[byte] == 0xff:
            if not tripped:
                counter = counter + 1
                if counter == MAX:
                    tripped = True
                    with open('splitter-%08d-%08d.bin' % (start, byte - start - MAX + 1), 'wb') as wf:
                        wf.write(data[start:(byte - MAX + 1)])
                        wf.close()
                        print('wrote file. start: %08d. length was %08d' % (start, byte - start - MAX + 1))
                        start = byte + 1
                        counter = 0
            else:
                start = byte + 1
        else:
            tripped = False
            counter = 0
    f.close()

```

This split the binary blob up in to a bunch of smaller subdivisions.  I grepped for the Unix path containing `u-boot` from the binwalk results, and ended up utilizing the last chunk that the binary split script generated.  This file was named `splitter-00589824-00450316.bin`.  This was the file I ended up loading into Ghidra for analysis.

The first 256 bytes of this file look like this:

```
  00000000: c79e 0b00 0000 0000 0500 0000 0000 0000  ................
  00000010: 0000 0000 0000 0000 4e2f 4100 0000 0000  ........N/A.....
  00000020: 0000 0000 0000 0000 c0de 0600 0000 0000  ................
  00000030: 0000 1000 0000 0000 0000 1000 0000 0000  ................
  00000040: 0000 0000 f703 0000 be00 00ea 14f0 9fe5  ................
  00000050: 14f0 9fe5 14f0 9fe5 14f0 9fe5 14f0 9fe5  ................
  00000060: 14f0 9fe5 14f0 9fe5 6000 1000 c000 1000  ........`.......
  00000070: 2001 1000 8001 1000 e001 1000 4002 1000   ...........@...
  00000080: a002 1000 efbe adde dec0 ad0b 00f0 20e3  .............. .
  00000090: 00f0 20e3 00f0 20e3 00f0 20e3 00f0 20e3  .. ... ... ... .
  000000a0: 00f0 20e3 00f0 20e3 28d0 1fe5 00e0 8de5  .. ... .(.......
  000000b0: 00e0 4fe1 04e0 8de5 13d0 a0e3 0df0 69e1  ..O...........i.
  000000c0: 0fe0 a0e1 0ef0 b0e1 48d0 4de2 ff1f 8de8  ........H.M.....
  000000d0: 5020 1fe5 0c00 92e8 4800 8de2 3450 8de2  P ......H...4P..
  000000e0: 0e10 a0e1 0f00 85e8 0d00 a0e1 1702 00fa  ................
  000000f0: 00f0 20e3 00f0 20e3 00f0 20e3 00f0 20e3  .. ... ... ... .
```


Now that I have the bootloader binary identified, I need to figure out the load address (`CONFIG_SYS_TEXT_BASE`) for the u-boot image.  I don't have console output from the boot process, but I found a post on netgear's community forum where someone had pasted their NAS boot output.  The board seemed similar.

```
[...]
Stage 3 2013.10-alpine_spl-1.49.d-00659-ge082932 (Nov 19 2014 - 16:21:12)

DRAM: 2 GiB
EEPROM Revision ID = 32
Device ID = a212
Device Info: AL21200-1400
Loading DT to 00100000 (16813 bytes)...
Board config ID: Netgear NAS RN20x
SRAM agent up: agent_wakeup v1.49
Loading U-Boot to 00100000 (401888 bytes)...
Executing U-Boot...


U-Boot 2013.10-alpine_db-1.49 (Jan 08 2016 - 10:41:23)
[...]
```

Source: [https://community.netgear.com/t5/Using-your-ReadyNAS-in-Business/RN204-can-t-start/td-p/1280284](https://community.netgear.com/t5/Using-your-ReadyNAS-in-Business/RN204-can-t-start/td-p/1280284)

The important output is `Loading U-Boot to 00100000`.  The u-boot version and alpine board revision are slightly older, but its as good a guess as any for a load address.  I used the value `0x00100000` as the load address in Ghidra, with a File Offset value of `0x48` since the first 72 bytes of the split image turned out to be a header.  Note that this is a proprietary header and thus binwalk doesn't recognize it.

Using these values and selecting a CPU Architecture of ARMv7 32-bit default little-endian allowed me to successfully load the bootloader in Ghidra.

Once I had the bootloader loaded in Ghidra, I went looking for console output signifier strings. One thing I honed in on pretty quickly was the string `NTGR`.  Like I mentioned above, I don't have a console set up for this device, but I'm guessing that if you type that string in during the boot process you can break into some sort of undocumented u-boot shell.  The community has seen similar "magic strings" on TP-Link devices, using the `tpl` string.  The disassembly between the code hosting the `tpl` string from the TP-Link bootloader does not look comparable to the code hosting the `NTGR` string, and I will have to do further analysis to prove this `NTGR` string is what I think it is.  

Other interesting tidbits I gleaned were that the U-Boot image was compiled with support for Secure Boot.  Secure Boot is definitely not enabled on this device, however.  These seemns to be the capability to set a "passphrase" for the board, though it is unclear whether or not such a feature is utilized.  

Stay Tuned for Part 2.