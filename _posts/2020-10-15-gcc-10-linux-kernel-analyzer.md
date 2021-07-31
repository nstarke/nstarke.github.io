---
layout: post
title:  "Modifying BIOS Using RU.EFI"
date:   2020-10-15 00:00:00 -0600
categories: linux kernel static-analysis gcc
author: Nicholas Starke
---

GCC version 10 has a new static analyzer built into it that can be utilized with the `-fanalyzer` CFLAG.  By hacking the Linux kernel Makefile, it is possible to a) force `make` to use `gcc-10` and b) force `make` to apply the `-fanalzyer` KBUILD_CFLAGS option. 

Why would one want to do this? Static analysis can provide direction on where to look for bugs.  In a project as large as the Linux kernel, this can be very helpful.

However, when you try to build in this configuration, you will be greated with a GCC error that has no flag:

```
error: arrays of functions are not meaningful 
```

I could find no way around this GCC error.  There was almost nothing on google about how to circumvent this error either. I decided to compile my own version of GCC with the `error()` code commented out. 

In GCC 10.2.0, this is line 8187 of `gcc/tree.c`:

```c
error ("arrays of functions are not meaningful");
```

Comment this line out and then compile GCC. For better instructions on compiling GCC, check this out: [https://solarianprogrammer.com/2016/10/07/building-gcc-ubuntu-linux/](https://solarianprogrammer.com/2016/10/07/building-gcc-ubuntu-linux/).  Note that if you set a unique `--program-suffix` it will make using your new GCC alongside your system installed gcc.  I chose `-10.2` for convenience.

Once you've compiled GCC, you'll need to modify the kernel Makefile to point to your newly compiled GCC installation. Or, you can use the CROSS_COMPILE environment variable.  If you're changing the Makefile, you will need to change this line:

```make
CC              = $(CROSS_COMPILE)gcc
```
And point it at your new GCC 10 installation.  If you set the `--program-suffix` to a unique value, you can just change the above line to:

```make
CC              = $(CROSS_COMPILE)gcc-10.2
```

To apply the `-fanalyzer` CFLAG, add `-fanalyzer` to the `KBUILD_CFLAGS` environment variable, or modify the following code in the Makefile:

```make
KBUILD_CFLAGS   := -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs \
                   -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE \
                   -Werror=implicit-function-declaration -Werror=implicit-int \
                   -Wno-format-security \
                   -std=gnu89
```

Change to:

```
KBUILD_CFLAGS   := -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs \
                   -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE \
                   -Werror=implicit-function-declaration -Werror=implicit-int \
                   -Wno-format-security \
                   -fanalyzer           \
                   -std=gnu89
```

When you run the kernel compilation process using make, the analyzer output will be on stderr.  I just redirected all stderr output from the `make` process to a `analyzer.log` file:

```bash
make -j 2> analyzer.log
```

After the kernel `make` process completes, you should have a decent amount of `analyzer.log` content to review.  Happy bug hunting!