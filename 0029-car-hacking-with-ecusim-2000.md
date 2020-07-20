# Car hacking with ScanTool ECUSim 2000

Published: February 22, 2020

An upcoming project has me looking at car hacking at the moment.  I watched a great video ( [https://www.youtube.com/watch?v=nvxN5G21aBQ](https://www.youtube.com/watch?v=nvxN5G21aBQ) ) which caught me up to speed on the fundamentals.  There are a few other videos out there on introductory car hacking, but they all seem to revolve around the virtual can interface provided by `vcan`. I decided I didn't want to test virtually because then I wouldn't know how to work with the actual connection hardware.  At the same time, being a beginner, I DID NOT want to plug into my personal vehicle's ODB2 port.

I was looking for something between `vcan` and a real car.  A little googling led me to the ScanTools ECUSim 2000:
[https://www.amazon.com/OBDLink-ScanTool-ECUsim-Simulator-Development/dp/B008NAH6WE](https://www.amazon.com/OBDLink-ScanTool-ECUsim-Simulator-Development/dp/B008NAH6WE)

This board simulates a car.  It has a ODB2 port for interfacing just like one would do with a real car, and then a serial console for manipulating the board.

The first thing I will note is that at the $200 price-point, you do not get much in terms of software features.  There are a lot of additional features that you can upgrade to by buying firmware packages, but they tend to add up in cost fairly quickly.  If you have the money, I would recommend the ECUSIM 5100; it has all the bells and whistles and is fully configurable and comes with a nice chassis.  However, at about $1500, it isn't cheap.

For my purposes, the $200 entry-level board was enough to get started with the hardware process of connecting to a car and transmitting and receiving data from it.

I also purchased `ScanTool OBDLink SX USB` [https://www.amazon.com/ScanTool-OBDLink-USB-Professional-Diagnostics/dp/B005ZWM0R4](https://www.amazon.com/ScanTool-OBDLink-USB-Professional-Diagnostics/dp/B005ZWM0R4) .  This is a dongle made by the same company as the board, and it works well with the sclan0 driver.

## Setup
I used a Raspberry Pi 4 (2GB RAM) as the main CPU for this project.  To get the Pi ready for CAN Bus hacking, I had to do some setup:

```bash
$ sudo usermod -aG dialout $USER
# apt install can-utils picocom
```

The first command adds the current user to the `dialout` group which will be then allow you to interact with serial devices without running everything as root.  **After you run this command, you must log out and log back in before the group change takes affect.**
The second command installs the `can-utils` package, which contains `slcand` amongst other utility programs we will need.

The reason I decided to use a Pi for the main hardware host is because I anticipate eventually wanting to "go mobile" with my setup; meaning I might want to plug the ODB2 dongle into my car at some point **strictly to record** driving sessions.  

## Configuring the ECUSim 2000

Once the Pi was set up. I plugged in the power and then attached the USB cable from the board to a raspberry pi.  The Pi recognized the USB connection as a serial device, assigning it to `/dev/ttyUSB1`. The baud rate is 115200, so I tried using `screen /dev/ttyUSB1 115200` to connect to the device.  When I did so, I received output like this:

```
>AN Baud Rate               500.0 kbps    EA 3
```

Only the last line of the previously run command output was showing up, and it was the top line in screen.  It turns out that the serial console on the board only outputs return carriages, and not return carriage/new line combinations.  Screen has no way of handling this situation unfortunately, so I ended up using `picocom` with the following command line arguments:

```
$ picocom -b 115200 --imap crcrlf /dev/ttyUSB1
```

The `--imap crcrlf` replaces all carriage returns with carriage return/new line combinations.  After interacting with the serial console this way, I can see sane input and output:

```
>SPI
OBD Protocol               ISO 15765-4
CAN ID Type                     29 bit
CAN Baud Rate               500.0 kbps

>
```

The next command I ran was:

```
>SOMM on
OK

>
```

This enables CAN Bus traffic monitoring on the board.  We will now be able to observe the CAN traffic coming and going from the board.


## Configuring the USB Dongle

The aforementioned dongle doesn't work as a raw can interface, so we must use the `slcand` driver.  The dongle actually apprears as a serial interface similar to the board serial interface, only the dongle serial interface doesn't respond to commands.  The donge runs at a baud rate of 115200, so we create the can0 interface by running the following command:

```bash
# slcand -o -s6 -S 115200 /dev/ttyUSB2 can0 
```

Where `/dev/ttyUSB2` is the block device for the serial connection that the dongle exposes.

Next, we need to bring the CAN interface up:

```bash
# ip link set up can0
```

At this point I received some errors when trying to generate packets with `cangen`:

```bash
# cangen -I i -D 0000 -g S can0
write: No buffer space available
```

The solution, found [here](https://stackoverflow.com/questions/40424433/write-no-buffer-space-available-socket-can-linux-can):

```bash
# ifconfig can0 txqueuelen 1000
```

A this point we can run the `cangen` command again and we should see traffic coming through on the board serial connection:

Pi: 
```bash
# cangen -I i -D 0000 -g S can0
```

And we would see something like this via the board serial console:

```
[...]
Rx: 18DB33F1 61 A1 00                 @ 4108210 ms
Rx: 18DB33F1 10                       @ 4108221 ms
Rx: 18DB33F1 00                       @ 4108223 ms
Rx: 18DB33F1 62 62 00 00              @ 4108226 ms
Rx: 18DB33F1 30 00 00                 @ 4108233 ms
Rx: 18DB33F1 62 C3 00 00 00           @ 4108236 ms
Rx: 18DB33F1 62 E4 00 00 00 00        @ 4108239 ms
Rx: 18DB33F1 00 00 00 00 00           @ 4108242 ms
Rx: 18DB33F1 50 00 00 00 00           @ 4108250 ms
Rx: 18DB33F1 00 00 00 00 00 00 00     @ 4108253 ms
Rx: 18DB33F1 00 00 00 00 00           @ 4108256 ms
Rx: 18DB33F1 00 00                    @ 4108259 ms
Rx: 18DB33F1 63 E0                    @ 4108261 ms
Rx: 18DB33F1 00 00 00                 @ 4108264 ms
Rx: 18DB33F1 31 00                    @ 4108266 ms
Rx: 18DB33F1 64 55 00 00 00 00 00     @ 4108269 ms
Rx: 18DB33F1 64 74 00 00 00 00        @ 4108273 ms
Rx: 18DB33F1 40 00 00 00              @ 4108284 ms
Rx: 18DB33F1 51 60 00 00 00 00 00     @ 4108287 ms
Rx: 18DB33F1 00 00 00 00              @ 4108290 ms
Rx: 18DB33F1 65 54 00 00 00 00        @ 4108293 ms
Rx: 18DB33F1 20 00                    @ 4108309 ms
Rx: 18DB33F1 66 10                    @ 4108312 ms
Rx: 18DB33F1 00 00 00                 @ 4108314 ms
Rx: 18DB33F1 00 00                    @ 4108317 ms
Rx: 18DB33F1 30 00 00                 @ 4108321 ms
Rx: 18DB33F1 A2 00 00                 @ 4108324 ms
Rx: 18DB33F1 01 00                    @ 4112772 ms
Tx: 18DAF110 06 41 00 BE 1B 30 13     @ 4112773 ms
Tx: 18DAF118 06 41 00 88 18 00 10     @ 4112774 ms
Tx: 18DAF128 06 41 00 80 08 00 10     @ 4112776 ms
Rx: 18DB33F1 A2 00 00                 @ 4112926 ms
[...]
```

## Other notes

The ECUsim 2000 Command reference can be found [here](https://www.scantool.net/static/documentation/ecusim/ecusim-pm.pdf).  However, keep in mind that at the $200 version, most of the commands in that document will not work!

The ECUSim 2000 User Manual can be found [here](https://www.scantool.net/scantool/downloads/101/ecusim_2000-ug.pdf).  It has some useful information in it for first time configuration.

ScanTool also makes a linux utility called `scantool` that provides a GUI for interfaces with ODB2 connections, and just like a real car, it will work with the board.  You can install this free tool by running:

```bash
# apt install scantool
```

Technical information surrounding the limitations of the $200 model can be found in [this amazon review](https://www.amazon.com/gp/customer-reviews/R1EO3VAMV8R818/ref=cm_cr_dp_d_rvw_ttl?ie=UTF8&ASIN=B008NAH6WE).

[Back](https://nstarke.github.io/)
