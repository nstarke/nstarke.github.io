# Linksys EA4500 v3.0 Firmware Decryption

Published: March 24, 2020

The first thing I wanted to do was to update the firmware for the device.
[https://www.linksys.com/us/support-article?articleNum=148385](https://www.linksys.com/us/support-article?articleNum=148385) offers the latest version of the firmware, which is `3.1.7` as of this writing.  

However, we can see with the filename that its probably encrypted: `FW_EA4500V3_3.1.7.181919_prod.gpg.img`

When I run binwalk I don't get any meaningful results, confirming my suspcicions:

```sh
$ binwalk FW_EA4500V3_3.1.7.181919_prod.gpg.img

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
3168198       0x3057C6        Cisco IOS experimental microcode, for "k"
```

__Note that the Cisco IOS experimental microcode result is almost always a false positive, even though this is a Cisco branded device.__

So we need to find the key. We know that it is a GPG key. The Linksys product support page says this:

>However, if you prefer to do manual updates and your router is on version 3.1.6.166173 or older, YOU MUST download & update your router using firmware version 3.1.6 (Build 172023) first before loading the latest firmware

This leads me to believe the firmware was once not encrypted and then a subsequent version was encrypted.  That means the gpg key is probably somewhere in an earlier firmware version.  So next we download the version `3.1.6`.  The filename is thus: `FW_EA4500V3_3.1.6.172023_prod.img`

Note that there is no `gpg` substring in the file name.  This leads me to believe that this firmware version is not encrypted.  We run binwalk to confirm:

```sh
$ binwalk FW_EA4500V3_3.1.6.172023_prod.img

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0x522CD43A, created: 2016-04-21 20:36:47, image size: 1536238 bytes, Data Address: 0x80060000, Entry Point: 0x80060000, data CRC: 0xF4B0BD80, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: "Linksys Impala Router"
64            0x40            LZMA compressed data, properties: 0x6D, dictionary size: 8388608 bytes, uncompressed size: 4555944 bytes
3145728       0x300000        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 14179096 bytes, 3201 inodes, blocksize: 262144 bytes, created: 2016-04-21 21:30:41
```

Score! So we can use binwalk's Matryoshka extraction mode to extract the squashs filesystem:

```sh
$ binwalk -eM FW_EA4500V3_3.1.6.172023_prod.img
```

Now that we have the filesystem extracted for version `3.1.6`, we need to go looking for something that looks like a public key.

I did a `find` for `*.asc*`, `*.pem`, and `*.gpg` to no avail, so I decided to just `grep` for what I was looking for:

```sh
$ grep -r 'PUBLIC KEY' .
./etc/certs/server.pem:-----BEGIN PUBLIC KEY-----
./etc/certs/server.pem:-----END PUBLIC KEY-----
./etc/keydata:-----BEGIN PGP PUBLIC KEY BLOCK-----
./etc/keydata:-----END PGP PUBLIC KEY BLOCK-----
Binary file ./lib/libcrypto.so.0.9.8 matches
Binary file ./usr/bin/gpg matches
```

From experience I can guess that `./etc/certs/server.pem` is most likely the web management interface's TLS cert for HTTPS communications.  But `./etc/keydata` looking interesting:

```sh
$ file ./etc/keydata 
./etc/keydata: PGP public key block Public-Key (old)
```

So let's try it out and see if it can decrypt the `3.1.7` firmware version:

```sh
$ gpg --import ./etc/keydata
$ gpg --decrypt FW_EA4500V3_3.1.7.181919_prod.gpg.img > FW_EA4500V3_3.1.7.181919_prod.img
```

And it works! Now we have a plaintext version of the firmware:

```sh
$ file FW_EA4500V3_3.1.7.181919_prod.img
FW_EA4500V3_3.1.7.181919_prod.img: u-boot legacy uImage, Linksys Impala Router, Linux/MIPS, OS Kernel Image (lzma), 1536489 bytes, Fri Jun 16 00:41:05 2017, Load Address: 0x80060000, Entry Point: 0x80060000, Header CRC: 0x9DACD513, Data CRC: 0xA097E0E3
$ binwalk FW_EA4500V3_3.1.7.181919_prod.img

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0x9DACD513, created: 2017-06-16 00:41:05, image size: 1536489 bytes, Data Address: 0x80060000, Entry Point: 0x80060000, data CRC: 0xA097E0E3, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: "Linksys Impala Router"
64            0x40            LZMA compressed data, properties: 0x6D, dictionary size: 8388608 bytes, uncompressed size: 4556000 bytes
3145728       0x300000        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 15154172 bytes, 3294 inodes, blocksize: 262144 bytes, created: 2017-06-16 01:34:18
```

Now I can start the reverse engineering process.

It's also worth noting that although we can decrypt the firmware for reverse engineering, we cannot encrypt our own firmware to upload to the device.  That integrity checking is mostly what the firmware decryption is in place for - to prevent malicious actors from uploading a modified firmware version.  Because the private key is kept private, it is not possible to encrypt custom firmware. Also, my bet is that the `3.1.7` firmware has checks in place to prevent unencrypted firmware from being flashed onto the device, meaning it is probably not possible to downgrade from `3.1.7`.

**Update:** Here is `/etc/keydata`:
```
-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v1

mQENBFZM/2EBCADOq3bG1vDkWZkMFynSoCSsz0iD5DtD7igVJ/GizWn/oD/fNGld
WX3D6VInykkcWPQf+CAmvxzGNEJGISBOd7x2Hcr3c1Yf/HMdTWwSzjRbfxj1lThf
DBv9JPBmZ8YYywK3N3dBh4s+m33pOqZxvoOAcR1cCJSECt8L0AV+BwI+1XnhUERy
nYt1BKCkh5MRhuYLFCqIcjFQxScfHEiCZFnuo7XPjSv4kFqdt8W+dxH4hh+ieFFw
m4lfeVKHR8GG+K1lFhAjVAiIKp49iY0tlgGqkARo5nIRyIW0KVOyZ+ikc0/M97LN
XIpf9u16jUb0xigjBt2njMm2bxc/3vsm92JrABEBAAG0KWxpbmtzeXMgPGxpbmtz
eXMtdG9vbC1zdXBwb3J0QGJlbGtpbi5jb20+iQE4BBMBAgAiBQJWTP9hAhsvBgsJ
CAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRB5cFSk6+K7j6pyCAC2DUoKbXyb5ZZs
bFTVdctM9DhYGEegXG68yQoHSk9GAeCXkpHHG6iRy2Jf1AJVFttKGx243Y6ewoGe
UUYyCg/v0p3ZxuWUjJpjeULNQZDXhD+xHIIYLBjb92bXHLIWcR+QTuKVHHWJSjtu
kD0yMPtcYKcxfO+r5OZTi0agQdvQZLVicQg09bkMdXg4+9QrOnMkjN85kD1zQSPt
Lov+M245NHtOJ80IZ5PI7bhMS/NY1uShWhBYfIMYlBmyYOse8uJaMMh3P3tBvE5O
qYNh9564Iq1f2Rpg3fmAJriwPrBMsEenOWLS8dt8EiLDM5DxKDoo6l6NwYNZ6sWM
O0q6JsIP
=svmk
-----END PGP PUBLIC KEY BLOCK-----
```

[Back](https://nstarke.github.io/)