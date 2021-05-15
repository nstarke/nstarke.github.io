# Inpoutx64.sys Windows Driver Analysis

Published: May 15, 2021

## Prior Art

[https://github.com/namazso/physmem_drivers/blob/master/README.MD](https://github.com/namazso/physmem_drivers/blob/master/README.MD) - a list of drivers allowing access to physical memory, this list contains the driver analyzed in this blog post.

[https://gist.github.com/k4nfr3/af970e7facb09195e56f2112e1c9549c](https://gist.github.com/k4nfr3/af970e7facb09195e56f2112e1c9549c) - A list of "IOC Vulnerable Drivers"; this list contains the driver analyzed in this blog post.

## Meta Data

File: `C:\Windows\System32\Drivers\inpoutx64.sys`

SHA256: `f8965fdce668692c3785afa3559159f9a18287bc0d53abb21902895a8ecf221b`

SHA1: `6afc6b04cf73dd461e4a4956365f25c1f1162387`

MD5: `9321a61a25c7961d9f36852ecaa86f55`

## Investigation

I recently analyzed a MSI PRO 10M-060US All-in-one PC.  This is a standard AIO-type PC.  My goal was to identify vulnerabilities in the factory image of the host as well as it's UEFI-based firmware.  One of the first things I did was to run [DriverQuery](https://github.com/matterpreter/OffensiveCSharp/tree/master/DriverQuery) against the system.  All of the drivers currently loaded on the system were signed by Intel or Realtek except for one, `inpoutx64.sys`.

DriverQuery output:

```
[...]
Kernel level port access driver
    Service Name: inpoutx64
    Path: C:\Windows\System32\Drivers\inpoutx64.sys
    Version: 1.2 x64 built by: WinDDK
    Creation Time (UTC): 3/9/2020 7:03:58 PM
    Cert Issuer: CN=VeriSign Class 3 Code Signing 2004 CA, OU=Terms of use at https://www.verisign.com/rpa (c)04, OU=VeriSign Trust Network, O="VeriSign, Inc.", C=US
    Signer: CN=Red Fox UK Limited, OU=Development, OU=Digital ID Class 3 - Microsoft Software Validation v2, O=Red Fox UK Limited, L=London, S=London, C=GB
[...]
```

There wasn't much information available on the `Signer` (`Red Fox UK Limited`) and that and the fact that it was the only driver signed by that signer made it stand out to me.  I loaded the Driver in Ghidra and decided to take a look.

The entry function:
```c++
void entry(PDRIVER_OBJECT param_1)

{
  int iVar1;
  undefined local_98 [16];
  undefined local_88 [16];
  undefined local_78 [8];
  undefined8 local_70;
  undefined8 local_68;
  undefined8 local_60;
  undefined8 local_58;
  undefined4 local_50;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined8 local_28;
  undefined4 local_20;
  ulonglong local_18;
  
  if (((DAT_00013108 == 0) || (DAT_00013108 == 0x2b992ddfa232)) &&
     (DAT_00013108 = (_DAT_fffff78000000320 ^ 0x13108) & 0xffffffffffff, DAT_00013108 == 0)) {
    DAT_00013108 = 0x2b992ddfa232;
  }
  DAT_00013100 = ~DAT_00013108;
  local_18 = DAT_00013108;
  local_70 = 0x7600650044005c;
  local_68 = 0x5c006500630069;
  local_60 = 0x6f0070006e0069;
  local_58 = 0x36007800740075;
  local_50 = 0x34;
  local_48 = 0x73006f0044005c;
  local_40 = 0x69007600650044;
  local_38 = 0x5c007300650063;
  local_30 = 0x6f0070006e0069;
  local_28 = 0x36007800740075;
  local_20 = 0x34;
  RtlInitUnicodeString(local_98,&local_70);
  RtlInitUnicodeString(local_88,&local_48);
  iVar1 = IoCreateDevice(param_1,0,local_98,0x22,0,0,local_78);
  if ((-1 < iVar1) && (iVar1 = IoCreateSymbolicLink(local_88,local_98), -1 < iVar1)) {
    param_1->MajorFunction[0] = &LAB_00011010;
    param_1->MajorFunction[0xe] = FUN_000112f0;
    param_1->DriverUnload = &LAB_00011040;
  }
  FUN_00011a00(local_18);
  return;
}
```

There are two 16-bit unicode strings denoting the driver device IO path (that is what `local_70`-`local_20` are in the above decompiler output).  These two strings are:

* `\Driver\inpoutx64`
* `\DosDevices\inpoutx64`

These are then passed to `IoCreateDevice` to create the Device Object in the `entry` function. `param_1->MajorFunction[0xe] = FUN_000112f0;` defines the `IRP_MJ_DEVICE_CONTROL` handler function, which is the function responsible for handling IOCTL calls.


```c++

int FUN_000112f0(undefined8 param_1,PIRP param_2)

{
  undefined2 uVar1;
  uint uVar2;
  uint uVar3;
  longlong lVar4;
  undefined8 *puVar5;
  int iVar6;
  undefined uVar7;
  undefined2 uVar8;
  undefined4 uVar9;
  int iVar10;
  ulonglong uVar11;
  ulonglong uVar12;
  undefined8 local_38;
  longlong local_30;
  longlong local_28;
  longlong local_20 [2];
  
  lVar4 = *(longlong *)&(param_2->Tail).field_0x40;
  uVar2 = *(uint *)(lVar4 + 8);
  uVar3 = *(uint *)(lVar4 + 0x10);
  uVar12 = (ulonglong)uVar3;
  puVar5 = *(undefined8 **)((longlong)&param_2->AssociatedIrp + 4);
  iVar10 = 0;
  iVar6 = 0;
  switch(*(undefined4 *)(lVar4 + 0x18)) {
  case 0x9c402004:
    if ((1 < uVar3) && (uVar2 != 0)) {
      uVar7 = in(*(undefined2 *)puVar5);
      *(undefined *)puVar5 = uVar7;
    }
    (param_2->IoStatus).Information = 1;
    goto LAB_000114ac;
  default:
    iVar6 = -0x3fffffff;
    break;
  case 0x9c402008:
    if (2 < uVar3) {
      uVar8 = *(undefined2 *)puVar5;
      uVar7 = *(undefined *)((longlong)puVar5 + 2);
      (param_2->IoStatus).Information = 10;
      out(uVar8,uVar7);
      goto LAB_000114ac;
    }
    break;
  case 0x9c40200c:
    if ((1 < uVar3) && (1 < uVar2)) {
      uVar8 = in(*(undefined2 *)puVar5);
      *(undefined2 *)puVar5 = uVar8;
    }
    (param_2->IoStatus).Information = 2;
    goto LAB_000114ac;
  case 0x9c402010:
    if (3 < uVar3) {
      uVar8 = *(undefined2 *)puVar5;
      uVar1 = *(undefined2 *)((longlong)puVar5 + 2);
      (param_2->IoStatus).Information = 10;
      out(uVar8,uVar1);
      goto LAB_000114ac;
    }
    break;
  case 0x9c402014:
    if ((3 < uVar3) && (3 < uVar2)) {
      uVar9 = in(*(undefined2 *)puVar5);
      *(undefined4 *)puVar5 = uVar9;
    }
    (param_2->IoStatus).Information = 4;
    goto LAB_000114ac;
  case 0x9c402018:
    iVar6 = 0;
    if (7 < uVar3) {
      uVar9 = *(undefined4 *)((longlong)puVar5 + 4);
      (param_2->IoStatus).Information = 10;
      out((short)puVar5,uVar9);
      goto LAB_000114ac;
    }
    break;
  case 0x9c40201c:
    if (uVar3 != 0) {
      FUN_000116b0(&local_38,puVar5,uVar12);
      uVar11 = FUN_000110e0(local_28,local_30,local_20,&local_38);
      iVar10 = (int)uVar11;
      if (-1 < iVar10) {
        FUN_000116b0(puVar5,&local_38,uVar12);
        (param_2->IoStatus).Information = uVar12;
      }
      *(int *)&(param_2->IoStatus).unlabelled0 = iVar10;
      goto LAB_000114ac;
    }
    goto LAB_0001149a;
  case 0x9c402020:
    if (uVar3 != 0) {
      FUN_000116b0(&local_38,puVar5,uVar12);
      iVar10 = ZwUnmapViewOfSection(0xffffffffffffffff,local_20[0]);
      ZwClose(local_38);
      *(int *)&(param_2->IoStatus).unlabelled0 = iVar10;
      goto LAB_000114ac;
    }
LAB_0001149a:
    *(undefined4 *)&(param_2->IoStatus).unlabelled0 = 0xc000000d;
    goto LAB_000114ac;
  }
  iVar10 = iVar6;
  (param_2->IoStatus).Information = 0;
LAB_000114ac:
  *(int *)&(param_2->IoStatus).unlabelled0 = iVar10;
  IofCompleteRequest(param_2,0);
  return iVar10;
}
```

The `switch` statement operates on the value `*(undefined4 *)(stack + 0x18)`.  This value is the IOCTL code value specified in User space. The function handles eight different IOCTL codes:

* `0x9c402004`
* `0x9c402008`
* `0x9c40200c`
* `0x9c402010`
* `0x9c402014`
* `0x9c402018`
* `0x9c40201c`
* `0x9c402020`

The following IOCTL's are particularly interesting:

* `0x9c402004` - `in`
* `0x9c402008` - `out`
* `0x9c40200c` - `in`
* `0x9c402010` - `out`
* `0x9c402014` - `in`

Each one of the IOCTL handlers allows input from user space to be passed to the `in` or `out` function, which translates to the `in` and `out` instructions in Intel assembly.  These functions/instructions allow writing to the processor's IO ports and are only accessible from kernel mode (Ring 0).  According to [Sentinel Labs](https://labs.sentinelone.com/cve-2021-21551-hundreds-of-millions-of-dell-computers-at-risk-due-to-multiple-bios-driver-privilege-escalation-flaws/) this capability can be abuse to write arbitrary data to the hard disk at the lowest level. 

They also allowing calling a System Mode Interrupt (SMI) by passing `0xb2` as the first operand to the `out` function.

So it would be really bad if a low privileged user could call these IOCTLs from user space.  This happens to be the case with this particular driver:

```
4: kd> !drvobj inpoutx64
Driver object (ffffab0806cf6e20) is for:
 \Driver\inpoutx64

Driver Extension List: (id , addr)

Device Object list:
ffffab0806cc6c80  
4: kd> !devobj ffffab0806cc6c80  
Device object (ffffab0806cc6c80) is for:
 inpoutx64 \Driver\inpoutx64 DriverObject ffffab0806cf6e20
Current Irp 00000000 RefCount 1 Type 00000022 Flags 00000040
SecurityDescriptor ffffbd07846fe0a0 DevExt 00000000 DevObjExt ffffab0806cc6dd0 
ExtensionFlags (0x00000800)  DOE_DEFAULT_SD_PRESENT
Characteristics (0000000000)  
Device queue is not busy.
4: kd> !sd ffffbd07846fe0a0 0x1
->Revision: 0x1
->Sbz1    : 0x0
->Control : 0x8814
            SE_DACL_PRESENT
            SE_SACL_PRESENT
            SE_SACL_AUTO_INHERITED
            SE_SELF_RELATIVE
->Owner   : S-1-5-32-544 (Alias: BUILTIN\Administrators)
->Group   : S-1-5-18 (Well Known Group: NT AUTHORITY\SYSTEM)
->Dacl    : 
->Dacl    : ->AclRevision: 0x2
->Dacl    : ->Sbz1       : 0x0
->Dacl    : ->AclSize    : 0x5c
->Dacl    : ->AceCount   : 0x4
->Dacl    : ->Sbz2       : 0x0
->Dacl    : ->Ace[0]: ->AceType: ACCESS_ALLOWED_ACE_TYPE
->Dacl    : ->Ace[0]: ->AceFlags: 0x0
->Dacl    : ->Ace[0]: ->AceSize: 0x14
->Dacl    : ->Ace[0]: ->Mask : 0x001201bf
->Dacl    : ->Ace[0]: ->SID: S-1-1-0 (Well Known Group: localhost\Everyone)

->Dacl    : ->Ace[1]: ->AceType: ACCESS_ALLOWED_ACE_TYPE
->Dacl    : ->Ace[1]: ->AceFlags: 0x0
->Dacl    : ->Ace[1]: ->AceSize: 0x14
->Dacl    : ->Ace[1]: ->Mask : 0x001f01ff
->Dacl    : ->Ace[1]: ->SID: S-1-5-18 (Well Known Group: NT AUTHORITY\SYSTEM)

->Dacl    : ->Ace[2]: ->AceType: ACCESS_ALLOWED_ACE_TYPE
->Dacl    : ->Ace[2]: ->AceFlags: 0x0
->Dacl    : ->Ace[2]: ->AceSize: 0x18
->Dacl    : ->Ace[2]: ->Mask : 0x001f01ff
->Dacl    : ->Ace[2]: ->SID: S-1-5-32-544 (Alias: BUILTIN\Administrators)

->Dacl    : ->Ace[3]: ->AceType: ACCESS_ALLOWED_ACE_TYPE
->Dacl    : ->Ace[3]: ->AceFlags: 0x0
->Dacl    : ->Ace[3]: ->AceSize: 0x14
->Dacl    : ->Ace[3]: ->Mask : 0x001200a9
->Dacl    : ->Ace[3]: ->SID: S-1-5-12 (Well Known Group: NT AUTHORITY\RESTRICTED)

->Sacl    : 
->Sacl    : ->AclRevision: 0x2
->Sacl    : ->Sbz1       : 0x0
->Sacl    : ->AclSize    : 0x1c
->Sacl    : ->AceCount   : 0x1
->Sacl    : ->Sbz2       : 0x0
->Sacl    : ->Ace[0]: ->AceType: SYSTEM_MANDATORY_LABEL_ACE_TYPE
->Sacl    : ->Ace[0]: ->AceFlags: 0x0
->Sacl    : ->Ace[0]: ->AceSize: 0x14
->Sacl    : ->Ace[0]: ->Mask : 0x00000001
->Sacl    : ->Ace[0]: ->SID: S-1-16-4096 (Label: Mandatory Label\Low Mandatory Level)
```

The important line from the above output is `->Dacl    : ->Ace[0]: ->SID: S-1-1-0 (Well Known Group: localhost\Everyone)` which means an unprivileged user can call these IOCTLs.  

A scan of one enterprise resulted in 42 hosts of 24,967 hosts having this driver in their environment.

## Analysis

It seems as though this driver could be used to escalate privileges on Window's hosts.  Actually exploiting the issue is complicated by the low level nature of writing to a hard disk at the IO port level, but it is my understanding this is possible. [As Sentinel Labs put it in the aforementioned Blog Post](https://labs.sentinelone.com/cve-2021-21551-hundreds-of-millions-of-dell-computers-at-risk-due-to-multiple-bios-driver-privilege-escalation-flaws/), regarding a similar vulnerability in a different driver:

>Another interesting vulnerability in this driver is one that makes it possible to run I/O (IN/OUT) instructions in kernel mode with arbitrary operands (LPE #3 and LPE #4). This is less trivial to exploit and might require using various creative techniques to achieve elevation of privileges.

>Since IOPL (I/O privilege level) equals to CPL (current privilege level), it is obviously possible to interact with peripheral devices such as the HDD and GPU to either read/write directly to the disk or invoke DMA operations. For example, we could communicate with ATA port IO for directly writing to the disk, then overwrite a binary that is loaded by a privileged process.

The driver does not seem to be exclusive to MSI and appears to be contained in other OEM factory images as well.  As such, I am applying full disclosure to this vulnerability since I do not have the resources to contact each OEM individually, and there appears to be prior art on this driver. Along these lines, without any contact information for `Red Fox UK Limted`, I did not have the option to contact the driver author either.

It is not clear to me what legitimate use such a driver would have or why its capabilities would need to be exposed to `localhost\Everyone`.

Sources:

[https://labs.sentinelone.com/cve-2021-21551-hundreds-of-millions-of-dell-computers-at-risk-due-to-multiple-bios-driver-privilege-escalation-flaws/](https://labs.sentinelone.com/cve-2021-21551-hundreds-of-millions-of-dell-computers-at-risk-due-to-multiple-bios-driver-privilege-escalation-flaws/)

[https://posts.specterops.io/methodology-for-static-reverse-engineering-of-windows-kernel-drivers-3115b2efed83](https://posts.specterops.io/methodology-for-static-reverse-engineering-of-windows-kernel-drivers-3115b2efed83)

[https://gist.github.com/k4nfr3/af970e7facb09195e56f2112e1c9549c](https://gist.github.com/k4nfr3/af970e7facb09195e56f2112e1c9549c)

[https://github.com/namazso/physmem_drivers/blob/master/README.MD](https://github.com/namazso/physmem_drivers/blob/master/README.MD)

## NB

I am fairly new to the Windows Operating System and its internal components, including Windows Drivers.  If you see anything that is factually incorrect in this article I would love to hear from you: [@nstarke](https://twitter.com/nstarke).

[Back](/)