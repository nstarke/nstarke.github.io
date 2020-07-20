# Decrypting DLINK Proprietary Firmware Images

Published: July 19, 2020

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
