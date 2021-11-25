## Overview

Last year around this time I took a look at the Walmart-branded Onn 14" laptop. I decided to revisit reverse engineering the UEFI implementation on this laptop recently and decided to focus on the SMI Handlers contained therein.  I will discuss potential vulnerabilities that I think are present in the firmware. I will not move to the exploitation phase for this laptop because my test version of this laptop is effectively ruined at this point and I have no ability to test anything I write for it. 

Also, I have not seen anything on the internet to suggest that the manufacturer supplies firmware updates for this laptop, so it seems as though any bugs or vulnerabilities in this laptop are forever.  I think many of the problems documented in the [last blog post](https://nstarke.github.io/onn/walmart/bios/pdos/2020/11/26/onn-laptop-bios-exploration.html) already spell out that this laptop should not be used for security intensive data or operations, so I don't really feel bad about publishing this information. Plus today is Thanksgiving in the US and I am thankful for, amongst many other things, vulnerable firmware.

## SmmExploit / Aptiocalypsis

Two resources I referenced heavily in this research are [Satoshi Tanda's](https://twitter.com/standa_t) [SmmExploit](https://github.com/tandasat/smmexploit) and [Cr4sh](https://twitter.com/d_olex) [Aptiocalypsis](https://github.com/Cr4sh/Aptiocalypsis). These two resources detail how to exploit SMI Handlers in Aptio-based UEFI images.  I extrapolated my "Theoretical Exploit" largely from these two resources.  Additionally, SmmExploit details a vulnerability in `SdioSmm` that surprisingly is not present in the version of `SdioSmm` that was included in the Onn Laptop Firmware.

## SdioSmm

The Software SMI Handler for the `SdioSmm` module in the Onn Laptop looks like this:

```cpp
EFI_STATUS
swSmiHandler8(EFI_HANDLE DispatchHandle, void *Context, void *CommBuffer, UINTN *CommBufferSize)
{
  longlong is_outside_smram;
  byte *attacker_controlled;
  
  DAT_800031b0 = 1;
  attacker_controlled =
       (byte *)(ulonglong)*(uint *)(ulonglong)((uint)uRam000000000000040e * 0x10 + 0x104);
  if (gAMI_SMM_BUFFER_VALIDATION_PROTOCOL_4 != (INT64 *)0x0) {
    is_outside_smram = (*(code *)*gAMI_SMM_BUFFER_VALIDATION_PROTOCOL_4)(attacker_controlled,0x67);
    if (-1 < is_outside_smram) {
      if (*attacker_controlled < 7) {
        (*(code *)(&PTR_FUN_80003070)[*attacker_controlled])(attacker_controlled);
      }
      else {
        attacker_controlled[2] = 7;
      }
    }
  }
  return 0;
}
```

This appears to be the fixed or non-vulnerable version of `SdioSmm`.  It corresponds nicely to SmmExploit's [Resolution](https://github.com/tandasat/smmexploit#resolution) code example. So, this Onn Laptop firmware from 2019 had the fix for Cr4sh's Aptiocalypsis. This is a good thing for users as it is one less vulnerability in the firmware.

## SbRunSmm

Unfortunately it appears as though there is a vulnerability in the `SbRunSmm` module.  One of the Software SMI Handlers calls out to RuntimeServices's `GetVariable` function.  It appears as though this callout occurs unconditionally; there is no conditional logic or short-circuit returns that would circumvent this callout. As such, it should be possible for an attacker with kernel-level code execution (CPL == 0) can overwrite the `GetVariable` function pointer in the RuntimeServices address range and thereby gain code execution as SMM. After the `GetVariable` function pointer is overwritten, it is only necessary to trigger the Software SMI Handler in question.

## Theoretical Exploit

The `SbRunSmm` Software SMI Handler that can be triggered by writing `0xbb` as the second argument to `out(0xb2, 0xbb)` (`swSmiHandler16` from `Appendix B`) contains a call to `RuntimeServices->GetVariable`. SmmExploit details a method by which it is possible to retrieve the UEFI RuntimeServices address range from the Windows Registry and then use that to retrieve the System Management System Table (SMST) address from the RuntimeServices struct.  My theoretical exploit would use the UEFI RuntimeServices Address Range to find the `GetVariable` function pointer and overwrite it with a pointer to shellcode. `Get)Variable` is defined in the TianoCore reference implementation [here](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Uefi/UefiSpec.h#L1843).

The first field of the RuntimeServices table is a [header](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Uefi/UefiSpec.h#L1824).  The [first field](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Uefi/UefiMultiPhase.h#L151) in this header is a signature defined as an 8-byte field.  [The signature](https://github.com/tianocore/edk2/blob/master/MdePkg/Include/Uefi/UefiSpec.h#L1814) for RuntimeServices is `RUNTSERV` as byte values.  [SmmExploit searches](https://github.com/tandasat/SmmExploit/blob/main/Demo/Demo/FindSystemManagementServiceTable.cpp#L199) for the signature `cmms`; while this is a 32-bit value, I think the code could be adjusted easily to search for the 64-bit RuntimeServices header signature.  I don't think even the [search range](https://github.com/tandasat/SmmExploit/blob/main/Demo/Demo/FindSystemManagementServiceTable.cpp#L128) would need to be adjusted as the `FindSmmCorePrivateData` function in SmmExploit already searches over the RuntimeServices range [pulled out of the registry](https://github.com/tandasat/SmmExploit/blob/main/Demo/Demo/FindSystemManagementServiceTable.cpp#L138). Once we've found the header of the RuntimeServices table we can calculate the offset to the `GetVariable` function pointer and overwite that memory location with a function pointer to shellcode.  SmmExploit handles this nicely and it should be possible to modify it to suit such needs.

```cpp
    smi_handler_code = 0xbb;
    sw_smi_handler_efi_handle = (EFI_HANDLE)0x0;
    [...]
    (*EFI_SMM_SW_DISPATCH2_PROTOCOL14->Register)
            (EFI_SMM_SW_DISPATCH2_PROTOCOL14,swSmiHandler16,&smi_handler_code,
             &sw_smi_handler_efi_handle);
    [...]
```

The call to `RuntimeServices->GetVariable` looks like this (from Appendix B):

```cpp
EVar3 = (*gRT->GetVariable)((CHAR16 *)u_MeSetup_80004d08,(EFI_GUID *)&DAT_80004000,(UINT32 *)0x0,
                              local_1338,local_1328);
```

## Mitigations

Chipsec reports that SMM_Code_Chk_En is enabled and locked down (from Appendix C), meaning it shouldn't be possible to execute code outside SMRR while in System Management Mode. There are some counter measures that can be employed to bypass this restriction.  See [this synacktiv blog post](https://www.synacktiv.com/publications/code-checkmate-in-smm.html) for more information.


## How did I find this?

Utilizing the SMI Handler enumeration technique I detailed in [this blog post](https://nstarke.github.io/smi/uefi/ghidra/2021/06/25/enumerating-smi-handlers.html), I built a list of all the `swSmiHandlers` in the firmware image and then just started going through them one by one.  You will notice that `SbRunSmm` is the first SMM module after `NbSmi`.  `NbSmi` seems to be the context switcher for transitioning into System Management Mode and storing the existing register values.  In any event, the SMI Handler in `NbSmi` does not immediately seem vulnerable to callouts or confused deputy attacks, so after it I immediately moved on to the three `swSmiHandlers` in `SbRunSmm`.  

This is all a long way of saying the actualy analysis was a manual process, and I didn't even get very far in the `swSmiHandler` list before I found something.  I will continue exploring this firmware image for interesting things and write additional blog posts should I find anything else fun.

## Appendix A: SMI Handlers

```
FindContainsFunction.java> Running...
FindContainsFunction.java> found swSmiHandler17 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 184 - NbSmi/PE32 Image Section
FindContainsFunction.java> found swSmiHandler16 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 186 - SbRunSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler18 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 186 - SbRunSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler17 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 186 - SbRunSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler15 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 190 - AcpiModeEnable/PE32 Image Section
FindContainsFunction.java> found swSmiHandler16 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 190 - AcpiModeEnable/PE32 Image Section
FindContainsFunction.java> found swSmiHandler6 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 191 - PepBccdSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler10 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 193 - TbtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler6 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 197 - OverClockSmiHandler/PE32 Image Section
FindContainsFunction.java> found swSmiHandler20 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 198 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler21 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 198 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler9 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 208 - MicrocodeUpdate/PE32 Image Section
FindContainsFunction.java> found swSmiHandler18 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 210 - LegacySmmSredir/PE32 Image Section
FindContainsFunction.java> found swSmiHandler27 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 213 - UsbRtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler14 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 216 - CmosSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler24 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 219 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found swSmiHandler13 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 220 - NvmeSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler8 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 223 - SdioSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler19 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 225 - SmbiosDmiEdit/PE32 Image Section
FindContainsFunction.java> found swSmiHandler25 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 226 - SmiFlash/PE32 Image Section
FindContainsFunction.java> found swSmiHandler6 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 227 - TcgSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler9 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 229 - SmmTcgStorageSec/PE32 Image Section
FindContainsFunction.java> found swSmiHandler19 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 231 - CrbSmi/PE32 Image Section
FindContainsFunction.java> found swSmiHandler10 in /onn-laptop-bios.bin/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 003 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 232 - PiSmmCommunicationSmm/PE32 Image Section
FindContainsFunction.java> Finished!
```

Appendix B: SbRunSmm swSmiHandler16

```cpp
undefined8 swSmiHandler16(void)

{
  char cVar1;
  undefined4 *puVar2;
  EFI_STATUS EVar3;
  short sVar4;
  longlong lVar5;
  longlong lVar6;
  uint *puVar7;
  int *piVar8;
  ulonglong uVar9;
  undefined4 *puVar10;
  undefined local_1358;
  byte local_1357 [3];
  undefined4 local_1354;
  undefined2 local_1350 [2];
  undefined2 local_134c;
  uint local_1348;
  undefined4 local_1344;
  uint local_1340 [2];
  UINTN local_1338 [2];
  undefined local_1328 [2311];
  char local_a21;
  
  sVar4 = 0;
  local_1340[0] = 0xc4;
  local_1340[1] = 200;
  lVar6 = 0xe;
  lVar5 = 2;
  piVar8 = &DAT_80004818;
  do {
    puVar10 = (undefined4 *)(*(longlong *)(piVar8 + -2) + 0xe00b8000);
    if (*piVar8 == 0) {
      local_1358 = *(undefined *)puVar10;
      puVar2 = (undefined4 *)&local_1358;
    }
    else {
      if (*piVar8 == 1) {
        local_134c = *(undefined2 *)puVar10;
        puVar2 = (undefined4 *)&local_134c;
      }
      else {
        local_1344 = *puVar10;
        puVar2 = &local_1344;
      }
    }
    (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
              (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,2,*piVar8,puVar10,1,puVar2);
    piVar8 = piVar8 + 4;
    lVar6 = lVar6 + -1;
  } while (lVar6 != 0);
  puVar7 = local_1340;
  uVar9 = (ulonglong)uRam00000000e00fd010;
  do {
    puVar10 = (undefined4 *)((ulonglong)*puVar7 + (uVar9 & 0xfffff000));
    local_1354 = *puVar10;
    (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
              (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,2,2,puVar10,1,&local_1354);
    puVar7 = puVar7 + 1;
    lVar5 = lVar5 + -1;
  } while (lVar5 != 0);
  local_1357[0] = in(0x1804);
  local_1357[0] = local_1357[0] | 1;
  (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
            (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,0,0,0x1804,1,local_1357);
  local_1348 = in(0x1830);
  local_1348 = local_1348 & 0xffffffbf;
  (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
            (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,0,2,0x1830,1,&local_1348);
  local_1350[0] = 0x10;
  (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
            (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,0,1,0x1800,1,local_1350);
  local_1338[0] = 0x1301;
  EVar3 = (*gRT->GetVariable)((CHAR16 *)u_MeSetup_80004d08,(EFI_GUID *)&DAT_80004000,(UINT32 *)0x0,
                              local_1338,local_1328);
  if ((local_a21 == '\0') && (-1 < (longlong)EVar3)) {
    cVar1 = FUN_80001ca0();
    if (cVar1 == '\x01') {
      sVar4 = 0x380;
    }
    else {
      cVar1 = FUN_80001ca0();
      if (cVar1 == '\x02') {
        sVar4 = 0x600;
      }
    }
    puVar10 = (undefined4 *)((ulonglong)(ushort)(sVar4 + 0x1c) | 0xfdba0000);
    local_1354 = *puVar10;
    (*(code *)*gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8)
              (gEFI_S3_SMM_SAVE_STATE_PROTOCOL_8,2,2,puVar10,1,&local_1354);
  }
  return 0;
}
```

## Appendix C: Chipsec Results

# Information:1
#### common.cpu.cpu_info
    
    [*] running module: chipsec.modules.common.cpu.cpu_info
    [x][ =======================================================================
    [x][ Module: Current Processor Information:
    [x][ =======================================================================
    [*] Thread 0000
    [*] Processor: Intel(R) Core(TM) i3-8145U CPU @ 2.10GHz
    [*]            Family: 06 Model: 8E Stepping: B
    [*]            Microcode: 000000A4
    [*]
    [#] INFORMATION: Processor information displayed

# Failed:4
#### common.bios_wp
    
    [*] running module: chipsec.modules.common.bios_wp
    [x][ =======================================================================
    [x][ Module: BIOS Region Write Protection
    [x][ =======================================================================
    [*] BC = 0x00000888 << BIOS Control (b:d.f 00:31.5 + 0xDC)
        [00] BIOSWE           = 0 << BIOS Write Enable 
        [01] BLE              = 0 << BIOS Lock Enable 
        [02] SRC              = 2 << SPI Read Configuration 
        [04] TSS              = 0 << Top Swap Status 
        [05] SMM_BWP          = 0 << SMM BIOS Write Protection 
        [06] BBS              = 0 << Boot BIOS Strap 
        [07] BILD             = 1 << BIOS Interface Lock Down 
    [-] BIOS region write protection is disabled!
    
    [*] BIOS Region: Base = 0x00600000, Limit = 0x00FFFFFF
    SPI Protected Ranges
    ------------------------------------------------------------
    PRx (offset) | Value    | Base     | Limit    | WP? | RP?
    ------------------------------------------------------------
    PR0 (84)     | 00000000 | 00000000 | 00000000 | 0   | 0 
    PR1 (88)     | 00000000 | 00000000 | 00000000 | 0   | 0 
    PR2 (8C)     | 00000000 | 00000000 | 00000000 | 0   | 0 
    PR3 (90)     | 00000000 | 00000000 | 00000000 | 0   | 0 
    PR4 (94)     | 00000000 | 00000000 | 00000000 | 0   | 0 
    
    [!] None of the SPI protected ranges write-protect BIOS region
    
    [!] BIOS should enable all available SMM based write protection mechanisms or configure SPI protected ranges to protect the entire BIOS region
    [-] FAILED: BIOS is NOT protected completely
#### common.me_mfg_mode
    
    [*] running module: chipsec.modules.common.me_mfg_mode
    [x][ =======================================================================
    [x][ Module: ME Manufacturing Mode
    [x][ =======================================================================
    [-] FAILED: ME is in Manufacturing Mode
#### common.spi_access
    
    [*] running module: chipsec.modules.common.spi_access
    [x][ =======================================================================
    [x][ Module: SPI Flash Region Access Control
    [x][ =======================================================================
    SPI Flash Region Access Permissions
    ------------------------------------------------------------
    
    BIOS Region Write Access Grant (00):
      FREG0_FLASHD: 0
      FREG1_BIOS  : 0
      FREG2_ME    : 0
      FREG3_GBE   : 0
      FREG4_PD    : 0
      FREG5       : 0
    BIOS Region Read Access Grant (00):
      FREG0_FLASHD: 0
      FREG1_BIOS  : 0
      FREG2_ME    : 0
      FREG3_GBE   : 0
      FREG4_PD    : 0
      FREG5       : 0
    BIOS Region Write Access (FF):
      FREG0_FLASHD: 1
      FREG1_BIOS  : 1
      FREG2_ME    : 1
      FREG3_GBE   : 1
      FREG4_PD    : 1
      FREG5       : 1
    BIOS Region Read Access (FF):
      FREG0_FLASHD: 1
      FREG1_BIOS  : 1
      FREG2_ME    : 1
      FREG3_GBE   : 1
      FREG4_PD    : 1
      FREG5       : 1
    [*] Software has write access to Platform Data region in SPI flash (it's platform specific)
    [!] WARNING: Software has write access to GBe region in SPI flash
    [-] Software has write access to SPI flash descriptor
    [-] Software has write access to Management Engine (ME) region in SPI flash
    [-] FAILED: SPI Flash Region Access Permissions are not programmed securely in flash descriptor
    [!] System may be using alternative protection by including descriptor region in SPI Protected Range Registers
#### common.spi_desc
    
    [*] running module: chipsec.modules.common.spi_desc
    [x][ =======================================================================
    [x][ Module: SPI Flash Region Access Control
    [x][ =======================================================================
    [*] FRAP = 0x0000FFFF << SPI Flash Regions Access Permissions Register (SPIBAR + 0x50)
        [00] BRRA             = FF << BIOS Region Read Access 
        [08] BRWA             = FF << BIOS Region Write Access 
        [16] BMRAG            = 0 << BIOS Master Read Access Grant 
        [24] BMWAG            = 0 << BIOS Master Write Access Grant 
    [*] Software access to SPI flash regions: read = 0xFF, write = 0xFF
    [-] Software has write access to SPI flash descriptor
    
    [-] FAILED: SPI flash permissions allow SW to write flash descriptor
    [!] System may be using alternative protection by including descriptor region in SPI Protected Range Registers

# Warning:3
#### common.cpu.spectre_v2
    
    [*] running module: chipsec.modules.common.cpu.spectre_v2
    [x][ =======================================================================
    [x][ Module: Checks for Branch Target Injection / Spectre v2 (CVE-2017-5715)
    [x][ =======================================================================
    [*] CPUID.7H:EDX[26] = 1 Indirect Branch Restricted Speculation (IBRS) & Predictor Barrier (IBPB)
    [*] CPUID.7H:EDX[27] = 1 Single Thread Indirect Branch Predictors (STIBP)
    [*] CPUID.7H:EDX[29] = 1 IA32_ARCH_CAPABILITIES
    [+] CPU supports IBRS and IBPB
    [+] CPU supports STIBP
    [*] checking enhanced IBRS support in IA32_ARCH_CAPABILITIES...
    [*]   cpu0: IBRS_ALL = 0
    [-] CPU doesn't support enhanced IBRS
    [!] WARNING: CPU supports mitigation (IBRS) but doesn't support enhanced IBRS
    [!] OS may be using software based mitigation (eg. retpoline)
    [!] WARNING: Retpoline check not implemented in current environment
#### common.rtclock
    
    [*] running module: chipsec.modules.common.rtclock
    [x][ =======================================================================
    [x][ Module: Protected RTC memory locations
    [x][ =======================================================================
    [!] WARNING: Unable to test lock bits without attempting to modify CMOS.
    [*] Run chipsec_main manually with the following commandline flags.
    [*] python chipsec_main -m common.rtclock -a modify
#### common.uefi.s3bootscript
    
    [*] running module: chipsec.modules.common.uefi.s3bootscript
    [x][ =======================================================================
    [x][ Module: S3 Resume Boot-Script Protections
    [x][ =======================================================================
    [*] SMRAM: Base = 0x000000008D000000, Limit = 0x000000008D7FFFFF, Size = 0x00800000
    [!] Found 1 S3 boot-script(s) in EFI variables
    
    [!] WARNING: S3 Boot-Script is inside SMRAM. The script is protected but Dispatch opcodes cannot be inspected
    [!] Additional testing of the S3 boot-script can be done using tools.uefi.s3script_modify

# NotApplicable:1
#### common.cpu.ia_untrusted
    
    [*] running module: chipsec.modules.common.cpu.ia_untrusted
    Skipping module chipsec.modules.common.cpu.ia_untrusted since it is not supported in this platform

# Passed:17
#### common.bios_kbrd_buffer
    
    [*] running module: chipsec.modules.common.bios_kbrd_buffer
    [x][ =======================================================================
    [x][ Module: Pre-boot Passwords in the BIOS Keyboard Buffer
    [x][ =======================================================================
    [*] Keyboard buffer head pointer = 0x0 (at 0x41A), tail pointer = 0x0 (at 0x41C)
    [*] Keyboard buffer contents (at 0x41E):
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |                 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |                 
    [*] Checking contents of the keyboard buffer..
    
    [+] PASSED: Keyboard buffer looks empty. Pre-boot passwords don't seem to be exposed
#### common.bios_smi
    
    [*] running module: chipsec.modules.common.bios_smi
    [x][ =======================================================================
    [x][ Module: SMI Events Configuration
    [x][ =======================================================================
    [-] SMM BIOS region write protection has not been enabled (SMM_BWP is not used)
    
    [*] Checking SMI enables..
        Global SMI enable: 1
        TCO SMI enable   : 1
    [+] All required SMI events are enabled
    
    [*] Checking SMI configuration locks..
    [+] TCO SMI configuration is locked (TCO SMI Lock)
    [+] SMI events global configuration is locked (SMI Lock)
    
    [+] PASSED: All required SMI sources seem to be enabled and locked
#### common.bios_ts
    
    [*] running module: chipsec.modules.common.bios_ts
    [x][ =======================================================================
    [x][ Module: BIOS Interface Lock (including Top Swap Mode)
    [x][ =======================================================================
    [*] BiosInterfaceLockDown (BILD) control = 1
    [*] BIOS Top Swap mode is disabled (TSS = 0)
    [*] RTC TopSwap control (TS) = 1
    [+] PASSED: BIOS Interface is locked (including Top Swap Mode)
#### common.debugenabled
    
    [*] running module: chipsec.modules.common.debugenabled
    [x][ =======================================================================
    [x][ Module: Debug features test
    [x][ =======================================================================
    
    [*] Checking IA32_DEBUG_INTERFACE msr status
    [+] CPU debug interface state is correct.
    
    [*] Checking DCI register status
    [+] DCI Debug is disabled
    
    [*] Module Result
    [+] PASSED: All checks have successfully passed
#### common.ia32cfg
    
    [*] running module: chipsec.modules.common.ia32cfg
    [x][ =======================================================================
    [x][ Module: IA32 Feature Control Lock
    [x][ =======================================================================
    [*] Verifying IA32_Feature_Control MSR is locked on all logical CPUs..
    [*] cpu0: IA32_Feature_Control Lock = 1
    [+] PASSED: IA32_FEATURE_CONTROL MSR is locked on all logical CPUs
#### common.memconfig
    
    [*] running module: chipsec.modules.common.memconfig
    [x][ =======================================================================
    [x][ Module: Host Bridge Memory Map Locks
    [x][ =======================================================================
    [*]
    [*] Checking register lock state:
    [+] PCI0.0.0_BDSM        = 0x        8E000001 - LOCKED   - Base of Graphics Stolen Memory
    [+] PCI0.0.0_BGSM        = 0x        8D800001 - LOCKED   - Base of GTT Stolen Memory
    [+] PCI0.0.0_DPR         = 0x        8D000001 - LOCKED   - DMA Protected Range
    [+] PCI0.0.0_GGC         = 0x             1C1 - LOCKED   - Graphics Control
    [+] PCI0.0.0_MESEG_MASK  = 0x      7FFE000C00 - LOCKED   - Manageability Engine Limit Address Register
    [+] PCI0.0.0_PAVPC       = 0x        8FF00047 - LOCKED   - PAVP Configuration
    [+] PCI0.0.0_REMAPBASE   = 0x       100000001 - LOCKED   - Memory Remap Base Address
    [+] PCI0.0.0_REMAPLIMIT  = 0x       16DF00001 - LOCKED   - Memory Remap Limit Address
    [+] PCI0.0.0_TOLUD       = 0x        90000001 - LOCKED   - Top of Low Usable DRAM
    [+] PCI0.0.0_TOM         = 0x       100000001 - LOCKED   - Top of Memory
    [+] PCI0.0.0_TOUUD       = 0x       16E000001 - LOCKED   - Top of Upper Usable DRAM
    [+] PCI0.0.0_TSEGMB      = 0x        8D000001 - LOCKED   - TSEG Memory Base
    [*]
    [+] PASSED: All memory map registers seem to be locked down
#### common.memlock
    
    [*] running module: chipsec.modules.common.memlock
    [x][ =======================================================================
    [x][ Module: Check MSR_LT_LOCK_MEMORY
    [x][ =======================================================================
    [X] Checking MSR_LT_LOCK_MEMORY status
    [*]   cpu0: MSR_LT_LOCK_MEMORY[LT_LOCK] = 1
    [+] PASSED: Check have successfully passed
#### common.remap
    
    [*] running module: chipsec.modules.common.remap
    [x][ =======================================================================
    [x][ Module: Memory Remapping Configuration
    [x][ =======================================================================
    [*] Registers:
    [*]   TOUUD     : 0x000000016E000001
    [*]   REMAPLIMIT: 0x000000016DF00001
    [*]   REMAPBASE : 0x0000000100000001
    [*]   TOLUD     : 0x90000001
    [*]   TSEGMB    : 0x8D000001
    
    [*] Memory Map:
    [*]   Top Of Upper Memory: 0x000000016E000000
    [*]   Remap Limit Address: 0x000000016DFFFFFF
    [*]   Remap Base Address : 0x0000000100000000
    [*]   4GB                : 0x0000000100000000
    [*]   Top Of Low Memory  : 0x0000000090000000
    [*]   TSEG (SMRAM) Base  : 0x000000008D000000
    
    [*] checking memory remap configuration..
    [*]   Memory Remap is enabled
    [+]   Remap window configuration is correct: REMAPBASE <= REMAPLIMIT < TOUUD
    [+]   All addresses are 1MB aligned
    [*] checking if memory remap configuration is locked..
    [+]   TOUUD is locked
    [+]   TOLUD is locked
    [+]   REMAPBASE and REMAPLIMIT are locked
    [+] PASSED: Memory Remap is configured correctly and locked
#### common.secureboot.variables
    
    [*] running module: chipsec.modules.common.secureboot.variables
    [x][ =======================================================================
    [x][ Module: Attributes of Secure Boot EFI Variables
    [x][ =======================================================================
    [*] Checking protections of UEFI variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:SecureBoot
    [*] Checking protections of UEFI variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:SetupMode
    [*] Checking protections of UEFI variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:PK
    [+] Variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:PK is authenticated (TIME_BASED_AUTHENTICATED_WRITE_ACCESS)
    [*] Checking protections of UEFI variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:KEK
    [+] Variable 61dfe48b-ca93-d211-aa0d-00e098032b8c:KEK is authenticated (TIME_BASED_AUTHENTICATED_WRITE_ACCESS)
    [*] Checking protections of UEFI variable cbb219d7-3a3d-9645-a3bc-dad00e67656f:db
    [+] Variable cbb219d7-3a3d-9645-a3bc-dad00e67656f:db is authenticated (TIME_BASED_AUTHENTICATED_WRITE_ACCESS)
    [*] Checking protections of UEFI variable cbb219d7-3a3d-9645-a3bc-dad00e67656f:dbx
    [+] Variable cbb219d7-3a3d-9645-a3bc-dad00e67656f:dbx is authenticated (TIME_BASED_AUTHENTICATED_WRITE_ACCESS)
    
    [*] Secure Boot appears to be disabled
    [+] PASSED: All Secure Boot UEFI variables are protected
#### common.smm
    
    [*] running module: chipsec.modules.common.smm
    [x][ =======================================================================
    [x][ Module: Compatible SMM memory (SMRAM) Protection
    [x][ =======================================================================
    [*] PCI0.0.0_SMRAMC = 0x1A << System Management RAM Control (b:d.f 00:00.0 + 0x88)
        [00] C_BASE_SEG       = 2 << SMRAM Base Segment = 010b 
        [03] G_SMRAME         = 1 << SMRAM Enabled 
        [04] D_LCK            = 1 << SMRAM Locked 
        [05] D_CLS            = 0 << SMRAM Closed 
        [06] D_OPEN           = 0 << SMRAM Open 
    [*] Compatible SMRAM is enabled
    [+] PASSED: Compatible SMRAM is locked down
#### common.smm_code_chk
    
    [*] running module: chipsec.modules.common.smm_code_chk
    [x][ =======================================================================
    [x][ Module: SMM_Code_Chk_En (SMM Call-Out) Protection
    [x][ =======================================================================
    [*] MSR_SMM_FEATURE_CONTROL = 0x00000005 << Enhanced SMM Feature Control (MSR 0x4E0)
        [00] LOCK             = 1 << Lock bit 
        [02] SMM_Code_Chk_En  = 1 << Prevents SMM from executing code outside the ranges defined by the SMRR 
    [+] PASSED: SMM_Code_Chk_En is enabled and locked down
#### common.smm_dma
    
    [*] running module: chipsec.modules.common.smm_dma
    [x][ =======================================================================
    [x][ Module: SMM TSEG Range Configuration Check
    [x][ =======================================================================
    [*] TSEG      : 0x000000008D000000 - 0x000000008D7FFFFF (size = 0x00800000)
    [*] SMRR range: 0x000000008D000000 - 0x000000008D7FFFFF (size = 0x00800000)
    
    [*] checking TSEG range configuration..
    [+] TSEG range covers entire SMRAM
    [+] TSEG range is locked
    [+] PASSED: TSEG is properly configured. SMRAM is protected from DMA attacks
#### common.smrr
    
    [*] running module: chipsec.modules.common.smrr
    [x][ =======================================================================
    [x][ Module: CPU SMM Cache Poisoning / System Management Range Registers
    [x][ =======================================================================
    [+] OK. SMRR range protection is supported
    
    [*] Checking SMRR range base programming..
    [*] IA32_SMRR_PHYSBASE = 0x8D000006 << SMRR Base Address MSR (MSR 0x1F2)
        [00] Type             = 6 << SMRR memory type 
        [12] PhysBase         = 8D000 << SMRR physical base address 
    [*] SMRR range base: 0x000000008D000000
    [*] SMRR range memory type is Writeback (WB)
    [+] OK so far. SMRR range base is programmed
    
    [*] Checking SMRR range mask programming..
    [*] IA32_SMRR_PHYSMASK = 0xFF800800 << SMRR Range Mask MSR (MSR 0x1F3)
        [11] Valid            = 1 << SMRR valid 
        [12] PhysMask         = FF800 << SMRR address range mask 
    [*] SMRR range mask: 0x00000000FF800000
    [+] OK so far. SMRR range is enabled
    
    [*] Verifying that SMRR range base & mask are the same on all logical CPUs..
    [CPU0] SMRR_PHYSBASE = 000000008D000006, SMRR_PHYSMASK = 00000000FF800800
    [+] OK so far. SMRR range base/mask match on all logical CPUs
    [*] Trying to read memory at SMRR base 0x8D000000..
    [+] PASSED: SMRR reads are blocked in non-SMM mode
    
    [+] PASSED: SMRR protection against cache attack is properly configured
#### common.spd_wd
    
    [*] running module: chipsec.modules.common.spd_wd
    [x][ =======================================================================
    [x][ Module: SPD Write Disable
    [x][ =======================================================================
    [+] PASSED: SPD Write Disable is set
#### common.spi_fdopss
    
    [*] running module: chipsec.modules.common.spi_fdopss
    [x][ =======================================================================
    [x][ Module: SPI Flash Descriptor Security Override Pin-Strap
    [x][ =======================================================================
    [*] HSFS = 0x0010E800 << Hardware Sequencing Flash Status Register (SPIBAR + 0x4)
        [00] FDONE            = 0 << Flash Cycle Done 
        [01] FCERR            = 0 << Flash Cycle Error 
        [02] AEL              = 0 << Access Error Log 
        [05] SCIP             = 0 << SPI cycle in progress 
        [11] WRSDIS           = 1 << Write status disable 
        [12] PR34LKD          = 0 << PRR3 PRR4 Lock-Down 
        [13] FDOPSS           = 1 << Flash Descriptor Override Pin-Strap Status 
        [14] FDV              = 1 << Flash Descriptor Valid 
        [15] FLOCKDN          = 1 << Flash Configuration Lock-Down 
        [16] FGO              = 0 << Flash cycle go 
        [17] FCYCLE           = 8 << Flash Cycle Type 
        [21] WET              = 0 << Write Enable Type 
        [24] FDBC             = 0 << Flash Data Byte Count 
        [31] FSMIE            = 0 << Flash SPI SMI# Enable 
    [+] PASSED: SPI Flash Descriptor Security Override is disabled
#### common.spi_lock
    
    [*] running module: chipsec.modules.common.spi_lock
    [x][ =======================================================================
    [x][ Module: SPI Flash Controller Configuration Locks
    [x][ =======================================================================
    [*] HSFS = 0x0010E800 << Hardware Sequencing Flash Status Register (SPIBAR + 0x4)
        [00] FDONE            = 0 << Flash Cycle Done 
        [01] FCERR            = 0 << Flash Cycle Error 
        [02] AEL              = 0 << Access Error Log 
        [05] SCIP             = 0 << SPI cycle in progress 
        [11] WRSDIS           = 1 << Write status disable 
        [12] PR34LKD          = 0 << PRR3 PRR4 Lock-Down 
        [13] FDOPSS           = 1 << Flash Descriptor Override Pin-Strap Status 
        [14] FDV              = 1 << Flash Descriptor Valid 
        [15] FLOCKDN          = 1 << Flash Configuration Lock-Down 
        [16] FGO              = 0 << Flash cycle go 
        [17] FCYCLE           = 8 << Flash Cycle Type 
        [21] WET              = 0 << Write Enable Type 
        [24] FDBC             = 0 << Flash Data Byte Count 
        [31] FSMIE            = 0 << Flash SPI SMI# Enable 
    [+] SPI write status disable set.
    [+] SPI Flash Controller configuration is locked
    [+] PASSED: SPI Flash Controller locked correctly.
#### common.uefi.access_uefispec
    
    [*] running module: chipsec.modules.common.uefi.access_uefispec
    [x][ =======================================================================
    [x][ Module: Access Control of EFI Variables
    [x][ =======================================================================
    [*] Testing UEFI variables ..
    [*] Variable Kernel_SiStatus (NV+BS+RT)
    [*] Variable WAND (NV+BS+RT)
    [*] Variable PlatformLangCodes (BS+RT)
    [*] Variable BootOrder (NV+BS+RT)
    [*] Variable SetupCpuFeatures (NV+BS+RT)
    [*] Variable move (BS)
    [*] Variable Kernel_WinSiStatus (NV+BS+RT)
    [*] Variable WakeUpType (NV+BS)
    [*] Variable InitSetupVariable (NV+BS)
    [*] Variable dbx (NV+BS+RT+TBAWS)
    [*] Variable CurrentActivePolicy (NV+BS)
    [*] Variable DmiVar0100010800 (NV+BS+RT)
    [*] Variable MACNamesListVar (NV+BS)
    [*] Variable ConOut (NV+BS+RT)
    [!] Found two instances of the variable 04ED33B3C3A6\0063.
    [*] Variable 04ED33B3C3A6\0063 (NV+BS)
    [*] Variable 04ED33B3C3A6\0063 (NV+BS)
    [*] Variable StdDefaults (NV+BS)
    [*] Variable lasterror (BS)
    [*] Variable NBPlatformData (BS)
    [*] Variable TcgInternalSyncFlag (NV+BS)
    [*] Variable PCRBitmap (NV+BS)
    [*] Variable AmiCpuSetupFeatures (BS)
    [*] Variable AmiGopPolicySetupData (BS)
    [*] Variable BootFlow (BS)
    [*] Variable CpuSetupVolatileData (BS+RT)
    [*] Variable cd\ (BS)
    [*] Variable ren (BS)
    [*] Variable BRDS (NV+BS+RT)
    [*] Variable DriverHealthCount (BS)
    [*] Variable uefishellversion (BS)
    [*] Variable md (BS)
    [!] Found two instances of the variable Setup.
    [*] Variable Setup (NV+BS)
    [*] Variable Setup (NV+BS)
    [*] Variable db (NV+BS+RT+TBAWS)
    [*] Variable NetworkStackVar (NV+BS)
    [!] Found two instances of the variable GPC.
    [*] Variable GPC (NV+BS+RT)
    [*] Variable GPC (NV+BS+RT)
    [*] Variable Kernel_RvkSiStatus (NV+BS+RT)
    [*] Variable OfflineUniqueIDEKPubCRC (NV+BS+RT)
    [*] Variable TPMPERBIOSFLAGS (NV+BS+RT)
    [*] Variable UsbMassDevNum (BS)
    [*] Variable DmiArray (NV+BS+RT)
    [*] Variable uefishellsupport (BS)
    [*] Variable WdtPersistentData (NV+BS)
    [*] Variable PreviousMemoryTypeInformation (NV+BS)
    [*] Variable SmbiosScratchBuffer (NV+BS+RT)
    [*] Variable ConOutDev (BS+RT)
    [*] Variable BugCheckParameter1 (NV+BS+RT)
    [*] Variable EvaluateDefaults4FirstBoot (NV+BS)
    [*] Variable DefaultBootOrder (NV+BS+RT)
    [*] Variable WGDS (NV+BS+RT)
    [*] Variable SmbiosEntryPointTable (NV+BS+RT)
    [*] Variable PlatformConfigurationChange (NV+BS)
    [*] Variable SecureBootSetup (NV+BS)
    [*] Variable SdioDevConfiguration (NV+BS)
    [*] Variable dbDefault (BS+RT)
    [*] Variable DriverManager (BS)
    [*] Variable UnlockIDCopy (NV+BS+RT)
    [*] Variable UsbMassDevValid (BS)
    [*] Variable PCI_COMMON (NV+BS)
    [*] Variable DynamicPageGroupCount (BS)
    [*] Variable UnlockID (NV+BS)
    [*] Variable IccAdvancedSetupDataVar (NV+BS)
    [*] Variable SinitSvn (NV+BS)
    [*] Variable PK (NV+BS+RT+TBAWS)
    [*] Variable cwd (BS)
    [*] Variable mem (BS)
    [*] Variable MaximumTableSize (NV+BS+RT)
    [*] Variable SPLC (NV+BS+RT)
    [*] Variable DmiVar0100010700 (NV+BS+RT)
    [*] Variable HSTI_RESULTS (NV+BS)
    [*] Variable DynamicPageCount (BS)
    [*] Variable MeBackupStorage (BS)
    [*] Variable SystemAccess (BS)
    [*] Variable PKDefault (BS+RT)
    [*] Variable FPDT_Volatile (BS+RT)
    [!] Found two instances of the variable 04ED33B3C3A6.
    [*] Variable 04ED33B3C3A6 (NV+BS)
    [*] Variable 04ED33B3C3A6 (NV+BS)
    [*] Variable 04ED33B3C3A6 (NV+BS)
    [*] Variable 04ED33B3C3A6 (NV+BS)
    [*] Variable nonesting (BS)
    [*] Variable BugCheckCode (NV+BS+RT)
    [*] Variable BootDebugPolicyApplied (NV+BS)
    [*] Variable Tpm20PCRallocateReset (NV+BS)
    [*] Variable BugCheckProgress (NV+BS+RT)
    [*] Variable HiiDB (BS+RT)
    [*] Variable Ep (NV+BS+RT)
    [*] Variable OfflineUniqueIDEKPub (NV+BS+RT)
    [*] Variable Boot0000 (NV+BS+RT)
    [*] Variable ConInDev (BS+RT)
    [*] Variable IsaIrqMask (BS)
    [*] Variable dir (BS)
    [*] Variable SetUpdateCountVar (NV+BS+RT)
    [*] Variable Boot0003 (NV+BS+RT)
    [*] Variable UsbControllerNum (BS)
    [*] Variable path (BS)
    [*] Variable AMITSESetup (NV+BS)
    [*] Variable DeploymentModeNv (NV+BS+RT)
    [*] Variable cd.. (BS)
    [*] Variable COM1 (NV+BS)
    [*] Variable TpmServFlags (NV+BS+RT)
    [*] Variable PlatformLastLangCodes (NV+BS)
    [*] Variable ErrOut (NV+BS+RT)
    [*] Variable MeSetupStorage (NV+BS)
    [*] Variable UsbSupport (NV+BS)
    [*] Variable VendorKeys (BS+RT)
    [*] Variable Boot0009 (NV+BS+RT)
    [*] Variable SIO_DEV_STATUS_VAR (BS)
    [*] Variable WriteOnceStatus (NV+BS+RT)
    [*] Variable uefiversion (BS)
    [*] Variable BootManager (BS)
    [*] Variable SIDSUPPORT (NV+BS+RT)
    [*] Variable OsIndicationsSupported (BS+RT)
    [*] Variable EWRD (NV+BS+RT)
    [*] Variable dbxDefault (BS+RT)
    [*] Variable WRDS (NV+BS+RT)
    [*] Variable ClientId (NV+BS)
    [*] Variable Timeout (NV+BS+RT)
    [*] Variable copy (BS)
    [*] Variable SmbiosV3EntryPointTable (NV+BS+RT)
    [*] Variable SignatureSupport (BS+RT)
    [*] Variable KEK (NV+BS+RT+TBAWS)
    [*] Variable HDDSecConfig (BS)
    [*] Variable UsbTypeC (NV+BS)
    [*] Variable mount (BS)
    [*] Variable AMITCGPPIVAR (NV+BS+RT)
    [*] Variable profiles (BS)
    [*] Variable cat (BS)
    [*] Variable MemoryTypeInformation (NV+BS)
    [*] Variable del (BS)
    [*] Variable SetupVolatileData (BS)
    [*] Variable SetupMode (BS+RT)
    [*] Variable TbtSetupVolatileData (BS+RT)
    [*] Variable IntUcode (NV+BS)
    [*] Variable WindowsBootChainSvn (NV+BS)
    [*] Variable MonotonicCounter (NV+BS+RT)
    [*] Variable ErrOutDev (BS+RT)
    [*] Variable Kernel_DriverSiStatus (NV+BS+RT)
    [*] Variable Boot0006 (NV+BS+RT)
    [*] Variable Boot0007 (NV+BS+RT)
    [*] Variable Boot0008 (NV+BS+RT)
    [*] Variable CpuSmm (NV+BS)
    [*] Variable MemoryOverwriteRequestControl (NV+BS+RT)
    [*] Variable MemoryOverwriteRequestControlLock (NV+BS+RT)
    [*] Variable KEKDefault (BS+RT)
    [*] Variable SecureBoot (BS+RT)
    [*] Variable WRDD (NV+BS+RT)
    [*] Variable BootNowCount (BS)
    [!] Found two instances of the variable 04ED33B3C3A6\0032.
    [*] Variable 04ED33B3C3A6\0032 (NV+BS)
    [*] Variable 04ED33B3C3A6\0032 (NV+BS)
    [*] Variable 04ED33B3C3A6\0032 (NV+BS)
    [*] Variable PBRDevicePath (NV+BS+RT)
    [*] Variable ChildHandleDpVar0 (BS)
    [*] Variable DriverHlthEnable (BS)
    [*] Variable DynamicPageGroupClass (BS)
    [*] Variable ConstructDefaults4FirstBoot (NV+BS)
    [!] Found two instances of the variable 04ED33B3C3A6\0001.
    [*] Variable 04ED33B3C3A6\0001 (NV+BS)
    [*] Variable 04ED33B3C3A6\0001 (NV+BS)
    [*] Variable 04ED33B3C3A6\0001 (NV+BS)
    [*] Variable RevocationList (NV+BS)
    [*] Variable ConIn (NV+BS+RT)
    [*] Variable MeInfoSetup (BS)
    [*] Variable CapsuleLongModeBuffer (NV+BS)
    [*] Variable BootingDeviceTypeInfo (NV+BS)
    [*] Variable FPDT_Variable_NV (NV+BS)
    [*] Variable AcousticVarName (NV+BS)
    [*] Variable CurrentPolicy (NV+BS+RT+TBAWS)
    [*] Variable RstOptaneConfig (NV+BS+RT)
    [*] Variable EfiTime (NV+BS+RT)
    [*] Variable Kernel_ATPSiStatus (NV+BS+RT)
    [!] Found two instances of the variable SADS.
    [*] Variable SADS (NV+BS+RT)
    [*] Variable SADS (NV+BS+RT)
    [*] Variable TerminalSerialVar (NV+BS)
    [*] Variable NBGopPlatformData (BS+RT)
    [*] Variable SaPegData (NV+BS)
    [*] Variable SOFTWAREGUARDSTATUS (BS+RT)
    [*] Variable Kernel_SkuSiStatus (NV+BS+RT)
    [*] Variable EsrtNonFmp (NV+BS)
    [*] Variable BootOptionSupport (BS+RT)
    [*] Variable EPCBIOS (NV+BS+RT)
    [*] Variable AcpiResetVar (NV+BS)
    [*] Variable AcpiGlobalVariable (NV+BS)
    [*] Variable VendorKeysNv (NV+BS)
    [*] Variable MemoryConfig (NV+BS)
    [*] Variable BootCurrent (BS+RT)
    [*] Variable PlatformLang (NV+BS+RT)
    
    [+] PASSED: All checked EFI variables are protected according to spec.

# Error:0

# Deprecated:0

# Skipped:1
#### common.sgx_check
    
    [*] running module: chipsec.modules.common.sgx_check
    ERROR: Currently this module cannot run within the EFI Shell. Exiting.
