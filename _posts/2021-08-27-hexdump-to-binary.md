---
layout: posts
title:  "Hexdump to Binary"
date:   2021-08-27 00:00:00 -0600
categories: hexdump binary linux
author: Nicholas Starke
---

This blog post is going to cover a little trick I developed recently that might be useful to others.  Let's say you have a hexdump of a binary.  For example, you create a hexdump using `xxd` that looks like this:

```
00000000: 6000 1c00 008e 0800 2222 1221 e000 0000  `......."".!....
00000010: 008e 0063 004e 57e0 0052 5201 0000 0000  ...c.NW..RR.....
00000020: c000 00e4 3112 fff7 02e7 3102 1600 12ff  ....1.....1.....
00000030: f615 fff6 16ff f7c0 30d7 2005 6601 6000  ........0. .f.`.
00000040: 2100 0000 0000 0000 0000 0000 0000 0000  !...............
```

How does one convert data in this format back to binary?  `xxd -r -p` doesn't handle the address column well in many cases, instead interpretting it as hex data into the final binary output.  For example, running `xxd -r -p` over the above content results in a binary in this format (hexdumping the output from `xxd -r -p`):

```
00000000: 0000 0000 6000 1c00 008e 0800 2222 1221  ....`......."".!
00000010: e000 0000 0000 0010 008e 0063 004e 57e0  ...........c.NW.
00000020: 0052 5201 0000 0000 0000 0020 c000 00e4  .RR........ ....
00000030: 3112 fff7 02e7 3102 1600 12ff 0000 0030  1.....1........0
00000040: f615 fff6 16ff f7c0 30d7 2005 6601 6000  ........0. .f.`.
00000050: 0000 0040 2100 0000 0000 0000 0000 0000  ...@!...........
00000060: 0000 0000                                ....
```

## Bash one-liner for converting hexdump back to binary

I looked extensively at tools that can convert a hexdump back to binary, and I could not find one that worked well.  Then I realized I could easily do this with `awk` and `xxd`.  Here is a one liner to convert hexdump back to binary:

```
$ awk -F' ' '{print $2$3$4$5$6$7$8$9}' input.txt | xxd -r -p > output.bin
```

The `awk` command splits each line on the space character as a delimiter.  Then it outputs the bytes columnns from `input.txt`.  Lastly, we pipe the output to `xxd -r -p` which can now easily convert the hex-only output back to binary! The output is redirected then to `output.bin`.