---
layout: posts
title:  "Extracting BIOS Image from Lenovo Update Exe"
date:   2022-08-28 00:00:00 -0600
categories: lenovo uefi
author: Nicholas Starke
---

I recently needed to extract the BIOS/UEFI Image from a Lenovo BIOS Update Exe file. I didn't see any written information on how to extract the BIOS image file from the Lenovo Update EXE file, so I wrote up some instructions on how to do so in case it helps anyone else down the road.

![Lenovo Update](/images/08282022/lenovo-update.png "Lenovo Update")

For example, from [https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/1-series/1-15ijl7/downloads/driver-list/component?name=BIOS%2FUEFI](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/1-series/1-15ijl7/downloads/driver-list/component?name=BIOS%2FUEFI), the BIOS update downloads as **htcn28ww.exe**.  Trying to run this executable will result in an error message on any host that is not the intended host.  But, it is possible to recover the UEFI image by using [innoextract](https://github.com/dscharrer/innoextract) and [7-zip](https://www.7-zip.org/).

The process is simple: download a release version of **innoextract** (or compile it from source yourself!).  Run **innoextract** from a windows Command Prompt:

```
C:\Users\stark\Documents>C:\Users\stark\innoextract-1.9-windows\innoextract.exe htcn28ww.exe
Extracting "Lenovo BIOS Update Utility" - setup data version 5.5.7 (unicode)
 - "code$GetExtractPath\install.bat" - overwritten
 - "code$GetExtractPath\install.bat"
 - "code$GetExtractPath\HTCN28WW.exe"
Done.
```

The resulting files are dumped into a newly created directory, **code$GetExtractPath**.  Navigate to this directory, and use 7-zip to extract **HTCN28WW.exe** by right clicking the file and using the **7-zip** context menu to **Extract here**.  

![Extracted](/images/08282022/extracted.png "Extracted")

The file **BIOS.fd** is the BIOS image file.  It will load properly into [UEFITool](https://github.com/LongSoft/UEFITool) for additional analysis.  

I hope that is helpful to someone else!