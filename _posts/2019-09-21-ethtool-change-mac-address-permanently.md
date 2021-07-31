---
layout: post
title:  "Change MAC Address Permanently"
date:   2019-09-21 00:00:00 -0600
categories: networking mac-address ethtool
author: Nicholas Starke
---

It is well know that through the `ip` and `ifconfig` commands it is possible to change a MAC address temporarily, meaning the change will not persist across host reboots.

But what if you would like to change your MAC address in a more permanent fashion? Is there a way to, through software, permanently change your network interface card's MAC address?

It turns out the answer is yes, and the tool to do so is called `ethtool`.

## Ethtool
Ethtool comes pre-installed on many stock distributions of Linux, but can also be installed from your package manager of choice if necessary.

It is possible to read and write to the network interface card EEPROM using the `ethtool -e $DEVICE_NAME` (read) and `ethtool -E $DEVICE_NAME < $EEPROM_BACKUP.bin` (write).

## Change EEPROM

If you would like more granular control over the EEPROM contents, the `-E` argument has several sub-arguments, of the form

```
$ ethtool -E eth0 magic=$MAGIC_NUMBER offset=0x0 length=0x1 value=0xff
```

Here `$MAGIC_NUMBER` is dependent on the NIC firmware.  For example, the `e1000` line of NICs uses `vendor_id` and the `device_id` (both of which can be retrieved via `lspci -n`)
_Source: [https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/intel/e1000/e1000_ethtool.c#L480](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/intel/e1000/e1000_ethtool.c#L480)_

When you update the NIC EEPROM in this manner, the MAC address will persist across host reboots.  

**NB: Some NICs do not support EEPROM write. Use** `ethtool -i` **to see NIC capabilities**
_For Example, VMWARE does not allow write access to the virtual NIC_