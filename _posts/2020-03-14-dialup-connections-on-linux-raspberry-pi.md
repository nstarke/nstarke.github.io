---
layout: post
title:  "Dial Up Connections on Linux"
date:   2020-03-14 00:00:00 -0600
categories: dialup raspberry-pi
author: Nicholas Starke
---

In this tutorial we will detail how to connect two linux hosts via 56k modems.  To do this we will use the following components:

* Two Raspberry Pi's 4 ($35-$55 / each)
* Two Conexant 56k Modems ( [https://www.amazon.com/gp/product/B008RZTJC0/](https://www.amazon.com/gp/product/B008RZTJC0/) | $17-$20 / each)
* a RJ-11 splitter( [https://www.amazon.com/Splitter-Duplex-line-Telephone-Adapter/dp/B07WNDZT6Y](https://www.amazon.com/Splitter-Duplex-line-Telephone-Adapter/dp/B07WNDZT6Y) | $8-$12)
* A Linksys SPA-2102 ( search ebay for `linksys SPA-2102` on ebay | $18-$25 ) 
* Two RJ-11 cables ( should come with the modems )

## Wiring up

We need to configure our gear.  
* Plug one USB Modem into each Pi
* Plug one end of the RJ-11 cable into each modem
* Plug the other end into the RJ-11 splitter
* Plug the RJ-11 splitter into the SPA-2102

## Hardware Configuration

The SPA-2102 serves as a "dial-tone generator" for the modems; that is to say the modem RJ-11 lines need voltage and the SPA-2102 supplies that voltage.  **You cannot plug the modems directly into each other.** The modems lack the ability to generate the current necessary for establishing commmunications between themselves.

## Software configuration

The modems will show up on the Pis as serial interfaces.  Run `dmesg` to see more information:

```
[    1.238294] usb 1-1.4: new full-speed USB device number 3 using xhci_hcd                               
[    1.390434] usb 1-1.4: New USB device found, idVendor=0572, idProduct=1340, bcdDevice= 1.00            
[    1.390478] usb 1-1.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3                        
[    1.390510] usb 1-1.4: Product: USB Modem                                                              
[    1.390534] usb 1-1.4: Manufacturer: Conexant                                                          
[    1.390557] usb 1-1.4: SerialNumber: 12345678 
```

On my Pis the modem appeared at `/dev/ttyACM0`, but your mileage may vary.  We will need to use PPP as the IP layer between the two modems.  We will need to create a "connect" script to pass to pppd so it knows how to handle the modem serial interface.  You can see generic instructions on how to do so here: [http://tldp.org/HOWTO/PPP-HOWTO/x1188.html](http://tldp.org/HOWTO/PPP-HOWTO/x1188.html).

The connect script I used for the client (connection initiator):

```sh
#!/bin/sh
#
# This is part 2 of the ppp-on script. It will perform the connection
# protocol for the desired connection.
#
/usr/sbin/chat -v                                                 \
        TIMEOUT         3                               \
        ABORT           '\nBUSY\r'                      \
        ABORT           '\nNO ANSWER\r'                 \
        ABORT           '\nRINGING\r\n\r\nRINGING\r'    \
        ''              \rAT                            \
        'OK-+++\c-OK'   ATH0                            \
        TIMEOUT         30                              \
        OK              ATD \
        CONNECT         ''                              \

```

The connect script I used for the server (connection receiver):

```sh
#!/bin/sh
#
# This is part 2 of the ppp-on script. It will perform the connection
# protocol for the desired connection.
#
/usr/sbin/chat -v                                                 \
        TIMEOUT         3                               \
        ABORT           '\nBUSY\r'                      \
        ABORT           '\nNO ANSWER\r'                 \
        ABORT           '\nRINGING\r\n\r\nRINGING\r'    \
        ''              \rAT                            \
        'OK-+++\c-OK'   ATH0                            \
        TIMEOUT         30                              \
        OK              ATA                  \
        CONNECT         ''                              \

```

The difference between these two scripts is the second to last line: `OK      ATA` on the receiver versus `OK      ATD` on the intiator.

Copy those scripts onto each Pi as `chat.sh`.  Make the `chat.sh` executable using `chmod +x chat.sh` and then you will be able to run the following commands as root:

Intiator
```sh
pppd /dev/ttyACM0 9600 noauth local lock defaultroute debug nodetach 172.16.1.1:172.16.1.2 ms-dns 8.8.8.8 connect ./chat.sh
```

Receiver:
```sh
pppd noauth local lock defaultroute debug nodetach /dev/ttyACM0 connect ./chat.sh
```

The path prefix to the `chat.sh` binary seems important, so you will need the `./` in front of `chat.sh` when you use it in the `pppd` command

You should see the ppp connection establish successfully, though it might take up to a minute for the IP link to come up.

## Looking under the hood at the modem connection

It is possible to send "modem commands" directy to the modem.  A list of semi-standardized commands is available here: [https://en.wikipedia.org/wiki/Hayes_command_set](https://en.wikipedia.org/wiki/Hayes_command_set)

If you desire to write commands directly to the modem, I recommend using the `cu` utility. Install with:

```sh
apt install cu
```

You can then establish AT communications sessions with:

Client:
```sh
cu -l /dev/ttyACM0
Connected.
ATD
CONNECT 9600

```

Server:
```sh
cu -l /dev/ttyACM0
Connected.
ATA
CONNECT 9600
```

Where `ATD` is the "AT" command run on the client and `ATA` is the "AT" command run on the server.

Once you have an "AT" connection established, any input from one terminal will be displayed in the opposite terminal. This is the basis for IP communication over 56k modem.

To exit cu, from the cu prompt run:

```
~.
```

## Why would anyone want to do this

Fax machines are still commonly used, these techniques could be used to fuzz fax modem connections.