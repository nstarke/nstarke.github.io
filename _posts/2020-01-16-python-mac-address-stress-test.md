---
layout: post
title:  "Python MAC Address Stress Test Script"
date:   2020-01-16 00:00:00 -0600
categories: python mac-address stress arp-table
author: Nicholas Starke
---

This is a script that will fill up a MAC Address Table on a switch very quickly.  Useful for testing stressed switch scenarios.

```python
#!/usr/bin/env python

#
# This script is meant to assist in filling up a MAC ADDRESS Table on a switch
# This script reuqires scapy to be installed, and most likely will need to be
# run as root.  That means scapy will have to be installed for the root user
# in order for this script to work.
#
# Arguments:
# * Interface to send ARP packet on
# * MAX number of entries
#

import sys, threading
from scapy.all import *

IFACE = sys.argv[1]
MAX = int(sys.argv[2])
THREADS = 8

def doARP():
    for _ in range(MAX / THREADS):
        # some of this copped from https://stackoverflow.com/questions/1487389/python-scapy-mac-flooding-script
        pkt = Ether(src=RandMAC(), dst="ff:ff:ff:ff:ff:ff")/ARP(op=2, psrc="0.0.0.0",hwdst="ff:ff:ff:ff:Ff:ff")/Padding(load="A" * 18)
        sendp(pkt, iface=IFACE, verbose=0)


for _ in range(THREADS):
    t = threading.Thread(target=doARP, args=())
    t.start()
```