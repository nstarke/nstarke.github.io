---
layout: posts
title:  "Dell Precision 7510 System Failure after Monitor turns off in Ubuntu 16.04 / 17.10 / 18.04"
date:   2018-05-03 00:00:00 -0600
categories: dell bios precision-7510
author: Nicholas Starke
---

I have been experiencing a problem with my Dell Precision 7510 laptop. 
When using Ubuntu, configured to turn the monitor off after n minutes, the computer would become unresponsive if I let it stay "asleep" for longer than a few minutes.  
At this point, when I tried to wake the laptop up by pressing a key or moving the mouse, the computer wouldn't respond at all.  The only option was to restart the computer using a hard stop (pressing the power key for 5 seconds).

After months of problems and troubleshooting, I isolated the problem to the system BIOS, and indeed, according to the Dell Bios Changelog ([link here](https://downloads.dell.com/FOLDER04736861M/1/Precision_7x10_1.15.4.txt?fn=Precision_7x10_1.15.4.txt)) some of the bug fixes mention problems with crashes related to the monitor shutting off.
It is worth noting that I had this problem with BIOS revision 1.15.4 and 1.14.4. BIOS version 1.15.4 is the latest as of this date.

The solution ended up being within the BIOS configuration.  By disabling "Switchable Graphics" within the "Video Settings" in the System BIOS, I was able to run the laptop without any problems.

This is just a note for anyone else out there experiencing the same problems, hopefully this helps solve your issues.