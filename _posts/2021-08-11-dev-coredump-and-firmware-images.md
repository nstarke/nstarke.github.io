---
layout: posts
title:  "Device Coredump and Firmware Images"
date:   2021-08-11 00:00:00 -0600
categories: firmware wifi linux kernel
author: Nicholas Starke
---

This blog post will document some research I did recently on WiFi firmware for the QCA6174 `ath10k` wifi chipset. Specifically I will discuss how I modified the Linux Kernel to dump out the contents of the internal RAM of this WiFi chipset.  Then we will discuss ways of generating this crash dump, detailing how and where the Linux Kernel stores device coredumps.

First of all, we will need the ability to build the Linux Kernel with custom CONFIG options set.  We want to set the following parameters in `.config`:

* `CONFIG_ATH10K_DEBUG=y`
* `CONFIG_ATH10K_DEBUGFS=y`
* `CONFIG_ATH10K_TRACING=y`

We also need to modify `drivers/net/wireless/ath/ath10k/coredump.c` file, making the following modifications:

```diff
diff --git a/drivers/net/wireless/ath/ath10k/coredump.c b/drivers/net/wireless/ath/ath10k/coredump.c
index 7eb72290a925..243c95052507 100644
--- a/drivers/net/wireless/ath/ath10k/coredump.c
+++ b/drivers/net/wireless/ath/ath10k/coredump.c
@@ -1448,12 +1448,11 @@ static u32 ath10k_coredump_get_ramdump_size(struct ath10k *ar)
 const struct ath10k_hw_mem_layout *ath10k_coredump_get_mem_layout(struct ath10k *ar)
 {
 	int i;
+	// if (!test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask))
+	// 	return NULL;
 
-	if (!test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask))
-		return NULL;
-
-	if (WARN_ON(ar->target_version == 0))
-		return NULL;
+	// if (WARN_ON(ar->target_version == 0))
+	// 	return NULL;
 
 	for (i = 0; i < ARRAY_SIZE(hw_mem_layouts); i++) {
 		if (ar->target_version == hw_mem_layouts[i].hw_id &&
@@ -1502,8 +1501,8 @@ static struct ath10k_dump_file_data *ath10k_coredump_build(struct ath10k *ar)
 		len += sizeof(*dump_tlv) + sizeof(*ce_hdr) +
 			CE_COUNT * sizeof(ce_hdr->entries[0]);
 
-	if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask))
-		len += sizeof(*dump_tlv) + crash_data->ramdump_buf_len;
+	//if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask))
+	len += sizeof(*dump_tlv) + crash_data->ramdump_buf_len;
 
 	sofar += hdr_len;
 
@@ -1572,15 +1571,16 @@ static struct ath10k_dump_file_data *ath10k_coredump_build(struct ath10k *ar)
 	}
 
 	/* Gather ram dump */
-	if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask)) {
+	//if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask)) {
 		dump_tlv = (struct ath10k_tlv_dump_data *)(buf + sofar);
 		dump_tlv->type = cpu_to_le32(ATH10K_FW_CRASH_DUMP_RAM_DATA);
 		dump_tlv->tlv_len = cpu_to_le32(crash_data->ramdump_buf_len);
-		if (crash_data->ramdump_buf_len) {
-			memcpy(dump_tlv->tlv_data, crash_data->ramdump_buf,
-			       crash_data->ramdump_buf_len);
-			sofar += sizeof(*dump_tlv) + crash_data->ramdump_buf_len;
-		}
+	// }
+
+	if (crash_data->ramdump_buf_len) {
+		memcpy(dump_tlv->tlv_data, crash_data->ramdump_buf,
+				crash_data->ramdump_buf_len);
+		sofar += sizeof(*dump_tlv) + crash_data->ramdump_buf_len;
 	}
 
 	mutex_unlock(&ar->dump_mutex);
@@ -1624,7 +1624,7 @@ int ath10k_coredump_register(struct ath10k *ar)
 {
 	struct ath10k_fw_crash_data *crash_data = ar->coredump.fw_crash_data;
 
-	if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask)) {
+	//if (test_bit(ATH10K_FW_CRASH_DUMP_RAM_DATA, &ath10k_coredump_mask)) {
 		crash_data->ramdump_buf_len = ath10k_coredump_get_ramdump_size(ar);
 
 		if (!crash_data->ramdump_buf_len)
@@ -1633,7 +1633,7 @@ int ath10k_coredump_register(struct ath10k *ar)
 		crash_data->ramdump_buf = vzalloc(crash_data->ramdump_buf_len);
 		if (!crash_data->ramdump_buf)
 			return -ENOMEM;
-	}
+	//}
 
 	return 0;
 }
```

These changes take out a few checks that prevent the full IRAM contents from being dumped out to device coredump.  The configuration parameter `ath10k_coredump_mask` is not writeable in the DEBUGFS, and the test condition is preventing the entirety of the internal RAM contents from being dumped as part of the device coredump.

Next, compile and install the new kernel and its corresponding modules.  We then boot into the newly compiled kernel.

At this point, we want to look around at the debug options available to us through the DEBUGFS.  Note that the DEBUGFS will not accept read or write operations while secure boot is enabled.  So if you are following along at home, hopefully you have some test equipment you can disable Secure Boot on.

There are three locations in the DEBUGFS which are interesting to us.  One is the kernel module parameters for the ath10k_core kernel module, the next is the actual debug interface for the wifi firmware itself, and the last is the dump location for `Device Coredumps`.

## Devce Coredumps

A feature has been added to the Linux Kernel that dumps out firmware-related information when device firmware crashes.  I couldn't find much information published about this feature other than here: [https://lwn.net/Articles/610887/](https://lwn.net/Articles/610887/).  When device firmware crashes, the dump file is created in a subdirectory of `/sys/class/devcoredump`. We'll look at a specific dump file that we create.

Let's look at the debug configuration options provided by the DEBUGFS interface first.  

```
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# ls
ani_enable            htt_stats_mask   reset_htt_stats
cal_data              mem_value        simulate_fw_crash
chip_id               nf_cal_period    spectral_bins
enable_extd_tx_stats  peer_stats       spectral_count
fw_checksums          pktlog_filter    spectral_scan0
fw_dbglog             ps_state_enable  spectral_scan_ctl
fw_reset_stats        quiet_period     sta_tid_stats_mask
fw_stats              reg_addr         tpc_stats
htt_max_amsdu_ampdu   reg_value        wmi_services
```

There are a few interesting nodes here:

```
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# cat fw_dbglog
0x               0 0
```

This expects two values, the first is a unsigned 64-bit integer and the second is a int value from 0-9.  The first parameter is a debug mask and the second is a "log level".  Values above 2 are not used for "log level".

We'll set the values to `0xffffffffffffffff` and `9`, respectively:

```
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# echo "0xffffffffffffffff 9" > fw_dbglog
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# cat fw_dbglog
0xffffffffffffffff 9
```

Also interesting is `simulate_fw_crash`, whose options are defined in `drivers/net/wireless/ath/ath10k/debug.c`: [https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/debug.c#L524](https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/debug.c#L524). We'll come back to this in a minute, but this is a way to simulate a firmware device crash.

```c++
const char buf[] =
		"To simulate firmware crash write one of the keywords to this file:\n"
		"`soft` - this will send WMI_FORCE_FW_HANG_ASSERT to firmware if FW supports that command.\n"
		"`hard` - this will send to firmware command with illegal parameters causing firmware crash.\n"
		"`assert` - this will send special illegal parameter to firmware to cause assert failure and crash.\n"
		"`hw-restart` - this will simply queue hw restart without fw/hw actually crashing.\n";
```

Next lets look at the kernel driver configuration settings that we can tweak:

```
root@ubuntu-laptop:/sys/module/ath10k_core/parameters# ls
coredump_mask  debug_mask   rawmode   uart_print
cryptmode      fw_diag_log  skip_otp
```

We need to set `debug_mask` to `0x22`.  We could also set this mask to `0xffffffff` if we want to see all the driver debug messages, but this will create a lot of `dmesg` output, so we just use `0x22`.  This will turn on  `ATH10K_DBG_BOOT` and `ATH10K_DBG_WMI` messages.  All error messages are defined here : [https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/debug.h#L20](https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/debug.h#L20)

`coredump_mask` is not writeable.

At this point we should be able to capture firmware boot messages.  Let's simulate a hardware restart and see what kind of dmesg output we receive:

```
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# echo "hw-restart" > simulate_fw_crash
```

dmesg output:

```
[...]
[ 2585.394544] ieee80211 phy0: Hardware restart was requested
[ 2585.394780] ath10k_pci 0000:01:00.0: boot hif power up
[ 2585.394851] ath10k_pci 0000:01:00.0: boot qca6174 chip reset
[ 2585.394854] ath10k_pci 0000:01:00.0: boot cold reset
[ 2585.447539] ath10k_pci 0000:01:00.0: boot cold reset complete
[ 2585.447545] ath10k_pci 0000:01:00.0: boot waiting target to initialise
[ 2585.447551] ath10k_pci 0000:01:00.0: boot target indicator 2
[ 2585.447560] ath10k_pci 0000:01:00.0: boot target initialised
[ 2585.447562] ath10k_pci 0000:01:00.0: boot warm reset
[ 2585.487699] ath10k_pci 0000:01:00.0: boot init ce src ring id 0 entries 16 base_addr ffff9152f7fff000
[ 2585.487731] ath10k_pci 0000:01:00.0: boot ce dest ring id 1 entries 512 base_addr ffff9152c111e000
[ 2585.487754] ath10k_pci 0000:01:00.0: boot ce dest ring id 2 entries 128 base_addr ffff9152c1100000
[ 2585.487776] ath10k_pci 0000:01:00.0: boot init ce src ring id 3 entries 32 base_addr ffff9152c1101000
[ 2585.487807] ath10k_pci 0000:01:00.0: boot init ce src ring id 4 entries 8192 base_addr ffff9152c1120000
[ 2585.487831] ath10k_pci 0000:01:00.0: boot init ce src ring id 7 entries 2 base_addr ffff9152c1102000
[ 2585.487851] ath10k_pci 0000:01:00.0: boot ce dest ring id 7 entries 2 base_addr ffff9152c1103000
[ 2585.487856] ath10k_pci 0000:01:00.0: boot waiting target to initialise
[...]
```

Now what happens when we do a `hard` simulation?

```
root@ubuntu-laptop:/sys/kernel/debug/ieee80211/phy0/ath10k# echo hard > simulate_fw_crash
```

Now we have data in `/sys/class/devcoredump/devcd1/data`.  This is a device coredump.  Because of our kernel modifications above, it should contain the full contents of the internal RAM sections for the wifi chipset.  A quick word of warning, the device coredumps are deleted by the kernel after five minutes, so if they contain something you want to look at later, please copy the `data` file to somewhere persistent.

For reference, this is what a device coredump looks like WITHOUT the kernel modifications:

```xxd
root@ubuntu-laptop:~# xxd dev_coredump.bin
00000000: 4154 4831 304b 2d46 572d 4455 4d50 0000  ATH10K-FW-DUMP..
00000010: 0003 0000 0100 0000 87a5 91f2 846a 9e46  .............j.F
00000020: 860b e205 389f add7 ff0a 3400 0000 0000  ....8.....4.....
00000030: 0000 0305 0100 0000 0000 0000 0000 0000  ................
00000040: 6502 0000 0300 0000 3f00 0000 3f00 0000  e.......?...?...
00000050: 5b08 0000 b211 9033 0200 0000 574c 414e  [......3....WLAN
00000060: 2e52 4d2e 342e 342e 312d 3030 3135 372d  .RM.4.4.1-00157-
00000070: 5143 4152 4d53 5750 5a2d 3100 0b8f 0861  QCARMSWPZ-1....a
00000080: 0000 0000 7441 0d04 0000 0000 0000 0000  ....tA..........
00000090: 352e 3131 2e32 3200 0000 0000 0000 0000  5.11.22.........
000000a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000100: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000110: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000120: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000130: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000140: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000150: 0000 0000 f000 0000 0000 0305 b315 0000  ................
00000160: e1f4 9800 315b 9500 e1f4 9800 3001 0600  ....1[......0...
00000170: 0500 0000 0100 0000 0850 0000 f40f 4300  .........P....C.
00000180: 0000 0000 1400 0000 0900 0000 0000 0000  ................
00000190: 6c2f 9500 772f 9500 c42c 9500 0d08 9100  l/..w/...,......
000001a0: 0000 0000 0d08 9100 e1f4 9840 78e8 4000  ...........@x.@.
000001b0: 0850 0000 5f00 0000 0ef9 9880 d8e8 4000  .P.._.........@.
000001c0: 4701 3000 e1f4 98c0 05b2 9a80 f8e8 4000  G.0...........@.
000001d0: 0000 0000 0850 0000 6858 9a80 48e9 4000  .....P..hX..H.@.
000001e0: b000 4200 9049 4300 bf4f 9a80 88e9 4000  ..B..IC..O....@.
000001f0: ace9 4000 7cd5 4200 52d2 9180 a8e9 4000  ..@.|.B.R.....@.
00000200: 0200 0000 0100 0000 557a 9f80 58ea 4000  ........Uz..X.@.
00000210: 44d9 4300 28d9 4200 023b 9f80 78ea 4000  D.C.(.B..;..x.@.
00000220: 44d9 4300 0100 0000 1012 9180 c8ea 4000  D.C...........@.
00000230: 1000 0000 d041 4000 5411 9180 28eb 4000  .....A@.T...(.@.
00000240: 0000 4000 0000 0000 0100 0000 b000 0000  ..@.............
00000250: 0800 0000 0000 0000 0000 0000 0000 0000  ................
00000260: 0044 0300 0e00 0000 0e00 0000 0300 0000  .D..............
00000270: 0300 0000 0048 0300 1f00 0000 1f00 0000  .....H..........
00000280: ec01 0000 ed01 0000 004c 0300 3900 0000  .........L..9...
00000290: 3900 0000 3800 0000 3900 0000 0050 0300  9...8...9....P..
000002a0: 1800 0000 1800 0000 1900 0000 1800 0000  ................
000002b0: 0054 0300 8109 0000 8109 0000 0200 0000  .T..............
000002c0: c200 0000 0058 0300 0000 0000 0000 0000  .....X..........
000002d0: 4000 0000 0000 0000 005c 0300 1e00 0000  @........\......
000002e0: 1e00 0000 1e00 0000 1e00 0000 0060 0300  .............`..
000002f0: 0100 0000 0100 0000 0100 0000 0100 0000  ................
```

Looking at the coredump structure [https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/coredump.h#L38](https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/coredump.h#L38) we can see that the first `0x150` bytes are part of the header data structure. The actual dump data begins at offset `0x150` leaving just `0x1b0` (432) bytes of data in the firmware dump.

With the kernel changes, the RAM dump is almost 2 Megabytes in size:

```
-rw-r--r-- 1 root root 2095976 Aug  3 13:59 data.bin
```

This firmware dump contains several component parts, which are documented here: 

https://github.com/torvalds/linux/blob/v5.11/drivers/net/wireless/ath/ath10k/coredump.c#L876

We can use this memory layout to build a Ghidra representation of the data from the device coredump, which I will cover in a subsequent blog post.