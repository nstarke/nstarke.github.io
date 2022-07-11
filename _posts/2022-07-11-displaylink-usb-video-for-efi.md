---
layout: posts
title:  "DisplayLink USB Video Card for EFI"
date:   2022-07-11 00:00:00 -0600
categories: efi video
author: Nicholas Starke
---

Let's say you have a x86/x64 based host that boots using EFI, but you do not have any sort of video out on the host.  Let's also say you need video out on the host, you know, so you can play [Doom](https://github.com/Cacodemon345/uefidoom) on it. This short blog post will document the hardware and techniques I used to do something along these lines.

## DisplayLink

There exists a EFI DXE driver for DisplayLink USB Video Card chipsets.  [From the official sources](https://github.com/tianocore/edk2-platforms/tree/master/Drivers/DisplayLink/DisplayLinkPkg):

>This package contains a GOP driver for Universal USB-connected docks containing the DisplayLink DL-6xxx chip or newer.

The very important part of this statement is `DisplayLink DL-6xxx chip or newer`.  DisplayLink has been around since at least 2010 if not longer (I have a **NL57AA** USB to DVI adapter from Jun 2010 lying around somewhere), but there are not many commercially available USB cards that use a DL-6xxx+ chipset.  I did manage to find a single card that used a DL-6950 chipset that was also USB based:

[https://www.amazon.com/dp/B094G5JLNC](https://www.amazon.com/dp/B094G5JLNC) 

**Note: this is not an affiliate link and I do not make any money off of referrals if you buy one**

![DisplayLink USB Video Card](/images/07112022/usb-video-card.jpeg "DisplayLink USB Video Card")

## GOP Driver

Now you can build the DisplayLinkPkg DXE GOP Driver using edk2 with edk2-stdlib.  Successful compilation results in **DisplayLinkGop.efi** outputted to the **BUILD** directory.  The resultant EFI driver is 18-19KB in size (depending on whether or not your build type is DEBUG or RELEASE). This driver can be loaded from an EFI shell like this:

```
load DisplayLinkGop.efi
```

Loading this driver may attempt to load PXE drivers for all attached network interfaces, but it should also start mirroring the EFI shell to whatever monitor is connected to the USB Video Card.  And from there you can play Doom, or do whatever else you need to do.

## Where is this useful?

The main utility I see this providing is for a UEFI-based system that has no graphical output whatsoever, maybe not even serial output easily accessible.  Think something like a laptop with a dead monitor that also won't boot into the host OS.  In that case, if you can control the media the host boots from, crafting a **startup.nsh** file that loads the **DisplayLinkGop.efi** DXE driver can result in UEFI shell access for such a system if it has a USB port. It is entirely possible OEMs are including this DXE driver in their EFI builds, especially in the case of laptops, so really the above information is only useful if you need to load an EFI Shell (or other EFI application) manually.  