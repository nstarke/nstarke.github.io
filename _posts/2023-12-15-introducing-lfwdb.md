---
layout: posts
title: "Introducing Linux Firmware DB"
date: 2023-12-15 00:00:00 -0600
categories: lfwdb
author: Nicholas Starke
---

# Introduction

## Why Build a Firmware Database?

Good question. I'm a big fan of the Linux operating system and the open source ecosystem that surrounds it, but there is a large proprietary dependency in that ecosystem: [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/). 

Linux-firmware is a git repository full of proprietary binary files used for component firmware.  These blobs are loaded onto component internal RAM at operating system boot and serve to configure the runtime behavior of the component. There is little to no visibility in terms of what these blobs do. I'm not opposed to proprietary software; it has it's place.  However, with the emphasis on open source in the Linux project, the proprietary nature of `linux-firmware` makes it stand out to me.

The Linux Firmware Database aims to shine light on this unknown territory.

## The Dataset

The [Linux Firmware Database](https://lfwdb.com/) (henceforth referred to as `lfwdb`) contains a dataset of the "guessed" Instruction Set Architecture of each firmware image, as "guessed" by [cpu_rec](https://github.com/airbus-seclab/cpu_rec).  Additionally, I included the `file` command output and the Shannon entropy for each file.  At this point, the dataset is not complete as only about a dozen of the ~50 git tags have been processed. However, I estimate at least 90% of the files in the repository have been analyzed at this point. Unfortunately processing each tag independently results in a lot of duplication of compute effort as most of the files do not change.  I decided to process the full file list for every tag because I wanted the end result dataset to be stateless so it could be easily shared and displayed.

## How do I work with the data?

The raw data is contained entirely in a [git repository](https://github.com/nstarke/linux-firmware-db/) if you wish to work with the raw data.  The most useful data is the JSON files produced for each git tag.  

If you're looking for a way to explore the data more interactively, I built a small VueJS application for interacting with the data via the web.  This application and all of the data are deployed on the web at [https://lfwdb.com/](https://lfwdb.com/).  There is no server side component to this project; all data is static content distributed via web or git.

## How was the dataset built?

I wrote a lot of bash and python scripts for processing the `cpu_rec` results and combining it with other utility functions such as a SHA256 checksum, `file` command output, and Shannon entropy value.  All of these scripts are included in the `lfwdb` git repository.

The process starts with generating `cpu_rec` results for every file in `linux-firmware` for a given git tag.  This process takes about 2 hour for early tags and about 4 hours on the newest tags, using a 16-core Intel i5 NUC running Ubuntu 23.10.  The reason for the vast difference is that the initial fileset contained about 1900 files whereas the most recent tag's fileset contains nearly 3200 files.  

After the `cpu_rec` results have been generated in `txt` format, the txt file is read line for line and shaped into a `csv` file.  This is where the extra utility functions are executed and their results appended to the dataset. 

Once we have a `csv` file, we can map that into JSON fairly easily using python.  Using this final JSON output, the diassembly scripts run and dump their contents out to be included as part of the dataset.  Disassembly files are mapped to repository files by SHA256 hash of the file at a given git tag.  The SHA256 digest is included in the JSON for each file result and also in the filename of the disassembly file.

## Disassembly

We can use disassembly as a barometer to judge how accurate the `cpu_rec` output ISA actually is.

I attempted to provide disassembly with [radare2](https://rada.re/n/) for the most common ISAs.  Generating disassembly allows us to more precisely determine if the cpu_rec-derived ISA guess is correct, as incorrect disassembly ISA will lead to meaningless disassembly output.  In the case of correct disassembly ISA, the disassembly output will be more meaningful and provide researchers insight into the inner workings of these proprietary blobs. In most cases, however, I believe the disassembly output will help guide researchers towards which blobs to focus on with more powerful dissassembly and decompiler tools that are not web-based.  TXT-based disassembly is great, but nothing beats `{IDA, BinaryNinja, Ghidra, Radare2, etc}` when it comes to interactively navigating a flat binary file with no symbols, sections, or other structured metadata.

It is worth noting that the disassembly output may not be accurate, as it seems `radare2` defaults to Intel x64 if it doesn't think the specifed ISA matches the binary files' actual ISA.  

# Preliminary Results

## Contents of the "linux-firmware" git repository

One of the things I learned early on in this project is that the linux-firmware directory contains a lot of files that are not firmware.  Indeed going further back in time, there's even source code in this repository for some of the firmware.  This shouldn't come as a total shock, as there are instances of vendors making their proprietary firmware blobs open source (see [https://github.com/qca/open-ath9k-htc-firmware](https://github.com/qca/open-ath9k-htc-firmware) for two such examples).  There are all kinds of ASCII/UTF-8 text files in the linux-firmware repository and these files are not component firmware. 

Some of the binary files are not firmware, but things like NVRAM data banks and other configuration data.  These types of files are binary data, but they do not contain CPU instructions, yet they are still downloaded to the device at boot.  My guess is these blobs allow vendors to configure the component but not actually tell the component CPU what instructions to execute.  In these cases, changing the configuration data might yield in "unlocking" features or debug settings that the vendor does not wish to be enabled by default. 

The file command output is useful for further narrowing valid firmware blob results.  Most firmware is listed as the file command output `data`, but there are a few interesting deviations from that.  One such interesting deviation is a set of encrypted zip files in later tags (more on this later).

## Shannon Entropy

The dataset includes the Shannon Entropy value for each file in the `linux-firmware` repository. Generally speaking, anything above a Shannon Entropy value of 7.5 is considered random. Through trial and error I have made a conservative estimate that anything above 7.0 is random enough to not be structured binary instructions.  This helps us further narrow down the valid firmware result set as one benefit, but it teaches us a few other interesting things about the repository files.

First of all, almost all of the > 7.0 files are marked as `Xtensa` by `cpu_rec`. This leads me to the conclusion that most `Xtensa` results from `cpu_rec` are mostly if not entirely false positives, as these high Shannon entropy value files seem too random to be valid programs.  To help drive that theory, I looked at the disassembly for these files and found that all of the `Xtensa`-flagged results have meaningless or highly incomplete disassembly.  Note that the `cpu_rec` seems to determine `XtensaEB` results quite accurately, as the open-source component firmware [repository](https://github.com/qca/open-ath9k-htc-firmware) I mentioned above uses the Big Endian Xtensa GNU toolchain during compilation (resulting in an `XtensaEB` blob output).

But if these high-entropy files are not programs, then what are they?  High-entropy values suggest these files are either compressed or encrypted. I initially suspected that most of these files would be compressed data.  I wrote a script that attempts to bruteforce decompressing these files (included in the project git repository), and have not yet discovered any that decompress to valid CPU instructions.  That part of the dataset is not complete, but I've processed enough files to know a large amount of them are not compressed data.

So it they are not compressed data, are they encrypted data?  I suspect the answer is yes, with my only doubt driven by how difficult key management must be at the component firmware level.  If these high entropy files are mostly encrypted data,at the very least, grouping the high entropy files tells us which vendors are encrypting their firmware. `Mellanox` seems to be one such vendor.  

If the files are indeed encrypted, to me that means the key is hard coded-into the component ROM and should be dumpable from a physical device.  This is one such focus area I'd like to explore in the future: given an encrypted component firmware image and a physical device for that image, can we find the encryption key on ROM and successfully decrypt the component firmware file?

## Code Signing?

A quick grep for the case-sensitive string `BEGIN` reveals a few `AMD` blobs with pgp signatures, but very little else that looks obviously like it is code-signed. A few `git log` messages indicate `Nvidia` may be signing some of their firmware as well.  This makes most of these blobs ripe for tampering.  

# Immediate Topics of Further Inquiry

The `git log` from the `linux-firmware` repository lists several dozen forks that various vendors have created in order to merge new component firmware files into the upstream repository. It will be interesting to analyze these forks for additional branches and files that didn't make it into upstream. 

Is it possible to use commodity password cracking tools such as [hashcat](https://hashcat.net/hashcat/) to crack the encrypted zip files we found and gain access to encrypted component firmware without knowing the password beforehand? I've looked into this briefly. Assuming the password is an 8-byte binary-maskable value (`?b?b?b?b?b?b?b?b`), hashcat cannot run because the keyspace is too large - at least on the compute resources available to me.  Successfully attacking these files will involve balancing the mask carefully and adjusting the mask over multiple workloads in order to find a valid key.  

Before we begin talking about reverse engineering the component firmware blobs, we should look at the string values contained in those blobs (using GNU bin-utils' `strings`) just to see if there is anything unexpected there.  I would be looking for something like "backdoor reverse shell connection successfully established" or [Coldplay lyrics](https://www.bleepingcomputer.com/news/technology/kingstons-ssd-firmware-has-coldplay-lyrics-hidden-within-it/).

I'd like to reverse engineer the ARM and MIPS component firmware files and look for interesting patterns. I'm sure digging into these programs, their functions, and their data, will produce interesting topics of their own to then go and run down.

Lastly, as mentioned before, I would like to look at tampering with the component firmware blobs, as well as the configuration data blobs, to see if we can't make them do unexpected things that might be advantageous.

# Conclusions

There are a lot of things this dataset can teach us; both about the data itself and about our tooling.  You can look forward to future blog posts regarding analysis of this dataset from me. If you have any questions or topics you'd like to see me write about here, as always, please reach out!

Also, if you're interested in contributing to the project, the VueJS application I wrote desperately needs some CSS love.  Pull requests of that nature would be most graciously accepted!
