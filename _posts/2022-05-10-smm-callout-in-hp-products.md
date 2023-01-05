---
layout: posts
title:  "SMM Callouts in HP Products"
date:   2022-05-10 00:00:00 -0600
categories: uefi smm
author: Nicholas Starke
---

**My HP PSRT case was PSR-2021-0177 which I have been working to make public since early November 2021.  The advisory was released May 10th, 2022 and did not, at least in the initial draft, credit me anywhere.**

In the HP ProBook G4 650 model of laptops running firmware version 1.17.0, there exists an SMI handler that calls out from SMM.

[HP Security Advisory](https://support.hp.com/us-en/document/ish_6184733-6184761-16/hpsbhf03788)

## Impact

This vulnerability could allow an attacker executing with kernel-level privileges (CPL == 0) to escalate privileges to System Management Mode (SMM). Executing in SMM gives an attacker full privileges over the host to further carry out attacks.

## Description

There is a software System Management Interrupt Handler (SMI Handler) registered with the SMI code 99 (0x63). This handler can be triggered from a kernel execution context such as a Windows Kernel Driver by executing the `out` instruction with the arguments `0xb2` and `0x63` (`__outbyte(0xb2, 0x63)`)). This will cause the SMI handler to execute.  The SMI Handler, as decompiled by Ghidra, looks like this:

```cpp
undefined8 swSmiHandler38(void)
{
  EFI_HANDLE local_res18 [2];
  
  (*gBS->LocateProtocol)((EFI_GUID *)&DAT_800071e0,(void *)0x0,(void **)&DAT_80008328);
  (*gBS->LocateProtocol)((EFI_GUID *)&DAT_80007240,(void *)0x0,(void **)&DAT_80008340);
  if (DAT_80008340 != 0) {
    DAT_80008320 = DAT_80008340 + 0x100;
    *(undefined *)(DAT_80008340 + 0x19d) = 1;
  }
  local_res18[0] = (EFI_HANDLE)0x0;
  (*gSmst12->SmmInstallProtocolInterface)
            (local_res18,&unknownProtocol_33fef311,EFI_NATIVE_INTERFACE,(void *)0x0);
  return 0;
}
```

An attacker can exploit this vulnerability by searching physical memory for the `EFI_BOOT_SERVICES` table.  From this table the attacker can determine the memory address of the `LocateProtocol` function and overwrite it in physical memory to point to attacker controlled code.  As discussed in the Mitigations section, this code would need to point inside the range specified by SMRR, which for this model of laptop's firmware version begins at `0xd0000000`. Counter-measures to bypass this requirement are discussed in the Mitigations section.  Code to find the `EFI_BOOT_SERVICES` table in physical memory from the `EFI_RUNTIME_TABLE` exists and is [freely distributed on Github](https://github.com/Cr4sh/fwexpl/blob/f3c6934d0c54ec7fcc800c04ea26c57934aba5fb/src/application/src/windows_specific.cpp#L27). 

Once an attacker has overwritten the function pointer for `LocateProtocol`, the attacker need only to cause the SMI Handler to execute by executing the `__outbyte(0xb2, 0x63)` instruction.

## Mitigations

The laptop in question has the `SMM_Code_Chk_En` bit of `MSR_SMM_FEATURE_CONTROL` set to 1, which means any jump to code outside of the ranges set by SMRR assert unrecoverably and cause the system to crash. An attacker would need to bypass this capability.  This can be accomplished by jumping to shellcode encoded in the r8-r15 registers that are saved as part of the SMM context switch and then jumping to this shellcode. For more information on this bypass, please see [this excellent article](https://www.synacktiv.com/en/publications/code-checkmate-in-smm.html).

Additionally, HP Sure Start detects that the firmware runtime has been tampered with in many scenarios.  HP Sure Start would also need to be bypassed to gain full code execution. I am not aware of any published HP Sure Start bypasses.

When the SMI handler is triggered using Chipsec, HP Sure Start detects memory corruption within the firmware and immediately shuts down the host.  When the host is powered back up, the HP Sure Start warning screen is displayed to the user before the system boots, letting them know a serious problem has occurred.  To reproduce this symptom, run the following chipsec command (tested against chipsec 1.7.3):

```
# python chipsec_util.py smi send 0 0x63 0
```

## Remediation

Version 01.19.00 of the firmware fixes this issue. The same SMI Handler when decompiled under version 01.19.00 looks like this:

```cpp
EFI_STATUS
ChildSmiHandler38(EFI_HANDLE DispatchHandle,void *Context,void *CommBuffer,UINTN *CommBufferSize)
{
  EFI_HANDLE *Handle40;
  if (((CommBuffer != (void *)0x0) && (CommBufferSize != (UINTN *)0x0)) &&
     (*CommBuffer == 0x4f494748)) {
    FUN_80001744((longlong)CommBuffer);
  }
  Handle40 = (EFI_HANDLE *)0x0;
  (*gSmst12->SmmInstallProtocolInterface)
            (&Handle40,&unknownProtocol_33fef311,EFI_NATIVE_INTERFACE,(void *)0x0);
  return 0;
}
```

I also traced the call graph to ensure the BootServices callouts had not simply been moved to a sub function.

I'd like to thank the following researchers for answering questions and providing tooling that enabled me to find and analyze this vulnerability.

* [Assaf Carlsbad](https://twitter.com/assaf_carlsbad) (for [brick](https://github.com/Sentinel-One/brick) and answering technical questions)
* [Alex Matrosov](https://twitter.com/matrosov) (for [efiXplorer](https://github.com/binarly-io/efiXplorer) and answering technical and disclosure questions)
* [Mickey](https://twitter.com/hackingthings) (for answering questions)
* [n3k](https://twitter.com/kiqueNissim) (for warning me about [MSR_SMM_FEATURE_CONTROL](https://twitter.com/kiqueNissim/status/1457775382086647815))

I'd also like to thank the HP PSRT for taking this issue seriously, coordinating with the corresponding product teams, creating a fix for it, and releasing a new fixed version.  It took about 6 months from disclosure to advisory, but given the software release cycles for firmware images, I don't think this is unreasonable.
