---
layout: posts
title:  "Backdooring Grub2"
date:   2022-08-07 00:00:00 -0600
categories: grub2 bootloader
author: Nicholas Starke
---

Earlier this year I worked on a project involving backdooring Grub2.  What I wanted to accomplish was to send the cleartext LUKS password to a remote server when the password is entered as part of the Grub2 boot process.  In the end I was able to make the desired modifications and have it function as intended.  

I committed my backdoor-enabled code here:

[https://github.com/nstarke/backdoored-grub2/commit/fa6c5aa77d1909b1e62eb04ce257e27f290c3499](https://github.com/nstarke/backdoored-grub2/commit/fa6c5aa77d1909b1e62eb04ce257e27f290c3499)

## Wait, is this a vulnerability?

No.

## Overview

I made a copy of the existing TCP/IP stack contained within Grub2 sources and modified it to meet my needs.  As it existed when I wrote the backdoor code, the TCP/IP stack was not sufficient to perform the actions I wanted to perform.  For example, there was no function to call to make arbitrary HTTP requests from elsewhere in source.  

There are three operations that the LUKS module must complete in order to obscond with the LUKS password:

1) Find the NIC

2) Obtain a IP Address from a DHCP server

3) Make the web request with the password hex encoded in the HTTP request query string.

## Attack Scenarios

How practical is this work to real-world attack scenarios? An attacker would need the ability to modify the grub EFI application, which assumes at least a root shell on the host.  Ideally, Secure Boot would be enabled and thereby detect custom modifications made to Grub2 before the shim passes control of execution over to Grub2.  This is a long winded way of saying Secure Boot should mitigate the threat from this type of attack.  Where I see this being the most useful for any real world attack is in virtualized environments where TPM features are not always available to the guest.  

## Testing

I tested this code using a Manjaro-based Virtual Machine running under Virtual Box.  I compiled the code with my modifications and then created an EFI executable using `grub-mkimage`.  

## Future work

The code I committed above uses a hardcoded IP addresses.  Future work could abstract that value out and make it configurable at compile time (or better yet, at run time!).  I'm pretty sure that the code will work with DNS hostnames, but I don't think it will work with HTTPS, which would also be a really great improvement to make at some point.