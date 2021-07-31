---
layout: posts
title:  "Decrypting DLINK Proprietary Firmware Images"
date:   2020-07-19 00:00:00 -0600
categories: dlink firmware cryptography
author: Nicholas Starke
---

The DIR-3040 models of DLINK routers feature encrypted firmware images in the most recent versions of the firmware. [https://support.dlink.com/ProductInfo.aspx?m=DIR-3040-US](https://support.dlink.com/ProductInfo.aspx?m=DIR-3040-US) details the firmware images available for this product.  

* `1.11B02` - [ftp://ftp2.dlink.com/PRODUCTS/DIR-3040/REVA/DIR-3040_REVA_FIRMWARE_v1.11B02.zip](ftp://ftp2.dlink.com/PRODUCTS/DIR-3040/REVA/DIR-3040_REVA_FIRMWARE_v1.11B02.zip)
* `1.02B03` - [ftp://ftp2.dlink.com/PRODUCTS/DIR-3040/REVA/DIR-3040_REVA_FIRMWARE_v1.02B03.zip](ftp://ftp2.dlink.com/PRODUCTS/DIR-3040/REVA/DIR-3040_REVA_FIRMWARE_v1.02B03.zip)

Unzipping the first reveals two files:

* `DIR-3040_REVA_RELEASE_NOTES_v1.11B02.pdf`
* `DIR3040A1_FW111B02.bin` 

Running binwalk on `DIR3040A1_FW111B02.bin` reveals:

```
$ binwalk DIR3040A1_FW111B02.bin 
DECIMAL       HEXADECIMAL     DESCRIPTION 
--------------------------------------------------------------------------------                                                                                                                                                                                                    
```

The lack of binwalk output almost surely means the firmware file is encrypted.

Unzipping the older firmware image reveals three files:

* `DIR-3040_REVA_RELEASE_NOTES_v1.02B03.pdf`
* `DIR3040A1_FW102B03.bin`
* `DIR3040A1_FW102B03_uncrypted.bin`

The last file ends with `uncrypted.bin`, which was my clue this version of the firmware image was not encrypted.  Running binwalk on this file reveals:

```
$ binwalk DIR3040A1_FW102B03_uncrypted.bin 
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0xDFA42E6F, created: 2019-09-18 07:16:48, image size: 17637193 bytes, Data Address: 0x81001000, Entry Point: 0x816357A0, data CRC: 0xAD066126, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: "Linux Kernel Image"
160           0xA0            LZMA compressed data, properties: 0x5D, dictionary size: 33554432 bytes, uncompressed size: 23062976 bytes
8476944       0x815910        MySQL ISAM compressed data file Version 4
```

Bingo, a uImage header and accompanying filesystem.  We can extract this using `binwalk -eM DIR3040A1_FW102B03_uncrypted.bin`.

Looking at the filesystem, the first thing I did is look for certificates:

```
./etc_ro/public.pem
./etc_ro/cacert.pem
./etc_ro/mydlink/client-ca.crt.pem
./etc_ro/mosquitto/ssl/clientcrt.pem
./etc_ro/mosquitto/ssl/cacrt.pem
./etc_ro/mosquitto/ssl/cakey.pem
./etc_ro/mosquitto/ssl/servercrt.pem
./etc_ro/mosquitto/ssl/serverkey.pem
./etc_ro/mosquitto/ssl/clientkey.pem 
```

Next I take a look at the binaries on the filesystem.  One catches my eye: `/bin/imgdecrypt`. We could also have found this by grepping for `/etc_ro/public.pem`.

We load this into Ghidra and take a look:

```c
int decrypt_firmare(int param_1,undefined4 *param_2)
{
  int iVar1;
  char *local_20;
  int local_1c;
  undefined4 local_18;
  undefined4 local_14;
  undefined4 local_10;
  undefined4 local_c;
  
  local_18 = 0x33323130;
  local_14 = 0x37363534;
  local_10 = 0x42413938;
  local_c = 0x46454443;
  local_20 = "/etc_ro/public.pem";
  if (param_1 < 2) {
    printf("%s <sourceFile>\r\n",*param_2);
    iVar1 = -1;
  }
  else {
    if (2 < param_1) {
      local_20 = (char *)param_2[2];
    }
    iVar1 = FUN_004021ac(local_20,0);
    if (iVar1 == 0) {
      FUN_004025a4(&local_18);
      printf("key:");
      local_1c = 0;
      while (local_1c < 0x10) {
        printf("%02X",(int)*(char *)((int)&local_18 + local_1c) & 0xff);
        local_1c = local_1c + 1;
      }
      puts("\r");
      iVar1 = FUN_00401770(param_2[1],"/tmp/.firmware.orig",&local_18);
      if (iVar1 == 0) {
        unlink((char *)param_2[1]);
        rename("/tmp/.firmware.orig",(char *)param_2[1]);
      }
      RSA_free(DAT_00413220);
    }
    else {
      iVar1 = -1;
    }
  }
  return iVar1;
}
```

The disassembly for this function looks like this:
```
                             **************************************************************
                             *                          FUNCTION                          *
                             **************************************************************
                             undefined __stdcall decrypt_firmare(undefined4 argc, und
                               assume gp = 0x41b110
             undefined         v0:1           <RETURN>
             undefined4        a0:4           argc
             undefined4        a1:4           infile
             undefined4        Stack[0x4]:4   local_res4                              XREF[6]:     00402624(W), 
                                                                                                   00402688(R), 
                                                                                                   004026d4(R), 
                                                                                                   004027e4(R), 
                                                                                                   00402828(R), 
                                                                                                   00402858(R)  
             undefined4        Stack[0x0]:4   local_res0                              XREF[3]:     00402620(W), 
                                                                                                   0040266c(R), 
                                                                                                   004026c0(R)  
             undefined4        Stack[-0x4]:4  local_4                                 XREF[2]:     00402610(W), 
                                                                                                   004028b8(R)  
             undefined4        Stack[-0xc]:4  local_c                                 XREF[1]:     00402654(W)  
             undefined4        Stack[-0x10]:4 local_10                                XREF[1]:     00402650(W)  
             undefined4        Stack[-0x14]:4 local_14                                XREF[1]:     0040264c(W)  
             undefined4        Stack[-0x18]:4 local_18                                XREF[1]:     00402648(W)  
             undefined4        Stack[-0x1c]:4 local_1c                                XREF[11]:    00402668(W), 
                                                                                                   004026fc(W), 
                                                                                                   00402700(R), 
                                                                                                   00402754(W), 
                                                                                                   00402768(R), 
                                                                                                   004027a0(R), 
                                                                                                   004027ac(W), 
                                                                                                   004027b0(R), 
                                                                                                   00402814(W), 
                                                                                                   00402818(R), 
                                                                                                   004028b4(R)  
             undefined4        Stack[-0x20]:4 local_20                                XREF[3]:     00402660(W), 
                                                                                                   004026e4(W), 
                                                                                                   004026e8(R)  
             undefined4        Stack[-0x28]:4 local_28                                XREF[11]:    0040261c(W), 
                                                                                                   004026b0(R), 
                                                                                                   004026f8(R), 
                                                                                                   0040272c(R), 
                                                                                                   00402750(R), 
                                                                                                   0040279c(R), 
                                                                                                   004027e0(R), 
                                                                                                   00402810(R), 
                                                                                                   00402854(R), 
                                                                                                   00402888(R), 
                                                                                                   004028b0(R)  
                             decrypt_firmare                                 XREF[3]:     Entry Point(*), main:00402b1c(c), 
                                                                                          00413138(*)  
        0040260c c8 ff bd 27     addiu      sp,sp,-0x38
             assume gp = <UNKNOWN>
        00402610 34 00 bf af     sw         ra,local_4(sp)
        00402614 42 00 1c 3c     lui        gp,0x42
        00402618 10 b1 9c 27     addiu      gp,gp,-0x4ef0
        0040261c 10 00 bc af     sw         gp=>_gp,local_28(sp)
        00402620 38 00 a4 af     sw         argc,local_res0(sp)
        00402624 3c 00 a5 af     sw         infile,local_res4(sp)
        00402628 40 00 02 3c     lui        v0,0x40
        0040262c 74 30 45 8c     lw         infile,offset s_0123456789ABCDEF_00403074(v0)    = "0123456789ABCDEF"
        00402630 74 30 43 24     addiu      v1,v0,0x3074
        00402634 04 00 64 8c     lw         argc,0x4(v1)=>s_456789ABCDEF_00403074+4          = "456789ABCDEF"
        00402638 74 30 43 24     addiu      v1,v0,0x3074
        0040263c 08 00 63 8c     lw         v1,0x8(v1)=>s_89ABCDEF_00403074+8                = "89ABCDEF"
        00402640 74 30 42 24     addiu      v0,v0,0x3074
        00402644 0c 00 42 8c     lw         v0,0xc(v0)=>s_CDEF_00403074+12                   = "CDEF"
        00402648 20 00 a5 af     sw         infile,local_18(sp)
        0040264c 24 00 a4 af     sw         argc,local_14(sp)
        00402650 28 00 a3 af     sw         v1,local_10(sp)
        00402654 2c 00 a2 af     sw         v0,local_c(sp)
        00402658 40 00 02 3c     lui        v0,0x40
        0040265c 30 30 42 24     addiu      v0,v0,0x3030
        00402660 18 00 a2 af     sw         v0=>s_/etc_ro/public.pem_00403030,local_20(sp)   = "/etc_ro/public.pem"
        00402664 ff ff 02 24     li         v0,-0x1
        00402668 1c 00 a2 af     sw         v0,local_1c(sp)
        0040266c 38 00 a2 8f     lw         v0,local_res0(sp)
        00402670 00 00 00 00     nop
        00402674 02 00 42 28     slti       v0,v0,0x2
        00402678 11 00 40 10     beq        v0,zero,LAB_004026c0
        0040267c 00 00 00 00     _nop
        00402680 40 00 02 3c     lui        v0,0x40
        00402684 44 30 43 24     addiu      v1,v0,0x3044
        00402688 3c 00 a2 8f     lw         v0,local_res4(sp)
        0040268c 00 00 00 00     nop
        00402690 00 00 42 8c     lw         v0,0x0(v0)
        00402694 21 20 60 00     move       argc=>s_%s_<sourceFile>_00403044,v1              = "%s <sourceFile>\r\n"
        00402698 21 28 40 00     move       infile,v0
        0040269c 44 80 82 8f     lw         v0,-0x7fbc(gp)=>->printf                         = 00402e00
        004026a0 00 00 00 00     nop
        004026a4 21 c8 40 00     move       t9,v0
        004026a8 09 f8 20 03     jalr       t9=>printf                                       int printf(char * __format, ...)
        004026ac 00 00 00 00     _nop
        004026b0 10 00 bc 8f     lw         gp,local_28(sp)
        004026b4 ff ff 02 24     li         v0,-0x1
        004026b8 2e 0a 10 08     j          LAB_004028b8
        004026bc 00 00 00 00     _nop
                             LAB_004026c0                                    XREF[1]:     00402678(j)  
        004026c0 38 00 a2 8f     lw         v0,local_res0(sp)
        004026c4 00 00 00 00     nop
        004026c8 03 00 42 28     slti       v0,v0,0x3
        004026cc 06 00 40 14     bne        v0,zero,LAB_004026e8
        004026d0 00 00 00 00     _nop
        004026d4 3c 00 a2 8f     lw         v0,local_res4(sp)
        004026d8 00 00 00 00     nop
        004026dc 08 00 42 8c     lw         v0,0x8(v0)
        004026e0 00 00 00 00     nop
        004026e4 18 00 a2 af     sw         v0,local_20(sp)
                             LAB_004026e8                                    XREF[1]:     004026cc(j)  
        004026e8 18 00 a4 8f     lw         argc,local_20(sp)
        004026ec 21 28 00 00     clear      infile
        004026f0 6b 08 10 0c     jal        FUN_004021ac                                     undefined FUN_004021ac()
        004026f4 00 00 00 00     _nop
        004026f8 10 00 bc 8f     lw         gp,local_28(sp)
        004026fc 1c 00 a2 af     sw         v0,local_1c(sp)
        00402700 1c 00 a2 8f     lw         v0,local_1c(sp)
        00402704 00 00 00 00     nop
        00402708 04 00 40 10     beq        v0,zero,LAB_0040271c
        0040270c 00 00 00 00     _nop
        00402710 ff ff 02 24     li         v0,-0x1
        00402714 2e 0a 10 08     j          LAB_004028b8
        00402718 00 00 00 00     _nop
                             LAB_0040271c                                    XREF[1]:     00402708(j)  
        0040271c 20 00 a2 27     addiu      v0,sp,0x20
        00402720 21 20 40 00     move       argc,v0
        00402724 69 09 10 0c     jal        FUN_004025a4                                     undefined FUN_004025a4()
        00402728 00 00 00 00     _nop
        0040272c 10 00 bc 8f     lw         gp,local_28(sp)
        00402730 40 00 02 3c     lui        v0,0x40
        00402734 58 30 42 24     addiu      v0,v0,0x3058
        00402738 21 20 40 00     move       argc=>DAT_00403058,v0                            = 6Bh    k
        0040273c 44 80 82 8f     lw         v0,-0x7fbc(gp)=>->printf                         = 00402e00
        00402740 00 00 00 00     nop
        00402744 21 c8 40 00     move       t9,v0
        00402748 09 f8 20 03     jalr       t9=>printf                                       int printf(char * __format, ...)
        0040274c 00 00 00 00     _nop
        00402750 10 00 bc 8f     lw         gp,local_28(sp)
        00402754 1c 00 a0 af     sw         zero,local_1c(sp)
        00402758 ec 09 10 08     j          LAB_004027b0
        0040275c 00 00 00 00     _nop
                             LAB_00402760                                    XREF[1]:     004027bc(j)  
        00402760 40 00 02 3c     lui        v0,0x40
        00402764 54 2f 43 24     addiu      v1,v0,0x2f54
        00402768 1c 00 a2 8f     lw         v0,local_1c(sp)
        0040276c 18 00 a4 27     addiu      argc,sp,0x18
        00402770 21 10 82 00     addu       v0,argc,v0
        00402774 08 00 42 80     lb         v0,0x8(v0)
        00402778 00 00 00 00     nop
        0040277c ff 00 42 30     andi       v0,v0,0xff
        00402780 21 20 60 00     move       argc=>DAT_00402f54,v1                            = 25h    %
        00402784 21 28 40 00     move       infile,v0
        00402788 44 80 82 8f     lw         v0,-0x7fbc(gp)=>->printf                         = 00402e00
        0040278c 00 00 00 00     nop
        00402790 21 c8 40 00     move       t9,v0
        00402794 09 f8 20 03     jalr       t9=>printf                                       int printf(char * __format, ...)
        00402798 00 00 00 00     _nop
        0040279c 10 00 bc 8f     lw         gp,local_28(sp)
        004027a0 1c 00 a2 8f     lw         v0,local_1c(sp)
        004027a4 00 00 00 00     nop
        004027a8 01 00 42 24     addiu      v0,v0,0x1
        004027ac 1c 00 a2 af     sw         v0,local_1c(sp)
                             LAB_004027b0                                    XREF[1]:     00402758(j)  
        004027b0 1c 00 a2 8f     lw         v0,local_1c(sp)
        004027b4 00 00 00 00     nop
        004027b8 10 00 42 28     slti       v0,v0,0x10
        004027bc e8 ff 40 14     bne        v0,zero,LAB_00402760
        004027c0 00 00 00 00     _nop
        004027c4 40 00 02 3c     lui        v0,0x40
        004027c8 5c 2f 44 24     addiu      argc=>DAT_00402f5c,v0,0x2f5c                     = 0Dh
        004027cc 40 80 82 8f     lw         v0,-0x7fc0(gp)=>->puts                           = 00402e10
        004027d0 00 00 00 00     nop
        004027d4 21 c8 40 00     move       t9,v0
        004027d8 09 f8 20 03     jalr       t9=>puts                                         int puts(char * __s)
        004027dc 00 00 00 00     _nop
        004027e0 10 00 bc 8f     lw         gp,local_28(sp)
        004027e4 3c 00 a2 8f     lw         v0,local_res4(sp)
        004027e8 00 00 00 00     nop
        004027ec 04 00 42 24     addiu      v0,v0,0x4
        004027f0 00 00 43 8c     lw         v1,0x0(v0)
        004027f4 20 00 a2 27     addiu      v0,sp,0x20
        004027f8 21 20 60 00     move       argc,v1
        004027fc 40 00 03 3c     lui        v1,0x40
        00402800 60 30 65 24     addiu      infile=>s_/tmp/.firmware.orig_00403060,v1,0x3060 = "/tmp/.firmware.orig"
        00402804 21 30 40 00     move       a2,v0
        00402808 dc 05 10 0c     jal        decrypt_and_writeout                             undefined decrypt_and_writeout(u
        0040280c 00 00 00 00     _nop
        00402810 10 00 bc 8f     lw         gp,local_28(sp)
        00402814 1c 00 a2 af     sw         v0,local_1c(sp)
        00402818 1c 00 a2 8f     lw         v0,local_1c(sp)
        0040281c 00 00 00 00     nop
        00402820 1a 00 40 14     bne        v0,zero,LAB_0040288c
        00402824 00 00 00 00     _nop
        00402828 3c 00 a2 8f     lw         v0,local_res4(sp)
        0040282c 00 00 00 00     nop
        00402830 04 00 42 24     addiu      v0,v0,0x4
        00402834 00 00 42 8c     lw         v0,0x0(v0)
        00402838 00 00 00 00     nop
        0040283c 21 20 40 00     move       argc,v0
        00402840 6c 80 82 8f     lw         v0,-0x7f94(gp)=>->unlink                         = 00402d60
        00402844 00 00 00 00     nop
        00402848 21 c8 40 00     move       t9,v0
        0040284c 09 f8 20 03     jalr       t9=>unlink                                       int unlink(char * __name)
        00402850 00 00 00 00     _nop
        00402854 10 00 bc 8f     lw         gp,local_28(sp)
        00402858 3c 00 a2 8f     lw         v0,local_res4(sp)
        0040285c 00 00 00 00     nop
        00402860 04 00 42 24     addiu      v0,v0,0x4
        00402864 00 00 42 8c     lw         v0,0x0(v0)
        00402868 40 00 03 3c     lui        v1,0x40
        0040286c 60 30 64 24     addiu      argc=>s_/tmp/.firmware.orig_00403060,v1,0x3060   = "/tmp/.firmware.orig"
        00402870 21 28 40 00     move       infile,v0
        00402874 50 80 82 8f     lw         v0,-0x7fb0(gp)=>->rename                         = 00402dd0
        00402878 00 00 00 00     nop
        0040287c 21 c8 40 00     move       t9,v0
        00402880 09 f8 20 03     jalr       t9=>rename                                       int rename(char * __old, char * 
        00402884 00 00 00 00     _nop
        00402888 10 00 bc 8f     lw         gp,local_28(sp)
                             LAB_0040288c                                    XREF[1]:     00402820(j)  
        0040288c 41 00 02 3c     lui        v0,0x41
        00402890 20 32 42 8c     lw         v0,offset DAT_00413220(v0)                       = ??
        00402894 00 00 00 00     nop
        00402898 21 20 40 00     move       argc,v0
        0040289c 80 80 82 8f     lw         v0,-0x7f80(gp)=>->RSA_free                       = 00402d10
        004028a0 00 00 00 00     nop
        004028a4 21 c8 40 00     move       t9,v0
        004028a8 09 f8 20 03     jalr       t9=>RSA_free                                     void RSA_free(RSA * r)
        004028ac 00 00 00 00     _nop
        004028b0 10 00 bc 8f     lw         gp,local_28(sp)
        004028b4 1c 00 a2 8f     lw         v0,local_1c(sp)
                             LAB_004028b8                                    XREF[2]:     004026b8(j), 00402714(j)  
        004028b8 34 00 bf 8f     lw         ra,local_4(sp)
        004028bc 38 00 bd 27     addiu      sp,sp,0x38
        004028c0 08 00 e0 03     jr         ra
        004028c4 00 00 00 00     _nop

```

This function is responsible for decrypting the firmware.  There is some sort of key derived from the local stack variables that is not straightforward to reverse.  So instead of spending hours trying to figure out exactly what the code that produces the key does, why don't we just try to run the binary passing it our firmware image we want to decrypt?

```
$ file bin/imgdecrypt                                                   
bin/imgdecrypt: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, stripped 
```

So we are working with a MIPSel binary.  We can emulate this using `qemu-mipsel-static`.  Copy that binary into the firmware filesystem at `/usr/bin`:
```
$ cp $(which qemu-mipsel-static) ./usr/bin
```

You will also need to mount `/proc`, `/dev`, and `/sys` in the firmware filesystem root in order to for the `imgdecrypt` binary to run successfully:

```
#!/bin/bash
mount -t proc /proc proc/
mount --rbind /sys sys/
mount --rbind /dev/ dev/  
```

Copy the firmware image you want to decrypt into the firmware filesystem root.  In this case that file is `DIR3040A1_FW111B02.bin`

Next let's launch into a shell so that we can call the `imgdecrypt` binary:

```
$ sudo chroot . qemu-mipsel-static /bin/sh
BusyBox v1.22.1 (2019-09-18 14:32:32 CST) built-in shell (ash)
Enter 'help' for a list of built-in commands.
/ # 
```

Run the `imgdecrypt` binary against the firmware image file:
```
/ # /bin/imgdecrypt DIR3040A1_FW111B02.bin
key:C05FBF1936C99429CE2A0781F08D6AD8
```

Not only does it output the key to decrypt the firmware file image, but it also places the decrypted version in `/tmp/.firmware.orig`.
```
$ ls -a tmp                                                             
.  
..  
.firmware.orig 
```

We can then run binwalk over the `.firmware.orig` file:
```
$ binwalk ./tmp/.firmware.orig
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0x339E3A96, created: 2020-05-09 03:35:18, image size: 17644523 bytes, Data Address: 0x81001000, Entry Point: 0x81637600, data CRC: 0x5C75FC87, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: "Linux Kernel Image"
160           0xA0            LZMA compressed data, properties: 0x5D, dictionary size: 33554432 bytes, uncompressed size: 23079360 bytes    
```

Now we can extract using `binwalk -eM ./tmp/.firmware.orig`.