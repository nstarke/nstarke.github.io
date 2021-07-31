---
layout: posts
title:  "Firmware Archaeology II: Return of the Devshell"
date:   2021-06-08 00:00:00 -0600
categories: netgear firmware
author: Nicholas Starke
---

In my [last blog post](/netgear/firmware/2021/06/07/firmware-archaeology-netgear-gs110tpv2.html) I went through the GS108Tv2 / GS110TPv2 firwamre images.  These are ProSafe-family switches manufacturered by Netgear.  In this post we will look at the `devshell` we identitifed in the last post in order to determine any security impact of this `devshell` being available in production builds.

The `devshell` is available through the web interace of the switch at the URL path `/base/devshell.html`.  Both the GET request which retrieves the UI and the POST request which actually executes the supplied command require a valid session identifier in the form of a `SID` cookie.  This effectively means authentication is required to access the `devshell`.

Normally, this sort of developer shell should only be used in test environments and not compiled into the release builds.  However, most of the time these sorts of shells execute OS commands; the `devshell` we are considering here does not execute OS commands but instead runs a predetermined list of commands that the developers chose. It appears this is mainly a diagnostic tool and not an arbitrary command execution shell.  As such, does having this `devshell` available have any security impact on the device?

## Enumeration

Let's enumerate all the commands available through the `devshell` by analyzing the firmware. For this post we will focus on the latest firmware version, which is `5.4.2.33` as of this writing.

I can enumerate all the commands and their corresponding handler functions by doing a search for on of the commands in the Ghidra String search.  I chose `upTime`, but any command from the list we enumerated in the last blog post will work.

When I navigate to the `upTime` string reference, I see the following disassembly output:

```
                             PTR_s_cablediag_80ef1390                        XREF[1]:     FUN_803bf9c8:803bfa1c (R)   
        80ef1390 80  9e  ec  5c    addr       s_cablediag_809eec5c                             = "cablediag"
                             PTR_LAB_80ef1394                                XREF[1]:     FUN_803bf9c8:803bfa38 (R)   
        80ef1394 80  07  81  4c    addr       LAB_8007814c
                             PTR_DAT_80ef1398                                XREF[1]:     FUN_803bf9c8:803bfa1c (R)   
        80ef1398 80  9e  ec  68    addr       DAT_809eec68                                     = 63h    c
        80ef139c 80  3c  06  ac    addr       FUN_803c06ac
        80ef13a0 80  9e  ec  6c    addr       s_changeMallocDebug_809eec6c                     = "changeMallocDebug"
        80ef13a4 80  3b  e0  5c    addr       LAB_803be05c
        80ef13a8 80  9e  ec  80    addr       s_cliInfoDump_809eec80                           = "cliInfoDump"
        80ef13ac 80  36  76  68    addr       FUN_80367668
        80ef13b0 80  9e  ec  8c    addr       s_cnfgrDump_809eec8c                             = "cnfgrDump"
        80ef13b4 80  29  e8  e0    addr       LAB_8029e8e0
        80ef13b8 80  9e  ec  98    addr       s_configClear_809eec98                           = "configClear"
        80ef13bc 80  2f  38  88    addr       LAB_802f3888
        80ef13c0 80  9e  ec  a4    addr       s_configDump_809eeca4                            = "configDump"
        80ef13c4 80  2e  8c  34    addr       LAB_802e8c34
        80ef13c8 80  9e  ec  b0    addr       s_configSave_809eecb0                            = "configSave"
        80ef13cc 80  2f  38  b0    addr       FUN_802f38b0
        80ef13d0 80  9e  ec  bc    addr       s_crashPrint_809eecbc                            = "crashPrint"
        80ef13d4 80  3b  c4  5c    addr       FUN_803bc45c
        80ef13d8 80  9e  ec  c8    addr       s_flashLogPrint_809eecc8                         = "flashLogPrint"
        80ef13dc 80  3b  c4  84    addr       FUN_803bc484
        80ef13e0 80  9e  ec  d8    addr       s_debugEwaSessionTableDump_809eecd8              = "debugEwaSessionTableDump"
        80ef13e4 80  3a  6e  5c    addr       FUN_803a6e5c
        80ef13e8 80  9e  ec  f4    addr       s_debugPolicyGroup_809eecf4                      = "debugPolicyGroup"
        80ef13ec 80  05  aa  c8    addr       LAB_8005aac8
        80ef13f0 80  9e  ed  08    addr       s_debugPolicyTable_809eed08                      = "debugPolicyTable"
        80ef13f4 80  05  4e  ec    addr       LAB_80054eec
        80ef13f8 80  9e  ed  1c    addr       s_debugTftp_809eed1c                             = "debugTftp"
        80ef13fc 80  3c  f3  9c    addr       LAB_803cf39c
        80ef1400 80  9e  ed  28    addr       DAT_809eed28                                     = 64h    d
        80ef1404 80  3c  00  08    addr       FUN_803c0008
        80ef1408 80  9e  ed  2c    addr       DAT_809eed2c                                     = 64h    d
        80ef140c 80  04  d5  4c    addr       FUN_8004d54c
        80ef1410 80  9e  ed  30    addr       DAT_809eed30                                     = 64h    d
        80ef1414 80  3b  fe  74    addr       LAB_803bfe74
        80ef1418 80  9e  ed  34    addr       s_dnsCfgDump_809eed34                            = "dnsCfgDump"
        80ef141c 80  2b  02  10    addr       FUN_802b0210
        80ef1420 80  9e  ed  40    addr       s_dnsClientTraceModeSet_809eed40                 = "dnsClientTraceModeSet"
        80ef1424 80  2b  01  f0    addr       LAB_802b01f0
        80ef1428 80  9e  ed  58    addr       s_dnsCountersDump_809eed58                       = "dnsCountersDump"
        80ef142c 80  2b  05  5c    addr       LAB_802b055c
        80ef1430 80  9e  ed  68    addr       s_dnsOperDataDump_809eed68                       = "dnsOperDataDump"
        80ef1434 80  2b  0c  b0    addr       LAB_802b0cb0
        80ef1438 80  9e  ed  78    addr       DAT_809eed78                                     = 64h    d
        80ef143c 80  2b  75  ac    addr       FUN_802b75ac
        80ef1440 80  9e  ed  80    addr       s_dtlDebug_809eed80                              = "dtlDebug"
        80ef1444 80  3b  98  08    addr       LAB_803b9808
        80ef1448 80  9e  ed  8c    addr       s_dtlEndStats_809eed8c                           = "dtlEndStats"
        80ef144c 80  3b  aa  b4    addr       FUN_803baab4
        80ef1450 80  9e  ed  98    addr       s_dtlMacToPortShow_809eed98                      = "dtlMacToPortShow"
        80ef1454 80  3b  bf  64    addr       LAB_803bbf64
        80ef1458 80  9e  ed  ac    addr       s_dtlNetDebugSet_809eedac                        = "dtlNetDebugSet"
        80ef145c 80  3b  94  0c    addr       LAB_803b940c
        80ef1460 80  9e  ed  bc    addr       s_dumpFdbStats_809eedbc                          = "dumpFdbStats"
        80ef1464 80  49  f8  84    addr       FUN_8049f884
        80ef1468 80  9e  ed  cc    addr       s_dumpMemory_809eedcc                            = "dumpMemory"
        80ef146c 80  3c  92  58    addr       FUN_803c9258
        80ef1470 80  9e  ed  d8    addr       s_dump_snmp_engine_id_809eedd8                   = "dump_snmp_engine_id"
        80ef1474 80  45  83  60    addr       LAB_80458360
        80ef1478 80  9e  ed  ec    addr       s_ecos_net_stats_809eedec                        = "ecos_net_stats"
        80ef147c 80  3c  5b  80    addr       FUN_803c5b80
        80ef1480 80  9e  ed  fc    addr       s_emwebDebugAccessSet_809eedfc                   = "emwebDebugAccessSet"
        80ef1484 80  3a  7e  c4    addr       FUN_803a7ec4
        80ef1488 80  9e  ee  10    addr       s_getBoardId_809eee10                            = "getBoardId"
        80ef148c 80  08  b3  70    addr       FUN_8008b370
        80ef1490 80  9e  ee  1c    addr       s_hapiBroadDebugPkt_809eee1c                     = "hapiBroadDebugPkt"
        80ef1494 80  04  6c  b8    addr       LAB_80046cb8
        80ef1498 80  9e  ee  30    addr       s_hapiBroadDebugPktFilterGet_809eee30            = "hapiBroadDebugPktFilterGet"
        80ef149c 80  04  6c  ec    addr       FUN_80046cec
        80ef14a0 80  9e  ee  4c    addr       s_hapiBroadDebugPktFilterSet_809eee4c            = "hapiBroadDebugPktFilterSet"
        80ef14a4 80  04  6d  1c    addr       FUN_80046d1c
        80ef14a8 80  9e  ee  68    addr       s_hapiBroadPolicyDebug_809eee68                  = "hapiBroadPolicyDebug"
        80ef14ac 80  07  49  c8    addr       FUN_800749c8
        80ef14b0 80  9e  ee  80    addr       DAT_809eee80                                     = 69h    i
        80ef14b4 80  2f  6d  74    addr       LAB_802f6d74
        80ef14b8 80  9e  ee  84    addr       s_ifconfig_809eee84                              = "ifconfig"
        80ef14bc 80  2f  6d  74    addr       LAB_802f6d74
        80ef14c0 80  9e  ee  90    addr       DAT_809eee90                                     = 6Ah    j
        80ef14c4 80  3b  fa  58    addr       LAB_803bfa58
        80ef14c8 80  9e  ee  98    addr       s_logClear_809eee98                              = "logClear"
        80ef14cc 80  2d  4c  d0    addr       FUN_802d4cd0
        80ef14d0 80  9e  ee  a4    addr       s_logConsole_809eeea4                            = "logConsole"
        80ef14d4 80  2d  04  b0    addr       FUN_802d04b0
        80ef14d8 80  9e  ee  b0    addr       s_logError_809eeeb0                              = "logError"
        80ef14dc 80  3c  98  48    addr       FUN_803c9848
        80ef14e0 80  9e  ee  bc    addr       s_logShow_809eeebc                               = "logShow"
        80ef14e4 80  2d  4f  a8    addr       LAB_802d4fa8
        80ef14e8 80  9e  ee  c4    addr       s_mbufFree_809eeec4                              = "mbufFree"
        80ef14ec 80  30  82  2c    addr       FUN_8030822c
        80ef14f0 80  9e  ee  d0    addr       s_mbufHistoryClear_809eeed0                      = "mbufHistoryClear"
        80ef14f4 80  30  74  44    addr       LAB_80307444
        80ef14f8 80  9e  ee  e4    addr       s_mbufHistoryDelete_809eeee4                     = "mbufHistoryDelete"
        80ef14fc 80  30  73  bc    addr       LAB_803073bc
        80ef1500 80  9e  ee  f8    addr       s_mbufHistoryDump_809eeef8                       = "mbufHistoryDump"
        80ef1504 80  30  74  bc    addr       LAB_803074bc
        80ef1508 80  9e  ef  08    addr       s_mbufHistoryInit_809eef08                       = "mbufHistoryInit"
        80ef150c 80  30  72  a0    addr       LAB_803072a0
        80ef1510 80  9e  ef  18    addr       s_mbufShow_809eef18                              = "mbufShow"
        80ef1514 80  30  76  08    addr       FUN_80307608
        80ef1518 80  9e  ef  24    addr       s_memCheck_809eef24                              = "memCheck"
        80ef151c 80  3b  d4  74    addr       LAB_803bd474
        80ef1520 80  9e  ef  30    addr       s_memShow_809eef30                               = "memShow"
        80ef1524 80  3b  ee  24    addr       LAB_803bee24
        80ef1528 80  9e  ef  38    addr       s_mmuConfig_809eef38                             = "mmuConfig"
        80ef152c 80  04  ec  a0    addr       FUN_8004eca0
        80ef1530 80  9e  ef  44    addr       s_mmuCount_809eef44                              = "mmuCount"
        80ef1534 80  05  21  a8    addr       LAB_800521a8
        80ef1538 80  9e  ef  50    addr       s_mmuState_809eef50                              = "mmuState"
        80ef153c 80  05  13  9c    addr       LAB_8005139c
        80ef1540 80  9e  ef  5c    addr       s_modMemory_809eef5c                             = "modMemory"
        80ef1544 80  3c  94  44    addr       FUN_803c9444
        80ef1548 80  9e  ef  68    addr       DAT_809eef68                                     = 6Dh    m
        80ef154c 80  3c  37  84    addr       FUN_803c3784
        80ef1550 80  9e  ef  70    addr       s_msgQprint_809eef70                             = "msgQprint"
        80ef1554 80  3c  3b  74    addr       LAB_803c3b74
        80ef1558 80  9e  ef  7c    addr       s_msgQshow_809eef7c                              = "msgQshow"
        80ef155c 80  3c  38  2c    addr       FUN_803c382c
        80ef1560 80  9e  ef  88    addr       s_nimDebugDump_809eef88                          = "nimDebugDump"
        80ef1564 80  2d  94  4c    addr       FUN_802d944c
        80ef1568 80  9e  ef  98    addr       s_nimPortDump_809eef98                           = "nimPortDump"
        80ef156c 80  2d  a4  14    addr       LAB_802da414
        80ef1570 80  9e  ef  a4    addr       s_nsdpDebugSet_809eefa4                          = "nsdpDebugSet"
        80ef1574 80  2e  88  d8    addr       LAB_802e88d8
        80ef1578 80  9e  ef  b4    addr       s_osapiDebugMallocDetail_809eefb4                = "osapiDebugMallocDetail"
        80ef157c 80  3b  dc  60    addr       LAB_803bdc60
        80ef1580 80  9e  ef  cc    addr       s_osapiDebugMallocDetailEnable_809eefcc          = "osapiDebugMallocDetailEnable"
        80ef1584 80  3b  db  7c    addr       LAB_803bdb7c
        80ef1588 80  9e  ef  ec    addr       s_osapiDebugMallocSummary_809eefec               = "osapiDebugMallocSummary"
        80ef158c 80  3b  d1  b0    addr       FUN_803bd1b0
        80ef1590 80  9e  f0  04    addr       s_osapiDebugMemoryInfo_809ef004                  = "osapiDebugMemoryInfo"
        80ef1594 80  3b  da  54    addr       LAB_803bda54
        80ef1598 80  9e  f0  1c    addr       s_osapiDebugMemoryStats_809ef01c                 = "osapiDebugMemoryStats"
        80ef159c 80  3b  d8  24    addr       FUN_803bd824
        80ef15a0 80  9e  f0  34    addr       s_osapiDebugMsgQueuePrint_809ef034               = "osapiDebugMsgQueuePrint"
        80ef15a4 80  3c  37  84    addr       FUN_803c3784
        80ef15a8 80  9e  f0  4c    addr       s_osapiMemShow_809ef04c                          = "osapiMemShow"
        80ef15ac 80  3b  d1  b0    addr       FUN_803bd1b0
        80ef15b0 80  9e  f0  5c    addr       s_osapiMsgQueueShow_809ef05c                     = "osapiMsgQueueShow"
        80ef15b4 80  3c  38  2c    addr       FUN_803c382c
        80ef15b8 80  9e  f0  70    addr       s_osapiRedir_809ef070                            = "osapiRedir"
        80ef15bc 80  3b  cf  8c    addr       LAB_803bcf8c
        80ef15c0 80  9e  f0  7c    addr       s_osapiTaskShow_809ef07c                         = "osapiTaskShow"
        80ef15c4 80  3c  91  20    addr       FUN_803c9120
        80ef15c8 80  9e  f0  8c    addr       s_phyDump_809ef08c                               = "phyDump"
        80ef15cc 80  04  d5  78    addr       FUN_8004d578
        80ef15d0 80  9e  f0  94    addr       s_phyget_809ef094                                = "phyget"
        80ef15d4 80  0b  e6  a4    addr       FUN_800be6a4
        80ef15d8 80  9e  f0  9c    addr       s_physet_809ef09c                                = "physet"
        80ef15dc 80  0b  ea  a4    addr       FUN_800beaa4
        80ef15e0 80  9e  f0  a4    addr       s_poeCfgDump_809ef0a4                            = "poeCfgDump"
        80ef15e4 80  2e  bb  34    addr       LAB_802ebb34
        80ef15e8 80  9e  f0  b0    addr       s_policy_809ef0b0                                = "policy"
        80ef15ec 80  07  49  c8    addr       FUN_800749c8
        80ef15f0 80  9e  f0  b8    addr       s_policyTable_809ef0b8                           = "policyTable"
        80ef15f4 80  07  44  a8    addr       LAB_800744a8
        80ef15f8 80  9e  f0  c4    addr       s_poolShow_809ef0c4                              = "poolShow"
        80ef15fc 80  29  bb  e0    addr       FUN_8029bbe0
        80ef1600 80  9e  f0  d0    addr       s_reboot_809ef0d0                                = "reboot"
        80ef1604 80  04  63  10    addr       FUN_80046310
        80ef1608 80  9e  f0  d8    addr       s_regDump_809ef0d8                               = "regDump"
        80ef160c 80  04  e4  30    addr       LAB_8004e430
        80ef1610 80  9e  f0  e0    addr       s_routePrint_809ef0e0                            = "routePrint"
        80ef1614 80  3c  5b  48    addr       LAB_803c5b48
        80ef1618 80  9e  f0  ec    addr       s_rxShow_809ef0ec                                = "rxShow"
        80ef161c 80  04  9c  60    addr       FUN_80049c60
        80ef1620 80  9e  f0  f4    addr       s_setdhcp_809ef0f4                               = "setdhcp"
        80ef1624 80  2f  37  bc    addr       FUN_802f37bc
        80ef1628 80  9e  f0  fc    addr       s_shadowDump_809ef0fc                            = "shadowDump"
        80ef162c 80  28  74  b4    addr       LAB_802874b4
        80ef1630 80  9e  f1  08    addr       s_showCS_809ef108                                = "showCS"
        80ef1634 80  3c  97  ec    addr       FUN_803c97ec
        80ef1638 80  9e  f1  10    addr       s_sntpDebugCurrentTime_809ef110                  = "sntpDebugCurrentTime"
        80ef163c 80  2f  f1  d4    addr       FUN_802ff1d4
        80ef1640 80  9e  f1  28    addr       s_sntpDebugMode_809ef128                         = "sntpDebugMode"
        80ef1644 80  2f  f1  c8    addr       LAB_802ff1c8
        80ef1648 80  9e  f1  38    addr       s_sntpStatusShow_809ef138                        = "sntpStatusShow"
        80ef164c 80  2f  eb  c8    addr       FUN_802febc8
        80ef1650 80  9e  f1  48    addr       s_soc_property_get_809ef148                      = "soc_property_get"
        80ef1654 80  24  00  04    addr       FUN_80240004
        80ef1658 80  9e  f1  5c    addr       s_ssltDebugLevelSet_809ef15c                     = "ssltDebugLevelSet"
        80ef165c 80  43  e3  e4    addr       LAB_8043e3e4
        80ef1660 80  9e  f1  70    addr       s_sysShow_809ef170                               = "sysShow"
        80ef1664 80  30  1d  d4    addr       FUN_80301dd4
        80ef1668 80  9e  f1  78    addr       s_taskShow_809ef178                              = "taskShow"
        80ef166c 80  3b  ef  3c    addr       FUN_803bef3c
        80ef1670 80  9e  f1  84    addr       s_upTime_809ef184                                = "upTime"
        80ef1674 80  3b  e6  1c    addr       FUN_803be61c

```
Note that this list contains commands that are not present in the list from the last blog post. That may be because the commands did not exist before this version, or perhaps they were omitted from the `help` command list in the last version that supported the `help` command (`5.4.2.29`).
Let's execute a few of these commands and see what we can see.

## debugEwaSessionTableDump

![debugEwaSessionTableDump screenshot](/images/0056/debugEwaSessionTableDump.PNG "debugEwaSessionTableDump Screenshot")

This appears to be the session information for authenticated clients.  It includes the username (in this case `admin`) and the privilege level.  Most importantly, it includes the session identifier that can be used as the value for the `SID` cookie.  With this value, we can masquerade as another user, should they currently be authenticated to the switch.

By default, the switch uses a local configuration database, but it does have options to connect to TACAs+ and RADIUS for additional user authentication.  If one of those latter two options is configured, more than one admin can authenticate to the switch, in which case admins could steal other admins sessions.  The bad news here is it makes it possible for admins to perform actions as other admins.  Obviously undesireable behavior, but it is not clear whether or not there is an escalation of privilege. 