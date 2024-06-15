---
layout: posts
title: "Chat GPT 4o vs. Codestral for Linux Kernel Modules"
date: 2024-06-15 00:00:00 -0600
categories: generativeai
author: Nicholas Starke
---

## Introduction

I've been working a lot with locally hosted generative AI using [Text Generation WebUI](https://github.com/oobabooga/text-generation-webui) and decided to do an experiment to compare the results of OpenAI hosted [ChatGPT 4o](https://openai.com/chatgpt/) with [Codestral](https://mistral.ai/news/codestral/) ([GGUF Version](https://huggingface.co/MaziyarPanahi/Codestral-22B-v0.1-GGUF)) for generating Linux Kernel Module C Code.

I submitted the same prompt to both services and then tried to compile and inject the Kernel Module in Ubuntu 24.04/22.04 Server hosts. The results are below.

## Specs for Local LLM Server

I used a high-end gaming PC to run the Codestral-GGUF model.  Specs are:

* i9-14900f CPU
* 192GB DDR5 RAM
* 1x 4090 GPU
* 2 TB NVM.e drive

## The Prompt

The prompt I chose to use i:

```
generate the source code and makefile for a linux kernel module that replaces the read system call handler with a handler that returns the string "starkeblog" for all file reads.
``` 

## Analysis

The first obvious difference is the time it takes to render the output. The ChatGPT server generated the output in less than 5 seconds whereas the locally hosted Codestral-GGUF took about 15 minutes. 

Neither model was able to generate C code that compiled and linked properly.  It's worth noting that the `sys_close` symbol is not exported in 5.x and 6.x kernel versions. Both outputs relied on the `sys_close` symbol denoting the first system call in the system call table. I'm sure the code could be adjusted to look for the proper first system call symbol, but the user would need to know beforehand what that symbol is.

I did like how the ChatGPT response included code to enable writes to the system call table.  The Codestral output did not include these lines.

In the end, neither technology produces useable results on recent kernel versions, but they do give a decent idea of how to work with syscall hooking generally speaking, and may even work in older versions of the Linux Kernel.

## ChatGPT 4o Output

Creating a Linux kernel module to replace the read system call handler is an advanced task that involves modifying kernel behavior. This example will demonstrate how to replace the read system call to always return the string "starkeblog" for all file reads.

### Warning
Modifying system calls is inherently risky and can make the system unstable or unusable. Use this code at your own risk, preferably in a virtual machine or a controlled environment.

### Kernel Module Code

First, create a file named `syscall_replace.c` with the following content:

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/syscalls.h>
#include <linux/uaccess.h>
#include <linux/fs.h>
#include <linux/slab.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("OpenAI ChatGPT");
MODULE_DESCRIPTION("Kernel module to replace read system call handler with a custom handler");
MODULE_VERSION("1.0");

#define MY_STRING "starkeblog"
#define MY_STRING_LEN (sizeof(MY_STRING) - 1)

static asmlinkage ssize_t (*original_sys_read)(unsigned int, char __user *, size_t);

static asmlinkage ssize_t custom_sys_read(unsigned int fd, char __user *buf, size_t count) {
    ssize_t ret;
    
    if (count < MY_STRING_LEN) {
        return -EINVAL; // Invalid argument if buffer is too small
    }

    if (copy_to_user(buf, MY_STRING, MY_STRING_LEN)) {
        return -EFAULT; // Bad address error if copy_to_user fails
    }

    return MY_STRING_LEN;
}

static unsigned long **find_sys_call_table(void) {
    unsigned long offset;
    unsigned long **sct;

    for (offset = PAGE_OFFSET; offset < ULLONG_MAX; offset += sizeof(void *)) {
        sct = (unsigned long **)offset;

        if (sct[__NR_close] == (unsigned long *) sys_close)
            return sct;
    }
    return NULL;
}

static unsigned long **sys_call_table;

static int __init syscall_replace_init(void) {
    if (!(sys_call_table = find_sys_call_table()))
        return -1;

    write_cr0(read_cr0() & (~0x10000));
    original_sys_read = (void *)sys_call_table[__NR_read];
    sys_call_table[__NR_read] = (unsigned long *)custom_sys_read;
    write_cr0(read_cr0() | 0x10000);

    printk(KERN_INFO "Custom read system call loaded.\n");
    return 0;
}

static void __exit syscall_replace_exit(void) {
    if (!sys_call_table)
        return;

    write_cr0(read_cr0() & (~0x10000));
    sys_call_table[__NR_read] = (unsigned long *)original_sys_read;
    write_cr0(read_cr0() | 0x10000);

    printk(KERN_INFO "Custom read system call unloaded.\n");
}

module_init(syscall_replace_init);
module_exit(syscall_replace_exit);
```

### Makefile

Next, create a file named `Makefile` in the same directory with the following content:

```makefile
obj-m += syscall_replace.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### Compiling and Testing

1. **Compile the Kernel Module**:
   Open a terminal in the directory containing `syscall_replace.c` and `Makefile`, then run:
   ```sh
   make
   ```

2. **Insert the Kernel Module**:
   After compiling, insert the module with:
   ```sh
   sudo insmod syscall_replace.ko
   ```

3. **Test the Module**:
   Open a file and try to read from it. You should see "starkeblog" instead of the actual file content.

4. **Remove the Kernel Module**:
   To remove the module and restore the original read system call, run:
   ```sh
   sudo rmmod syscall_replace
   ```

### Notes
- This code is for educational purposes and should be used with caution.
- Always backup your data and test in a non-critical environment.
- The kernel version and configuration may affect the success of this module. Adjustments might be necessary.

## Codestral Output

Here is an example of a simple Linux kernel module that replaces the system call handler for the 'read' system call. This module is named 'read_override.c' and can be compiled with a makefile. Note that this is a very basic example and does not handle errors or edge cases. It's also important to note that modifying the behavior of the 'read' system call can have serious consequences, so please use this code responsibly.

```c
// read_override.c
#include <linux/module.h>
#include <linux/unistd.h>
#include <asm/uaccess.h>
#include <linux/uaccess.h>
#include <linux/sched.h>

asmlinkage ssize_t (*original_read)(struct file *, char __user *, size_t, loff_t *);

asmlinkage ssize_t custom_read(struct file *file, char __user *buf, size_t count, loff_t *offset)
{
    const char *str = "starkeblog";
    size_t len = strlen(str);

    if (copy_to_user(buf, str, len))
        return -EFAULT;

    return len;
}

static int __init init_module(void)
{
    original_read = sys_call_table[__NR_read];
    sys_call_table[__NR_read] = custom_read;
    return 0;
}

static void __exit exit_module(void)
{
    sys_call_table[__NR_read] = original_read;
}

module_init(init_module);
module_exit(exit_module);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Read override module");
```

Here is a simple makefile to compile this module:

```makefile
# Makefile
obj-m += read_override.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

You can compile this module using the following commands:

```bash
$ make
$ sudo insmod read_override.ko
```

And to remove the module:

```bash
$ sudo rmmod read_override
$ make clean
```

## ChatGPT 4o Make Output

```sh
make -C /lib/modules/6.8.0-35-generic/build M=/home/nick/chatgpt modules
make[1]: Entering directory '/usr/src/linux-headers-6.8.0-35-generic'
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: x86_64-linux-gnu-gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0
  You are using:           gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0
  CC [M]  /home/nick/chatgpt/syscall_replace.o
/home/nick/chatgpt/syscall_replace.c: In function ‘custom_sys_read’:
/home/nick/chatgpt/syscall_replace.c:19:13: warning: unused variable ‘ret’ [-Wunused-variable]
   19 |     ssize_t ret;
      |             ^~~
/home/nick/chatgpt/syscall_replace.c: In function ‘find_sys_call_table’:
/home/nick/chatgpt/syscall_replace.c:39:50: error: ‘sys_close’ undeclared (first use in this function)
   39 |         if (sct[__NR_close] == (unsigned long *) sys_close)
      |                                                  ^~~~~~~~~
/home/nick/chatgpt/syscall_replace.c:39:50: note: each undeclared identifier is reported only once for each function it appears in
make[3]: *** [scripts/Makefile.build:243: /home/nick/chatgpt/syscall_replace.o] Error 1
make[2]: *** [/usr/src/linux-headers-6.8.0-35-generic/Makefile:1926: /home/nick/chatgpt] Error 2
make[1]: *** [Makefile:240: __sub-make] Error 2
make[1]: Leaving directory '/usr/src/linux-headers-6.8.0-35-generic'
make: *** [Makefile:4: all] Error 2
```

## Codestral Make Output

```sh
make -C /lib/modules/6.8.0-35-generic/build M=/home/nick/codestral modules
make[1]: Entering directory '/usr/src/linux-headers-6.8.0-35-generic'
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: x86_64-linux-gnu-gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0
  You are using:           gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0
  CC [M]  /home/nick/codestral/read_override.o
/home/nick/codestral/read_override.c:10:20: warning: no previous prototype for ‘custom_read’ [-Wmissing-prototypes]
   10 | asmlinkage ssize_t custom_read(struct file *file, char __user *buf, size_t count, loff_t *offset)
      |                    ^~~~~~~~~~~
/home/nick/codestral/read_override.c:21:19: error: static declaration of ‘init_module’ follows non-static declaration
   21 | static int __init init_module(void)
      |                   ^~~~~~~~~~~
In file included from /home/nick/codestral/read_override.c:2:
./include/linux/module.h:78:12: note: previous declaration of ‘init_module’ with type ‘int(void)’
   78 | extern int init_module(void);
      |            ^~~~~~~~~~~
/home/nick/codestral/read_override.c: In function ‘init_module’:
/home/nick/codestral/read_override.c:23:21: error: ‘sys_call_table’ undeclared (first use in this function); did you mean ‘sys_desc_table’?
   23 |     original_read = sys_call_table[__NR_read];
      |                     ^~~~~~~~~~~~~~
      |                     sys_desc_table
/home/nick/codestral/read_override.c:23:21: note: each undeclared identifier is reported only once for each function it appears in
/home/nick/codestral/read_override.c: In function ‘exit_module’:
/home/nick/codestral/read_override.c:30:5: error: ‘sys_call_table’ undeclared (first use in this function); did you mean ‘sys_desc_table’?
   30 |     sys_call_table[__NR_read] = original_read;
      |     ^~~~~~~~~~~~~~
      |     sys_desc_table
/home/nick/codestral/read_override.c: At top level:
./include/linux/module.h:135:13: warning: ‘copy’ attribute ignored on a redeclaration of the referenced symbol [-Wattributes]
  135 |         int init_module(void) __copy(initfn)                    \
      |             ^~~~~~~~~~~
/home/nick/codestral/read_override.c:33:1: note: in expansion of macro ‘module_init’
   33 | module_init(init_module);
      | ^~~~~~~~~~~
/home/nick/codestral/read_override.c:21:19: note: previous declaration here
   21 | static int __init init_module(void)
      |                   ^~~~~~~~~~~
./include/linux/module.h:135:13: error: redefinition of ‘init_module’
  135 |         int init_module(void) __copy(initfn)                    \
      |             ^~~~~~~~~~~
/home/nick/codestral/read_override.c:33:1: note: in expansion of macro ‘module_init’
   33 | module_init(init_module);
      | ^~~~~~~~~~~
/home/nick/codestral/read_override.c:21:19: note: previous definition of ‘init_module’ with type ‘int(void)’
   21 | static int __init init_module(void)
      |                   ^~~~~~~~~~~
make[3]: *** [scripts/Makefile.build:243: /home/nick/codestral/read_override.o] Error 1
make[2]: *** [/usr/src/linux-headers-6.8.0-35-generic/Makefile:1926: /home/nick/codestral] Error 2
make[1]: *** [Makefile:240: __sub-make] Error 2
make[1]: Leaving directory '/usr/src/linux-headers-6.8.0-35-generic'
make: *** [Makefile:5: all] Error 2
```