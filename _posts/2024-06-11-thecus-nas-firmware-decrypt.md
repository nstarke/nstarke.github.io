---
layout: posts
title: "Thecus NAS Firmware Decryption"
date: 2024-06-11 00:00:00 -0600
categories: cryptography firmware
author: Nicholas Starke
---


## tl;dr

The password I've seen consistently used across a few different firmware images is `N16000`. The cipher is DES-CBC using the MIT Kerberos `DES_to_string_key` function for key derivation.

## Introduction

Back in 2018 I did some research into [Thecus NAS firmware](https://www.thecus.com).  I found that the firmware was encrypted, and I figured out how to decrypt it using parts of the filename.  The firmware for Thecus NAS are encrypted with DES-CBC using the MIT Kerberos `DES_string_to_key` function.  I wrote up some basics scripts to complete this task and posted them [here](https://gist.github.com/nstarke/eaba741a99049430bdcb74f1b4ebc651).

Flash forward to earlier this week, its been six years since I looked at this research and someone online reached out to me to ask a few questions about my scripts.  For whatever reason they no longer were working.  Thanks to `pyro_phoenix` for reaching out, and also for figuring out how to enable `openssl` cli to decrypt the firmware.

It turns out DES-CBC is deprecated and the `openssl` cli will no longer allow the user to use DEC-CBC or DES-ECB by default.  We can work around this by using a custom `openssl.cnf` file that enables legacy ciphers.  Thanks again to pyro_phoenix for pointing out how to make the `openssl` cli command work for DES-CBC.  This is an example of the `openssl` cli output when trying to use `DES-CBC` as the cipher:

```
(./string2key N16000) -nopad -nosalt
hex string is too long, ignoring excess
Error setting cipher DES-CBC
4097B607E1750000:error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:../crypto/evp/evp_fetch.c:386:Global default library context, Algorithm (DES-CBC : 8), Properties ()
```

However, I did end up rewriting my origin bash scripts into a python program.  The python program iterates through a list of all possible model names (I used the thecus website to build a collection of model names) and attempts to use each as the key to the encrypted firmware blob.  

## New Scripts

My new python-based scripts are [available on github](https://github.com/nstarke/thecus-firmware-decrypt) for those who are interested.  The new script still uses the `string2key` program I wrote, which is included in the repository.  This program utilizes the `DES_string_to_key` function from openssl to derive a key from a string.  I looked at several difficult python re-implementations of this function and none of the python-based solutions worked properly. Thus I decided to resort to shelling out to this small c program to handle key derivation.  

The script is able to decrypt the firmware file itself using the [pyDes](https://github.com/twhiteman/pyDes) python-only implementation of the DES cryptographic functions.  The encrypt/decrypt functions are very very slow, as the author of the library notes: `10Kb/s`.  The average Thecus firmware file is about 180 megabytes in size, which means it takes hours to decrypt using the `pyDes` library.  However, with using `openssl` cli, the time is reduced to less than a minute. 

The example `openssl_legacy.cnf` file provided in the repo can be used to enable legacy ciphers such as DES-CBC.  You can pass this config file path using the `OPENSSL_CONF` environment variable. Even the original gist scripts should work if you set this environment variable to the provided `openssl_legacy.cnf` file.

**Example**

```
sudo apt install libssl-dev
gcc -o string2key string2key.c -lssl -lcrypto
OPENSSL_CONF=openssl_legacy.cnf openssl des-cbc -d -in Thecus_x86_64_FW.2.06.03.cdv_build9857_N2800_N4510U_N4800_N5550_N7510.rom -out Thecus_x86_64_FW.2.06.03.cdv_build9857_N2800_N4510U_N4800_N5550_N7510.rom.decrypted.bin -iv 00000000000000000 -K $(./string2key N16000) -nopad -nosalt
```

# Conclusion

I haven't looked at or thought about this research in 6 years and it was fun to revisit.  I can't really remember how I figured all of this out originally, but it was fun to formalize the research into a git repository instead of gists.

If you find anything cool in the firmware images, give me a shout out!