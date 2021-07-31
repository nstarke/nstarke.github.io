---
layout: post
title:  "Create Debian PowerPC32 VM Under QEMU"
date:   2017-05-31 00:00:00 -0600
categories: ppwerpc reverse-engineering debugging qemu virtualization
author: Nicholas Starke
---

I have a collection of QEMU VMs for different CPU Architectures.  In an attempt to fill in some gaps on architectures I lacked VMs for, I decided to spin up a PowerPC32 VM under QEMU.  I chose Debian-PowerPC as the OS.

## Gathering Resources

Install the prerequisite PowerPC packages:

```bash
# apt-get install qemu-system-ppc openbios-ppc
```

Download an image of the Debian PowerPC ISO DVD from [https://cdimage.debian.org/debian-cd/current/powerpc/iso-dvd/](https://cdimage.debian.org/debian-cd/current/powerpc/iso-dvd/)

As of this writing the current version DVD ISO file name is `debian-8.8.0-powerpc-DVD-1.iso` and this is the filename we will use for the following examples.

Mount this ISO:

```bash
$ mkdir ~/powerpc-mnt
$ sudo mount /path/to/iso/debian-8.8.0-powerpc-DVD-1.iso ~/powerpc-mnt
```

You will need two files from this iso:

```bash
$ cp ~/powerpc-mnt/install/powerpc/initrd.gz ./
$ cp ~/powerpc-mnt/install/powerpc/vmlinux ./
```

Unmount the iso:

```bash
$ sudo umount ~/powerpc-mnt
```

## QEMU VM Installation

Create the virtual disk:

```bash
$ qemu-img create -f qcow2 powerpc32.img 12G
```

Start up the QEMU VM with the iso attached as a usb device for installation:

```bash
$ qemu-system-ppc -m 1024 -boot d -hda powerpc32.img -initrd initrd.gz -kernel vmlinux -append  "cdrom-detect/try-usb=true" -usb -usbdevice disk:/full/path/to/iso/debian-8.8.0-powerpc-DVD-1.iso  -noreboot
```

Make sure you selected `Guided - use entire disk` when partitioning the hard disk.  **Do not** use LVM.

When selecting packages, ensure `ssh server` is selected.

## Extract initrd.gz and vmlinux

You will need the initrd.gz and vmlinux files from the powerpc32.img disk.

```bash
$ sudo modprobe nbd max_part=16
$ sudo qemu-nbd -c /dev/nbd0 armdisk.img
$ mkdir ~/qemu-mounted
$ sudo mount /dev/nbd0p2 ~/qemu-mounted
$ mkdir after-copy

$ cp ~/qemu-mounted/initrd.img-3.16.0-4-powerpc after-copy/
$ cp ~/qemu-mounted/vmlinux-3.16.0-4-powerpc after-copy/

$ sudo umount ~/qemu-mounted
$ sudo qemu-nbd -d /dev/nbd0
```
_source: [https://gist.github.com/Liryna/10710751](https://gist.github.com/Liryna/10710751)_
 
Note that these two files may have different names depending on the version of the kernel provided in your ISO.  Look for files in ~/qemu-mounted with the same prefix as these files:
* initrd.img-3.16.0-4-powerpc
* vmlinux-3.16.0-4-powerpc

## Boot into installed OS

You are now ready to boot into the fully installed operating system.  Use a command like the one below to boot:

```bash
$ qemu-system-ppc -m 1024 -hda powerpc32.img -initrd after-copy/initrd.img-3.16.0-4-powerpc -kernel after-copy/vmlinux-3.16.0-4-powerpc -append  "root=/dev/sda3" -redir tcp:7777::22
```

To install packages via apt, you will need to edit `/etc/apt/sources.list`.
Make sure all sources referencing the cdrom source are commented out.  These lines begin with `deb cdrom:[Debian...`
Uncomment out the last two lines of the file.  They should be changed from:

```
deb http://ftp.debian.org/debian/ jessie-updates main
deb-src http://ftp.debian.org/debian/ jessie-updates main
```

To:

```
deb http://ftp.debian.org/debian/ jessie main contrib non-free
deb-src http://ftp.debian.org/debian/ jessie main contrib non-free
```
You should now have a fully functional PowerPC VM running Debian Jessie under QEMU.