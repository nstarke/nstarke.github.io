# U-Boot Fuzzing

Published: March 12, 2021

I recently read some novel research into using AFL to fuzz the Grub2 Boot Loader [1].  Something I was not aware of before reading that article was that grub2 has an emulation mode that allows it to run as an elf binary. I decided to investigate whether or not these techniques could apply to the U-boot bootloader.  U-Boot, it turns out, also has an ELF emulation mode. All of my testing was against the `v2021.01` tag.

The hope was to find memory corruption bugs that might allow an attacker to bypass secure boot.

## TL;DR

My fuzz testing did not produce any meaningful crashing test cases against a "real" (virtualized) U-Boot instance.  This may have been because my test corpus was limited, as well as the fact that I could only execute one command at a time before the U-Boot emulated process exited.  The methodology is presented here in case anyone wants to perform similar testing, perhaps on a different input corpus.

## Patching U-Boot

I developed a small patch that will exit out of the U-Boot runtime after a command is run.  

```diff
diff --git a/common/cli_hush.c b/common/cli_hush.c
index 66995c255b..c05bad819e 100644
--- a/common/cli_hush.c
+++ b/common/cli_hush.c
@@ -84,6 +84,7 @@
 #include <cli.h>
 #include <cli_hush.h>
 #include <command.h>        /* find_cmd */
+#include <os.h>
 #ifndef CONFIG_SYS_PROMPT_HUSH_PS2
 #define CONFIG_SYS_PROMPT_HUSH_PS2     "> "
 #endif
@@ -3218,6 +3219,10 @@ static int parse_stream_outer(struct in_str *inp, int flag)
                        }
                        if (code == -1)
                            flag_repeat = 0;
+
+            if (code >= 0 ){
+                os_exit(0);
+            }
 #endif
                } else {
                        if (ctx.old_flag != 0) {
```

I also added `CONFIG_BOOTDELAY=0` to the U-Boot config file located at `.config`.  This will shorten the execution time of the U-Boot emulation by skipping the U-Boot autoboot countdown before each boot.

Once I had the patch developed, I built U-Boot for the sandbox environment:

```
$ make sandbox_defconfig
$ make
```

This generates an executable elf file named `u-boot` in the U-Boot source tree root. It is possible to send commands to the emulated U-Boot environment on stdin, provided there is a space character preceeding the command:

```
echo " echo test" | ./u-boot


U-Boot 2021.01-dirty (Mar 10 2021 - 19:32:21 +0000)

DRAM:  128 MiB
WDT:   Not found!
MMC:
In:    serial
Out:   serial
Err:   serial
SCSI:
Net:   No ethernet found.
Hit any key to stop autoboot:  0
=> echo test
test
```

## Developing AFL Input Corpus

At this point I developed a small corpus of U-Boot CLI commands.  I used:

```
 printenv
 showvar asdf
 source 0x0
 md 0 20
 ls emmc 0:0 /
 ls nand 0:0 /
```

Each line was saved to a separate file in my AFL input directory.  I then launched the AFL process:

```
$ afl-fuzz -Q -i ../in -o ../out -M main -- $UBOOT_PATH/u-boot
```

I used AFL++ [2] in qemu mode for this round of fuzzing. I let it run for two days at about 2-3 executions per second. This is obviously very slow, but it is still much faster than trying to manually test.

After two days AFL was reporting 160 unique crashes, and I needed some way to triage these test cases and validate them against a more "realistic" U-Boot environment.  I developed a few scripts to aid in automating the process.

## Processing Results

First of all, I needed the ability to run the AFL test cases as scripts within U-Boot. A U-Boot script is a set of U-Boot commands with a header prepended to the commands.  The script can be created using the `mkimage` executable that is located in the U-Boot source tree folder `tools`. 

Before we can run `mkimage` though, we need to remove the colons from the file names generated by AFL.  For example, one of my test cases files was of the form `id:000159,sig:11,src:000481,time:150469147,op:arith8,pos:114,val:+26`.  The `mkimage` command does not like file names with colons in them, so I used the `rename` command to remove the colons [3]:

```
$ cp out/main/crashes /tmp/fuzzer/crashes
$ cd /tmp/fuzzer/crashes
$ rename 's|:|-|g' *
```

Now we can feed out test cases files to the `mkimage` command and generate U-Boot scripts.  I wrote a small bash script to do this in bulk:

```bash
#!/bin/bash

COUNTER=0
for FILE in $(ls /tmp/fuzzer/crashes)
do
    echo "$FILE"
    $UBOOT_PATH/tools/mkimage -A arm -T script -C none -n "$FILE" -d "/tmp/fuzzer/crashes/$FILE" "$COUNTER".img
    COUNTER=$((COUNTER+1))
    echo $COUNTER
done
```

This script will create `.img` U-Boot script files for every testcases located in `/tmp/fuzzer/crashes`.  The `.img` file will be written to the current working directory.

In order to make the `.img` files accessible to the virtualized U-Boot instance, it is necessary to create a virtual SD card that U-Boot can then load the `.img` scripts from the SD card [4].

```
$ dd if=/dev/zero of=sd.img bs=4096 count=$((16384*8)
$ mkfs.vfat sd.img
$ mount sd.img /mnt -o loop,rw
```

This will create a 512MB virtual SD card.  I mounted the virtual SD card to `/mnt` and then `cd /mnt`.  Then I can run the above bash script and it will generate test cases for each of my AFL output files on the virtual SD card.

The last step is to generate a single script that will run all of the test case scripts.  I wrote a small bash script to do just that:

```bash
#!/bin/bash
COUNTER=0
rm runner.txt
for FILE in $(ls /tmp/fuzzer/crashes)
do
    echo "echo $COUNTER.img" >> runner.txt
    echo "fatload mmc 0:0 0x80000000 crashes/$COUNTER.img" >> runner.txt
    echo "source 0x80000000" >> runner.txt
    COUNTER=$((COUNTER+1))
done
$UBOOT_PATH/tools/mkimage -A arm -T script -C none -n "runner" -d runner.txt runner.img
```

This bash script creates a U-Boot script that does three things for each crashing test case U-Boot script file:

1) echo out the `.img` file name that is about to be run.
2) uses the `fatload` U-Boot command to load the current test case U-Boot script from the SD Card into memory.  I chose the memory address `0x80000000`.
3) uses the `source` U-Boot command to execute the script loaded into memory at address `0x80000000`.

When I run this script in `/mnt/crashes`, it will generate a `runner.img` U-Boot script that will run all the AFL test-case U-Boot scripts.

## Validating against a Virtualized Instance

Now that we have `runner.img` created, it is time to unmount the virtual SD card and fire up a virtualized instance of U-Boot. I chose the `vexpress_ca9x4` board and built a secondary instance of U-Boot as my "reference" build.  This board can be built by running the following commands:

```
$ sudo umount /mnt
$ make vexpress_ca9x4_defconfig
$ CROSS_COMPILE=arm-linux-gnueabihf- make
```

We can run the resulting `u-boot` file in qemu easily:

```
$ qemu-system-arm -m 1024 -M vexpress-a9 -kernel ./u-boot -nographic -sd ./sd.img
```

Within U-Boot we then use `fatload` to load `runner.img` and execute it:

```
=> fatload mmc 0 0x80000000 crashes/runner.img
=> source 0x80000000
```

This will execute all tests cases against the virtualized U-Boot instance.

# Results

While AFL reported 160 crashing test cases, none of the crashes could be triggered from within a virtualized U-Boot instance.  While the crashing test cases could be triggered in Emulation mode, they did not cause crashes or hangs in Virtualized mode.  I tested all crashing and hanging test cases generated by AFL++. This is effectively a negative result. 

Sources:

[1] [https://sthbrx.github.io/blog/2021/03/04/fuzzing-grub-part-1/](https://sthbrx.github.io/blog/2021/03/04/fuzzing-grub-part-1/)

[2] [https://github.com/AFLplusplus/AFLplusplus](https://github.com/AFLplusplus/AFLplusplus)

[3] [https://askubuntu.com/questions/227410/replace-all-colons-from-filenames-with-terminal](https://askubuntu.com/questions/227410/replace-all-colons-from-filenames-with-terminal)

[4] [https://stackoverflow.com/questions/46239926/booting-kernel-from-sd-in-qemu-arm-with-u-boot](https://stackoverflow.com/questions/46239926/booting-kernel-from-sd-in-qemu-arm-with-u-boot)

[Home](/)