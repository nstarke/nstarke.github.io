# Reversing Raw Binary Firmware Files in Ghidra

This brief tutorial will show you how to go about analyzing a raw binary firmware image in Ghidra.

## Prep work in Binwalk

I was recently interested in reversing some older Cisco IOS images.  Those images come in the form of a single binary blob, without any sort of ELF, Mach-o, or PE header to describe the binary.

While I am using Cisco IOS Images in this example, the same process should apply to other Raw Binary Firmware Images.

That makes importing this type of firmware file difficult, as Ghidra doesn't have any idea what type of ISA it needs to disassemble and decompile for.

The following are a few things I learned while trying to get the Cisco IOS image in a reversible state within Ghidra.

First I had to extract the image. I pulled the firmware image off a switch I recently bought over TFTP.  It turns out the first 112 bytes are some sort of Cisco proprietary header that is not useful for our purpose.  We need to extract the bzip2 archive that we are interested in.  The easist way to do that is binwalk:

```bash
binwalk -eM c3750-ipservicesk9-mz.122-50.SE3.bin
```
This will create a `_c3750-ipservicesk9-mz.122-55.SE.bin.extracted` directory which will have a file named `70` inside it. 

Now we need to figure out the CPU ISA.  For this we use binwalk again:

```bash
binwalk -m /usr/local/lib/python2.7/dist-packages/binwalk/magic/binarch _c3750-ipservicesk9-mz.122-55.SE.bin.extracted/70
```

This will output a lot of things, so lets take a look at the output:

```bash
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
24            0x18            PowerPC big endian instructions, function prologue
1360          0x550           PowerPC big endian instructions, function epilogue
1364          0x554           PowerPC big endian instructions, function epilogue
1372          0x55C           PowerPC big endian instructions, function epilogue
1380          0x564           PowerPC big endian instructions, function epilogue
1388          0x56C           PowerPC big endian instructions, function prologue
1612          0x64C           PowerPC big endian instructions, function epilogue
1648          0x670           PowerPC big endian instructions, function epilogue
1656          0x678           PowerPC big endian instructions, function prologue
3224          0xC98           PowerPC big endian instructions, function epilogue
3232          0xCA0           PowerPC big endian instructions, function prologue
6772          0x1A74          PowerPC big endian instructions, function epilogue
6780          0x1A7C          PowerPC big endian instructions, function prologue
[...]
```

We can see that the binary has Big Endian PowerPC function prologues followed by epilogues.  This is a good indicator that the firmware image ISA is PowerPC Big Endian.

Now that we know the ISA, we need to know the text-base offset and the data-base offset within the firmware image. The best way to figure this out is to load the firmware on an actual device and boot up the device.

To retrieve the base address (fileOffset), run the `show version` command on the Cisco Switch:
```bash
Switchy>
Switchy>enable
Password:
Switchy#show version
Cisco IOS Software, C3750 Software (C3750-IPSERVICESK9-M), Version 12.2(55)SE, RELEASE SOFTWARE (fc2)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2010 by Cisco Systems, Inc.
Compiled Sat 07-Aug-10 22:45 by prod_rel_team
Image text-base: 0x01000000, data-base: 0x02F00000

ROM: Bootstrap program is C3750 boot loader
BOOTLDR: C3750 Boot Loader (C3750-HBOOT-M) Version 12.2(44)SE5, RELEASE SOFTWARE (fc1)
[...]
```

This dumps out a line: `Image text-base: 0x01000000, data-base: 0x02F00000`.  
Both of those addresses are important, so note them and save them for later.

# Ghidra Time

Now open the `70` binary blob in ghidra.  Again, since there is no standardized binary format for the binary, you will have to import the file as `Raw Binary`, and then set the Code Architecture to `PowerPC Big Endian 4xx`.  Also, click the options button and set the image offset base to the value we retrieve from the `show version` command: 0x01000000.  Then import.

![Ghidra File Import Options Screenshot](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/e580a6464ce787b20ce3442490a3cc5f647a6a98/ghidra-import-file-options.png)

Ghidra will then churn on the binary for a while, and when it is done the strings should be resolved to labels within the decompiler. This is because the label regions in memory are marked as Read/Write within ghidra. We want to resolve those labels to strings for ease of use.

![Ghidra Decompiler Output with Labes](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/71968514689b2b9ebf6a9f82e12e27aa57bfb0a0/03-ghidra-decompiler-with-labels.png)

Navigate to the data-base address of `0x02f00000 - 0x01000000` in Ghidra.  This last portion of the blob is where all the strings that are referenced in code live.  The actual location where these strings start is a little lower than `0x02f00000 - 0x01000000`, we will need to manually inspect the binary to see just where that location is.  This is most likely due to some sort of offset within text-base that I am simply not aware of.  I came up with the location `DAT_02de33d8` (`0x02de33d8`).  We arent necessarily looking for an exact location.

![Ghidra Memory Map Default](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/2a2f8d79c176a22dc10333bf951497a888e25633/04-ghidra-memory-map-default.png)

Open up `Window->Memory Map` and click the `Split` button up in the right hand corner of that screen.  You will need to split at the data-base address `0x02f00000`, and then mark the data-base memory region as Read Only.  

![Ghidra Memory Map Default Configuration Menu](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/f50c6488e7414390041b06d68ff14ff4ddf399dc/05-ghidra-memory-map-split.png)

![Ghidra Memory Map Split](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/f50c6488e7414390041b06d68ff14ff4ddf399dc/06-ghidra-memory-map-readonly.png)

After you have completed these steps, the labels should resolve to strings and you should be able to start reversing the image quite easily.

![Ghidra decompiler with strings resolved](https://gist.github.com/nstarke/ed0aba2c882b8b3078747a567ee00520/raw/f50c6488e7414390041b06d68ff14ff4ddf399dc/07-ghidra-decompiler-with-strings-resolved.png)

[Back](https://nstarke.github.io/)