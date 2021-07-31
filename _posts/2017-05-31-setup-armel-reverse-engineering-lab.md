---
layout: post
title:  "Setting up an ARMEL Reverse Engineering / Debug Lab in QEMU"
date:   2017-05-31 00:00:00 -0600
categories: arm reverse-engineering debugging qemu virtualization
author: Nicholas Starke
---

I recently came across a tutorial on ARM Reverse Engineering [https://azeria-labs.com/writing-arm-assembly-part-1/](https://azeria-labs.com/writing-arm-assembly-part-1/).

However, this tutorial seems to recommend using a Raspberry Pi for following along with the tutorial.  I decided I wanted to be able to work through the tutorial using a virtual machine, so I built a QEMU VM of the ARMEL architecture.  This is the same architecture that the Raspberry Pi is based off of.
I went with debian for ARMEL because its the OS I'm most familiar with.
After the operating system is installed, I install tools like GDB and GEF for debugging / reverse engineering.

GEF is a plugin for GDB specifically built for reverse engineering and exploit development.  From [https://github.com/hugsy/gef.git](https://github.com/hugsy/gef.git):

```
GEF is a kick-ass set of commands for X86, ARM, MIPS, PowerPC and SPARC to make GDB cool again for exploit dev. It is aimed to be used mostly by exploiters and reverse-engineers, to provide additional features to GDB using the Python API to assist during the process of dynamic analysis and exploit development.
```

Most of the QEMU setup instructions came from [this gist](https://gist.github.com/Liryna/10710751), but are provided again here for completeness. I've also streamlined / edited many of the commands.

If you want to actually run Raspian on QEMU, I recommend [this guide](https://azeria-labs.com/emulate-raspberry-pi-with-qemu/).

## Creating the QEMU VM

Create a disk image:

```bash
$ qemu-img create -f qcow2 armdisk.img 12G
```

Download VMLinuz and Initrd.gz for installation:

```bash
$ wget -r --no-parent -nH --cut-dirs=9 -R index.html* http://ftp.debian.org/debian/dists/wheezy/main/installer-armel/current//images/versatile/netboot/  
```

Boot into install mode:

```bash
$ qemu-system-arm -m 1024M -M versatilepb -kernel vmlinuz-3.2.0-4-versatile -initrd initrd.gz -append "root=/dev/ram" -hda armdisk.img -no-reboot
```

When installing the operating system, ensure that you **DO NOT** select LVM when partitioning the disk.  Simply use the first partitioning option, which is something along the lines of `Use the entire disk`.

Extract the kernel and initrd:

```bash
$ sudo modprobe nbd max_part=16
$ sudo qemu-nbd -c /dev/nbd0 armdisk.img
$ mkdir ~/qemu-mounted
$ sudo mount /dev/nbd0p1 ~/qemu-mounted
$ mkdir after-copy

$ cp ~/qemu-mounted/boot/* after-copy/

$ sudo umount ~/qemu-mounted
$ sudo qemu-nbd -d /dev/nbd0
```
_source: https://gist.github.com/Liryna/10710751_

Fire up the VM with the proper parammeters:

```bash
$ qemu-system-arm -M versatilepb -m 1024M  -kernel after-copy/vmlinuz-3.2.0-4-versatile -initrd after-copy/initrd.img-3.2.0-4-versatile -hda armdisk.img -append "root=/dev/sda1" -redir tcp:5555::22 -nographic 
```

## Setting up RE Tools
Wait a few minutes for the VM to boot up, then SSH into the VM on localhost port 5555 using the following command:

```bash
$ ssh -p 5555 user@localhost
```

Once inside the VM, change to the root user:

```bash
$ su
```

Then run the following to install a few prerequisite packages:

```bash
# apt-get install git build-essential python-dev
```

A brief rundown on what each of the dependencies is for:
* git - cloning GEF
* build-essential - the build tools we'll need to build GDB
* python-dev - headers necessary for building GDB with python support.

Next grab a recent version of GDB, build, and install:

```bash
# wget http://ftp.gnu.org/gnu/gdb/gdb-7.12.tar.gz
# tar xzf gdb-7.12.tar.gz
# cd gdb-7.12
# ./configure --with-python
# make
# make install
# ln -s /usr/local/bin/gdb /usr/bin/gdb
# exit
```

Note that the build process will take a long time (2-4 hours) under QEMU. We need to install a recent version of GDB because GEF does not support any version of GDB before 7.x. See [this issue](https://github.com/hugsy/gef/issues/148) for more information.

Next we want to install GDB-GEF:

```bash
$ git clone https://github.com/hugsy/gef.git ~/gef
$ echo "source ~/gef/gef.git" >> ~/.gdbinit
```

If at this point you run `gdb`, you will get a SEGFAULT.

Lastly, we need to set `readline_comapt = true`.  Add the following to `~/.gef.rc`:
```
[gef]
readline_compat = True
```

As per [this document](https://gef.readthedocs.io/en/latest/faq/#i-get-a-segfault-when-starting-gdb-with-gef), this will prevent GEF from segfaulting upon launch of GDB.

At this point you should be able to use GEF with GDB on an ARMEL VM under QEMU.