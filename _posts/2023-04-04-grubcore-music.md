---
layout: posts
title: "Grubcore - Music with Grub2"
date: 2023-04-04 00:00:00 -0600
categories: grub music bootloader
author: Nicholas Starke
---

## Introduction

I've been really into music recently.  I write music regularly (see my [bandcamp](https://nstarke.bandcamp.com/) if you're interested), and recently I was able to combine my love for writing weird music with my love for bootloaders, and thus this blog post was begotten.

## Play

Grub 2.06 has a command called [play](https://www.gnu.org/software/grub/manual/grub/grub.html#play). From the Grub2 docs:

>If the argument is a file name (see File name syntax), play the tune recorded in it. The file format is first the tempo as an unsigned 32bit little-endian number, then pairs of unsigned 16bit little-endian numbers for pitch and duration pairs. 
>
>If the arguments are a series of numbers, play the inline tune.
>
>The tempo is the base for all note durations. 60 gives a 1-second base, 120 gives a half-second base, etc. Pitches are Hz. Set pitch to 0 to produce a rest.

The Grub2 `play` command can also read the musical data from a file, passing the filename as the first argument:

```
play file | tempo [pitch1 duration1] [pitch2 duration2] …
```

If a filename is not specified, the first integer value is the tempo.

Sweet! So we can play tunes. I decided to use an emulated version of Grub2 in order to effectively capture the audio out without having to use a microphone. This blog post will describe how I went about setting up the Grub2 virtual instance and how I then generated some appropriately formatted numbers to achieve a semi-musical expression.

## Virtualization Set Up

As I started out, I decided to use an EFI-based x86_64 version of Grub2.  I made this decision because I knew I could easily create a Grub EFI application from the Grub2 source code and combine it with `OVMF` to perform BIOS/UEFI based booting with QEMU. The steps to create a Grub2 EFI application are outlined nicely in [this stackoverflow post](https://stackoverflow.com/questions/43872078/debug-grub2-efi-image-running-on-qemu), but I will provide them here for consistencies sake:

```bash
git clone https://git.savannah.gnu.org/git/grub.git
cd grub
./bootstrap
./configure --prefix=$(pwd)/local --with-platform=efi --target=x86_64
make
make install
./local/bin/grub-mkstandalone -O x86_64-efi -o bootx64.efi
mkdir -p fs/efi/boot
cp bootx64.efi fs/efi/boot
```
This sets up an EFI system partition (ESP) in the `fs` directory that we can then use with OVMF to boot into Grub2.

## OVMF

OVMF is an open source version of TianoCore that can be used with QEMU to achieve EFI-based booting.  OVMF can be installed through `apt` using:

```
# apt install ovmf
```

Which will install `OVMF.fd` to `/usr/share/ovmf/OVMF.fd`.

## QEMU

We can use QEMU now to emulate the necessary hardware to boot into Grub2.

```
qemu-system-x86_64 -bios /usr/share/ovmf/OVMF.fd -hda fat:rw:fs --soundhw pcspk -nographic -serial mon:stdio
```

Adding `--soundhw pcspk` is necessary to emulate a PC Speaker for our listening pleasure.

Which eventually launches into a Grub2 CLI:

```
BdsDxe: failed to load Boot0001 "UEFI QEMU DVD-ROM QM00003 " from PciRoot(0x0)/Pci(0x1,0x1)/Ata(Secondary,Master,0x0): Not Found                     GNU GRUB  version 2.11
BdsDxe: loading Boot0002 "UEFI QEMU HARDDISK QM00001 " from PciRoot(0x0)/Pci(0x1,0x1)/Ata(Primary,Master,0x0)
   Minimal BASH-like line editing is supported. For the first word, TAB   Pci(0x1,0x1)/Ata(Primary,Master,0x0)
   lists possible command completions. Anywhere else TAB lists possible
   device or file completions. To enable less(1)-like paging, "set
   pager=1".

grub>
```

## Playing Music

I used ChatGPT to generate musical expressions in the format that the grub2 `play` command expects

![](/images/04042023/chatgpt.png)

>Sure, here's the space-separated list of frequency in Hertz and length in seconds pairs for the theme song of the original Super Mario Brothers game with the length values multiplied by two:
>
>262 1.0 262 1.0 0 1.0 262 1.0 0 1.0 196 1.0 262 1.0 0 1.0 349 1.0 0 1.0 329 1.0 0 1.0 294 1.0 0 1.0 262 1.0 0 2.0 392 1.0 0 2.0 349 1.0 0 2.0 330 1.0 0 2.0 294 1.0 0 1.0 0 1.0 0 1.0 262 1.0 0 1.0 0 1.0 196 1.0 0 1.0 0 1.0 262 1.0 0 1.0 0 1.0 0 1.0 349 1.0 0 1.0 0 1.0 329 1.0 0 1.0 0 1.0 294 1.0 0 1.0 0 1.0 0 1.0 262 1.0 0 2.0

A little formatting results in the following commands:

```
play 480 262 1 262 1 0 1 262 1 0 1 196 1 262 1 0 1 349 1 0 1 329 1 0 1 294 1 0 1 262 1 0 2 392 1 0 2 349 1 0 2 330 1 0 2 294 1 0 1 0 1 0 1 262 1 0 1 0 1 196 1 0 1 0 1 262 1 0 1 0 1 0 1 349 1 0 1 0 1 329 1 0 1 0 1 294 1 0 1 0 1 0 1 262 1 0 2
```

An audio sample sounds like this:

{% include embed-audio.html src="/images/04042023/grubcore.wav" %}

_Note that while I instructed ChatGPT to use the original Super Mario Brothers Theme Song as the source material, the resulting melody sounds nothing like that particular song_

I was even able to run the emulated Grub2 instance, with audio out, on a Raspberry Pi 4. I can imagine that being useful for sonic installations. A small note for the Raspberry Pi 4, there is no built in PC Speaker and thus the virtualized instance requires plugging into the audio out / headphone jack to be able to physically hear the tune generated.  

And hence the new musical genre `grubcore` has been born!

## New Directions

I used OBS Studio to record the audio out of the process as a video and then stripped the audio out of the video using ffmpeg.  There are many other ways of capturing the audio output, such as using [Audacity](https://www.audacityteam.org/). 

Beyond the virtualized instances, these techniques can be used on bare metal.  For example, [this YouTube video](https://www.youtube.com/watch?v=srPqJc9B6Ts) demonstrates making music using Grub2 on bare metal.  Note that if you are going to go this way, the audio tones are emitted from the PC Speaker and not the Audio Out / Headphone jack, so you will need a microphone to capture the sound.

{% include embed-youtube.html src="https://www.youtube.com/watch?v=srPqJc9B6Ts" %}

All of this I believe opens up some interesting musical potential.  While trying to integrate a Grub2-based melody into a DAW setup is probably not possible, sampling the results could really be interesting approach. Both the bare metal and the virtualized versions offer new creative approaches for writing music. 