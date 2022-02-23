---
layout: posts
title:  "HP Firmware DXE Extraction"
date:   2022-02-22 00:00:00 -0600
categories: hp firnmware dxe extractopm
author: Nicholas Starke
---

I recently analyzed some HP Laptop firmware using UEFITool, and one thing I noticed in newer firmware images is that there are no results for DXE modules listed in UEFITool. After a lot of messing around, I figured out that there is a Firmware Volume (FV) LZMA compressed in the UEFI image file. I'm not sure why UEFITool doesn't recognize this FV.  I needed a way to examine DXE modules though, so I wrote a small python3 script that takes a file path to an HP UEFI firmware image file and dumps the DXE modules to a directory in `$PWD` called `volume-0`.

I used a combination of `binwalk` and `uefi_firmware` to perform the extraction.  It is important to note that in the segment that `binwalk` extracts, the UEFI PI FV begins at 16 (0x10) bytes into the segment.

Without further adieu:

```python
#!/usr/bin/env python3
#
# Author: Nicholas Starke
# Date: 2022-02-22
#
# To install necessary dependencies, run:
# $ pip3 install uefi_firmware binwalk
#

import uefi_firmware, binwalk, sys, os

print("[*] Beginning HP Firmware DXE Extraction.")
for module in binwalk.scan(sys.argv[1], signature=True, quiet=True, extract=True):
    for result in module.results:
        if "LZMA" in result.description:
            try:
                lzma_file = open(module.extractor.output[result.file.path].extracted[result.offset].files[0], 'rb').read()
            except:
                print("[-] An error occured and extraction failed.  Bummer.")
                continue

            parser = uefi_firmware.AutoParser(lzma_file[0x10:])
            parsed_lzma_file = parser.parse()
            parsed_lzma_file.dump()
            print("[+] Success! Extracted DXE files should be in directory 'volume-0'")
            os.rmdir(module.extractor.directory)
            print("[*] Cleaned up binwalk extraction contents. Exiting now")
            sys.exit()

    os.rmdir(module.extractor.directory)
    print("[-] Extraction failed.")
```