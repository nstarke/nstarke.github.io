# Kexec and Kdump on Raspberry Pi

Published: April 2nd, 2021

I recently had a need to configure `kexec` and specifically `kdump` in order to capture kernel core dumps on raspberry pi.  This blog post is a short write up on things I learned in the process. I performed this operation on a Raspberry Pi Zero W but I have no reason to believe it wouldn't work on any other model of Rasperry Pi.

## Install

On Raspian, install `kdump-tools`:
```
# apt install kdump-tools
```

## Building the Kernel

You must set the following flags when you build the kernel:
```
CONFIG_KEXEC=y
CONFIG_DEBUG_INFO=y
CONFIG_CRASH_DUMP=y
CONFIG_SYSFS=y
CONFIG_PROC_VMCORE=y
```

These flags must be set in the kernel `.config` file.  I usually place them near the bottom, right before the line:

```
# end of Kernel hacking
```

The kernel build process might re-order the contents of the `.config` file once you start compiling.

Instructions on building the raspberry pi kernel, including cross compilation, are detailed here: [https://www.raspberrypi.org/documentation/linux/kernel/building.md](https://www.raspberrypi.org/documentation/linux/kernel/building.md).

## Setup

The way kdump works is it uses kexec to launch a new kernel image after the original kernel image panics. It does this without bouncing the operating system through the BIOS and bootloader.  In the second kernel image launched by kexec, `/proc/vmcore` exists and contains the kernel core dump. `/proc/vmcore` does not exist until the second kernel load. kdump then copies that core dump to the configured location.  By default, that is `/var/crash` on the root `ext4` filesystem, located at `/dev/mmcblk0`.

You must insure `/var/crash` exists on the root `ext4` fileystem.  If it does not, create it:

```
# mkdir /var/crash
```

kdump requires kexec to have a "captured" kernel image.  For our purposes, this is the same kernel image that we initially boot into. This one command should work across Raspberry Pi models with a stock raspbian install, but some things (like the network stack) won't work when kexec loads the second kernel image.  You might be able to get the network stack running by specifying the correct `dtb` (located in `/boot`). Also, you will need to adjust the `root` kernel command line to match your configuration if you use Noobs.

```
# cat /sys/kernel/kexec_crash_loaded
# kexec --type zImage -p /boot/kernel.img --append "root=/dev/mmcblk0p2 rootfstype=ext4 rootwait init=/sbin/init maxcpus=1"
# cat /sys/kernel/kexec_crash_loaded
```

You should see the first `cat` command result in the output `0` and the second `cat` command (after the `kexec` command), result in the output `1`.

You can check to make sure everything is working properly by triggering a kernel panic:

```
# echo c > /proc/sysrq-trigger
```

You should see on the main HDMI out of the Raspberry Pi the first kernel panic, ending in a message saying `Bye`.  Then the second kernel, loaded by kexec, should execute.  You will see the normal kernel boot up process, and then once `init` is launched, kdump will copy the kernel core dump to `/var/crash`.  kdump then reboots the Rasperry Pi to return the device to a known good state.

Once the Pi has rebooted, you should be able to access the crash files and utilize the `crash` command to analyze them.

[Back](/)