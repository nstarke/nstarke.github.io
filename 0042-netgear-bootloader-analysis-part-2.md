# Netgear Bootloader Analysis: Part 2

Published: January 20, 2021

This is the second part of my analysis on the bootloader for the Netgear Nighthawk R9000 (x10) router.

## Static Comparison

In order to better understand the structure of the Netgear Bootloader and to have a point of reference, I compiled a similar U-Boot image using the `Bananapi` board as a reference board.  This will not result in parity to the Anna Purna Labs Alpine board that the Netgear router runs on, but it will be close enough to confirm a few things.

First things first though, compiling the `v2015.01` tag/version of U-Boot was not simple on a modern distribution; it required installing `gcc-4.9-arm-linux-gnueabihf` which hasn't been available via apt since Ubuntu Vivid.  To easily install this version of the GCC cross compiler via apt, I ran this command:

```
$ echo "deb http://old-releases.ubuntu.com/ubuntu vivid main" | sudo tee /etc/apt/sources.list.d/vivid.list
$ sudo apt update
$ sudo apt install gcc-4.9-arm-linux-gnueabihf
```

Then I linked the binary into place:
```
$ sudo ln -s /usr/bin/arm-linux-gnueabihf-gcc-4.9 /usr/bin/arm-linux-gnueabihf-gcc
```
Sym linking the binary is not the preferred method; using `update-alternatives` is a much better approach.

I needed a few dependencies to build u-boot:

```
$ sudo apt install python-dev python3-dev flex bison build-essential
```

There may be other dependencies that I had installed previous to this work that I might be missing.

Then to compile u-boot:
```
$ git clone https://github.com/u-boot/u-boot.git
$ git checkout v2015.01
$ ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make Bananapi_defconfig
$ ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make
```

At the end of the compilation process, a few binaries should be dropped on the filesystem.

```
$ file {u-boot,u-boot.bin}
u-boot:     ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /usr/lib/ld.so.1, with debug_info, not stripped
u-boot.bin: COM executable for DOS
```

The second file, `u-boot.bin`, is the primary file I used for comparison.  This file is stripped of all symbols, as opposed to the `u-boot` file which has a valid elf header and debug symbols.  I used it for a few comparison tasks.

The next step is to load `u-boot.bin` into Ghidra.  I needed to know the load address for the Bananapi board, which I found by grepping for the config key `CONFIG_SYS_TEXT_BASE` in `./spl/include/autoconf.mk`:

```
$ grep CONFIG_SYS_TEXT_BASE ./spl/include/autoconf.mk
CONFIG_SYS_TEXT_BASE=0x4a000000
```

This told me the load address is `0x4a000000`. Using this value, I load the binary into Ghidra.

One of the first things I was struck by was a similarity in the first few bytes of the `u-boot.bin` file.  After the first four bytes, which is a branch instruction, there are seven double words with the same value `0xE59FF014`. This is the same pattern as in the carved out Netgear bootloader.  

**BananaPi:**
```
                             //
                             // ram 
                             // ram:4a000000-ram:4a04df9f
                             //
             assume spsr = 0x0  (Default)
                             LAB_4a000000                                    XREF[6]:     FUN_4a0002c0:4a000314(*), 
                                                                                          4a000378(*), 
                                                                                          FUN_4a000d64:4a000e20(R), 
                                                                                          4a000e64(*), 4a003c30(*), 
                                                                                          4a003fe4(*)  
        4a000000 b8 00 00 ea     b          LAB_4a0002e8
                             DAT_4a000004                                    XREF[1]:     FUN_4a000d64:4a000e20(R)  
        4a000004 14 f0 9f e5     undefined4 E59FF014h
                             DAT_4a000008                                    XREF[1]:     FUN_4a000d64:4a000e20(R)  
        4a000008 14 f0 9f e5     undefined4 E59FF014h
                             DAT_4a00000c                                    XREF[1]:     FUN_4a000d64:4a000e20(R)  
        4a00000c 14 f0 9f e5     undefined4 E59FF014h
        4a000010 14 f0 9f e5     ddw        E59FF014h
        4a000014 14 f0 9f e5     ddw        E59FF014h
        4a000018 14 f0 9f e5     ddw        E59FF014h
        4a00001c 14 f0 9f e5     ddw        E59FF014h

```

**Netgear:**
```
                             //
                             // ram 
                             // ram:00100000-ram:0016df0f
                             //
             assume spsr = 0x0  (Default)
                             LAB_00100000                                    XREF[8]:     0010032c(*), 00100384(*), 
                                                                                          FUN_00100494:00100570(R), 
                                                                                          001005cc(*), 0010313a(*), 
                                                                                          FUN_00105d4c:00105d74(*), 
                                                                                          00113578(*), 0011384c(*)  
        00100000 be 00 00 ea     b          LAB_00100300
                             DAT_00100004                                    XREF[1]:     FUN_00100494:00100570(R)  
        00100004 14 f0 9f e5     undefined4 E59FF014h
                             DAT_00100008                                    XREF[1]:     FUN_00100494:00100570(R)  
        00100008 14 f0 9f e5     undefined4 E59FF014h
                             DAT_0010000c                                    XREF[1]:     FUN_00100494:00100570(R)  
        0010000c 14 f0 9f e5     undefined4 E59FF014h
                             LAB_00100010                                    XREF[4]:     00138e58(*), 00139758(*), 
                                                                                          0013975c(*), 00139760(*)  
        00100010 14 f0 9f e5     ldr        pc,[LAB_0010002c]
                             DWORD_00100014+1                                XREF[0,1]:   0013978c(*)  
        00100014 14 f0 9f e5     ddw        E59FF014h
        00100018 14 f0 9f e5     ddw        E59FF014h
        0010001c 14 f0 9f e5     ddw        E59FF014h

```

This leads me to understand that I did figure out the correct file offset for the Netgear image and that I have the base load address correct for the Netgear image.

In the opcode located in the first four bytes, the BananaPi image branches to address `0x4a0002e8` while the Netgear image branches to address `00100300`.  Looking at the function at these addresses, they are very similar in both images:

**BananaPi:**
```
                             LAB_4a0002e8                                    XREF[1]:     4a000000(j)  
        4a0002e8 12 00 00 eb     bl         FUN_4a000338                                     undefined FUN_4a000338()
        4a0002ec 00 00 0f e1     mrs        r0,cpsr
        4a0002f0 1f 10 00 e2     and        r1,r0,#0x1f
        4a0002f4 1a 00 31 e3     teq        r1,#0x1a
        4a0002f8 1f 00 c0 13     bicne      r0,r0,#0x1f
        4a0002fc 13 00 80 13     orrne      r0,r0,#0x13
        4a000300 c0 00 80 e3     orr        r0,r0,#0xc0
        4a000304 00 f0 29 e1     msr        cpsr_cf,r0
        4a000308 10 0f 11 ee     mrc        p15,0x0,r0,cr1,cr0,0x0
        4a00030c 02 0a c0 e3     bic        r0,r0,#0x2000
        4a000310 10 0f 01 ee     mcr        p15,0x0,r0,cr1,cr0,0x0
        4a000314 5c 00 9f e5     ldr        r0=>LAB_4a000000,[PTR_LAB_4a000378]              = 4a000000
        4a000318 10 0f 0c ee     mcr        p15,0x0,r0,cr12,cr0,0x0
        4a00031c 06 00 00 eb     bl         FUN_4a00033c                                     undefined FUN_4a00033c()
        4a000320 13 00 00 eb     bl         thunk_FUN_4a00037c                               undefined thunk_FUN_4a00037c()
        4a000324 8e 02 00 eb     bl         FUN_4a000d64                                     undefined FUN_4a000d64()
        4a000328 15 0f 07 ee     mcr        p15,0x0,r0,cr7,cr5,0x0
        4a00032c 9a 0f 07 ee     mcr        p15,0x0,r0,cr7,cr10,0x4
        4a000330 95 0f 07 ee     mcr        p15,0x0,r0,cr7,cr5,0x4
        4a000334 1e ff 2f e1     bx         lr
```

**Netgear:**

```
                             LAB_00100300                                    XREF[1]:     00100000(j)  
        00100300 10 00 00 eb     bl         FUN_00100348                                     undefined FUN_00100348()
        00100304 00 00 0f e1     mrs        r0,cpsr
        00100308 1f 10 00 e2     and        r1,r0,#0x1f
        0010030c 1a 00 31 e3     teq        r1,#0x1a
        00100310 1f 00 c0 13     bicne      r0,r0,#0x1f
        00100314 13 00 80 13     orrne      r0,r0,#0x13
        00100318 c0 00 80 e3     orr        r0,r0,#0xc0
        0010031c 00 f0 29 e1     msr        cpsr_cf,r0
        00100320 10 0f 11 ee     mrc        p15,0x0,r0,cr1,cr0,0x0
        00100324 02 0a c0 e3     bic        r0,r0,#0x2000
        00100328 10 0f 01 ee     mcr        p15,0x0,r0,cr1,cr0,0x0
        0010032c 50 00 9f e5     ldr        r0=>LAB_00100000,[PTR_LAB_00100384]              = 00100000
        00100330 10 0f 0c ee     mcr        p15,0x0,r0,cr12,cr0,0x0
        00100334 56 00 00 eb     bl         FUN_00100494                                     undefined FUN_00100494()
        00100338 15 0f 07 ee     mcr        p15,0x0,r0,cr7,cr5,0x0
        0010033c 9a 0f 07 ee     mcr        p15,0x0,r0,cr7,cr10,0x4
        00100340 95 0f 07 ee     mcr        p15,0x0,r0,cr7,cr5,0x4
        00100344 1e ff 2f e1     bx         lr

```

If you look at both of these functions, they only have minor differences in the labels referenced at `0x4a000314` / `0010032c`.  This doesn't reveal much about the function of the bootloader, but it confirms for me that the version I compiled closely matches the version I pulled off the Netgear router.  It anecdotally suggests that the Netgear image is not heavily modified, but a full bindiff would be necessary to definitively draw that conclusion.

## Bootloader Upgrade

One of the things I honed in on in the first part of this blog post is the string `NTGR`. The function that string is found in looks like this in the Ghidra decompiler:

```c++
void maybe_remote_bootloader_upgrade(ushort *param_1)

{
  int iVar1;
  ushort uVar2;
  code **ppcVar3;
  int *piVar4;
  int *piVar5;
  int *piVar6;
  undefined4 *puVar7;
  undefined4 uVar8;
  undefined4 *puVar9;
  undefined *puVar10;
  int iVar11;
  undefined *puVar12;
  code *pcVar13;
  uint uVar14;
  uint uVar15;
  ushort *puVar16;
  uint uVar17;
  int in_stack_00000000;
  ushort local_6c;
  undefined local_6a;
  undefined local_69;
  ushort local_68;
  undefined2 local_66;
  undefined2 local_64;
  undefined auStack98 [74];
  
  ppcVar3 = DAT_00136084;
  uVar17 = (uint)*(byte *)(param_1 + 1);
  if (in_stack_00000000 != 0x912) {
    return;
  }
  if (((*DAT_00136084 != (code *)0x0) &&
      (iVar11 = FUN_0010040e(0), *DAT_0013608c < (uint)(iVar11 - *DAT_00136088))) &&
     (*DAT_00136090 != uVar17)) {
    pcVar13 = *ppcVar3;
    *ppcVar3 = (code *)0x0;
    (*pcVar13)();
  }
  uVar15 = (uint)*param_1;
  if (uVar15 != 0) {
    return;
  }
  puVar16 = param_1 + 3;
  FUN_001317d0(&local_6c,0,0x52);
  local_6c = *param_1;
  local_68 = param_1[2];
  local_6a = *(undefined *)(param_1 + 1);
  local_69 = *(undefined *)((int)param_1 + 3);
  iVar11 = ((local_68 & 0xff) << 8 | (uint)(local_68 >> 8)) - 6;
  while (0 < iVar11) {
    iVar1 = uVar15 * 0xc;
    uVar15 = uVar15 + 1;
    FUN_00131812(auStack98 + iVar1,puVar16,(puVar16[1] & 0xff) << 8 | (uint)(puVar16[1] >> 8));
    uVar14 = (puVar16[1] & 0xff) << 8 | (uint)(puVar16[1] >> 8);
    iVar11 = iVar11 - uVar14;
    puVar16 = (ushort *)((int)puVar16 + uVar14);
  }
  local_66 = (undefined2)uVar15;
  local_64 = (undefined2)(uVar15 >> 0x10);
  if (iVar11 != 0) {
    return;
  }
  *DAT_00136094 = uVar17;
  if (uVar17 == *DAT_00136098) {
    return;
  }
  *DAT_00136098 = uVar17;
  puVar12 = PTR_s__NMRP_CLOSED_001360f8;
  piVar5 = DAT_001360f0;
  piVar4 = DAT_0013609c;
  switch(uVar17) {
  case 1:
    if (*DAT_0013609c != 0) {
      return;
    }
    iVar11 = FUN_00135a38(&local_6c,1);
    if (iVar11 == 0) {
      return;
    }
    iVar11 = FUN_0013188e(iVar11 + 4,PTR_s_NTGR_001360a0,
                          ((*(ushort *)(iVar11 + 2) & 0xff) << 8 | (uint)*(byte *)(iVar11 + 3)) - 4)
    ;
    if (iVar11 != 0) {
      return;
    }
    *piVar4 = 1;
    FUN_0010291c();
    puVar12 = PTR_s__NMRP_CONFIGING_001360a4;
    break;
  default:
    goto switchD_00135eec_caseD_2;
  case 3:
    if (*DAT_0013609c != 1) {
      return;
    }
    iVar11 = FUN_00135a38(&local_6c,2);
    if (iVar11 == 0) {
      return;
    }
    FUN_00131812(PTR_DAT_001360a8,iVar11 + 4,4);
    FUN_00131812(DAT_001360ac,iVar11 + 8,4);
    iVar11 = FUN_00135a38(&local_6c,4);
    if (iVar11 != 0) {
      maybe_error_out(PTR_s_Get_DEV-REGION_option,_value:0x%_001360b0,*(undefined2 *)(iVar11 + 4));
      puVar12 = PTR_s_Write_Region_Number_0x%04x_to_bo_001360b8;
      puVar16 = DAT_001360b4;
      uVar2 = (ushort)((*(ushort *)(iVar11 + 4) & 0xff) << 8) | *(ushort *)(iVar11 + 4) >> 8;
      *DAT_001360b4 = uVar2;
      maybe_error_out(puVar12,uVar2);
      FUN_00119c1c(*puVar16);
    }
    piVar5 = DAT_001360bc;
    iVar11 = FUN_00135a38(&local_6c,0x101);
    if (iVar11 == 0) {
      maybe_error_out(PTR_s__No_FW-UP_option_001360c4);
      *piVar5 = 0;
    }
    else {
      maybe_error_out(PTR_s__Recv_FW-UP_option_001360c0);
      *piVar5 = 1;
    }
    piVar6 = DAT_001360c8;
    iVar11 = FUN_00135a38(&local_6c,0x102);
    if (iVar11 == 0) {
      maybe_error_out(PTR_s__No_ST-UP_option_001360e4);
      *piVar6 = 0;
    }
    else {
      maybe_error_out(PTR_s__Recv_ST-UP_option_001360cc);
      uVar8 = DAT_001360d4;
      puVar7 = DAT_001360d0;
      *piVar6 = 1;
      puVar9 = DAT_001360d8;
      *puVar7 = 0;
      *puVar9 = 0;
      FUN_001317d0(uVar8,0,0x80);
      uVar17 = *(uint *)(iVar11 + 4);
      *DAT_001360dc =
           uVar17 << 0x18 | (uVar17 >> 8 & 0xff) << 0x10 | (uVar17 >> 0x10 & 0xff) << 8 |
           uVar17 >> 0x18;
      FUN_00135dec();
      maybe_error_out(PTR_s__Total_%d_String_Table_need_upda_001360e0,*puVar7);
    }
    puVar12 = PTR_s__NMRP_WAITING_FOR_UPLOAD_FIRMWAR_001360ec;
    puVar10 = PTR_s__No_firmware_update,_nor_string_t_001360e8;
    if ((*piVar5 == 0) && (*piVar6 == 0)) {
      *piVar4 = 5;
      puVar12 = puVar10;
    }
    else {
      *piVar4 = 2;
    }
    break;
  case 5:
    if (*DAT_0013609c != 5) {
      return;
    }
    *DAT_0013609c = 6;
    break;
  case 7:
    if ((*DAT_0013609c == 7) && (*DAT_001360f0 == 1)) {
      *DAT_001360f4 = *DAT_001360f4 + 0xf;
      *piVar5 = 0;
    }
    goto switchD_00135eec_caseD_2;
  }
  maybe_error_out(puVar12);
  maybe_tftp_os_firmware_recovery();
switchD_00135eec_caseD_2:
  return;
}

```

Based on this function it looks like it has something to do with updating the firmware (bootloader firmware as opposed to OS firmware).  This string is interesting because it signifies custom functionality that the manufacturer may have coded into the bootloader.  Of course, this string is not going to exist in the BananaPi version.  However, some of the other strings in the function might be in the BananaPi version I compiled, so I checked a few.  The substring `NMRP` is used several times in the function, but it likewise does not appear anywhere in the BananaPi version.  None of the other strings and substrings I checked corresponded to anything in the BananaPi image.  My next step was to look in the Netgear image at locations where this function at `0x00135e20` (`maybe_remote_bootloader_upgrade`) is called and see if those functions have strings that I can find in the BananaPi version. I could not find any common strings in the parent functions either.  This suggests to me that the function containing the `NTGR` string and its parent functions are custom modifications made by the manufacturer to the bootloader.

This function looks to me like a way to update the bootloader firmware remotely. I'm not sure of the specifics, but it seems like the `NTGR` string may be a magic string that is either sent over the network or embedded in the update image blob.

I think the only time this capability would be possible is if the primary Linux OS did not boot properly.  At that point the bootloader starts a TFTP server on the router and you can reflash the OS firmware by sending a firmware image over the ethernet ports to the router IP.  This is a known, useful function of U-Boot. Upgrading the bootloader in this fashion is probably a relic left over from the development process, where it might be necessary to remotely flash new images directly to a real board in order to test out changes. 

## U-Boot Shell
One of the other things I identified in the Netgear image is the function responsible for the U-Boot shell.  I found this function by looking for a prompt string.  U-Boot has the ability to customize the shell prompt: the default value is `uboot>` or `u-boot>`, but can be changed by the OEM.  In this case I searched for strings in Ghidra that included the `>` character, and I found `ALPINE_DB>` which looks like the shell prompt to me.  The function it is contained in looks like this:

```c++
undefined4 maybe_shell_prompt(undefined4 param_1,int param_2,char **param_3,char **param_4)

{
  char cVar1;
  uint uVar2;
  int iVar3;
  int iVar4;
  char *pcVar5;
  int iVar6;
  char *pcVar7;
  char *pcVar8;
  char *pcVar9;
  char *pcVar10;
  char *pcVar11;
  char **ppcVar12;
  char *pcVar13;
  int iVar14;
  char *pcVar15;
  int *piVar16;
  char *pcVar17;
  char *pcVar18;
  undefined *puVar19;
  char *apcStack184 [17];
  int iStack116;
  char *local_70 [18];
  undefined *local_28;
  
  pcVar10 = *param_3;
  pcVar11 = *param_4;
  uVar2 = maybe_prompt_output(param_1,PTR_s_ALPINE_DB>_00121ab4);
  if (uVar2 != 0) {
    return 0;
  }
  iVar3 = FUN_001316e0(param_2);
  if (0 < iVar3) {
    uVar2 = (uint)*(byte *)(param_2 + iVar3 + -1);
  }
  FUN_00131618(DAT_00121ab8,param_2);
  pcVar13 = DAT_00121ab8;
  pcVar9 = (char *)0x0;
  while( true ) {
    do {
      do {
        pcVar18 = pcVar13;
        cVar1 = *pcVar18;
        pcVar13 = pcVar18 + 1;
      } while (cVar1 == ' ');
    } while (cVar1 == '\t');
    pcVar15 = pcVar9;
    if (cVar1 == '\0') break;
    pcVar15 = pcVar9 + 1;
    apcStack184[(int)(pcVar9 + 1)] = pcVar18;
    do {
      pcVar13 = pcVar18;
      cVar1 = *pcVar13;
      if (cVar1 == '\0') goto LAB_0012185a;
    } while ((cVar1 != ' ') && (pcVar18 = pcVar13 + 1, cVar1 != '\t'));
    *pcVar13 = '\0';
    pcVar13 = pcVar13 + 1;
    pcVar9 = pcVar15;
    if (pcVar15 == &DataAbort) break;
  }
LAB_0012185a:
  local_70[0] = (char *)0x0;
  apcStack184[(int)(pcVar15 + 1)] = (char *)0x0;
  pcVar13 = pcVar15;
  ppcVar12 = (char **)PTR_PTR_s_base_00121abc;
  if (pcVar15 == (char *)0x0) {
    while ((char **)PTR_PTR_s_baudrate_00121ac0 != ppcVar12) {
      if (pcVar13 == (char *)0x12) {
        local_28 = PTR_s_..._00121ac4;
        pcVar13 = (char *)0x13;
        break;
      }
      local_70[(int)pcVar13] = *ppcVar12;
      pcVar13 = pcVar13 + 1;
      ppcVar12 = ppcVar12 + 7;
    }
LAB_0012193e:
    local_70[(int)pcVar13] = (char *)0x0;
    if (pcVar13 == (char *)0x0) goto LAB_00121954;
  }
  else {
    if (((pcVar15 == (char *)0x1) && ((uVar2 & 0xdf) != 0)) && (uVar2 != 9)) {
      iVar4 = maybe_execute_shell_command(apcStack184[1],0x2e);
      if (iVar4 == 0) {
        iVar4 = FUN_001316e0(apcStack184[1]);
      }
      else {
        iVar4 = iVar4 - (int)apcStack184[1];
      }
      pcVar13 = (char *)0x0;
      puVar19 = PTR_PTR_s_bdinfo_00121ad4;
      while (PTR_PTR_s_baudrate_00121ac0 != puVar19 + -0x1c) {
        iVar14 = FUN_001316e0(*(undefined4 *)(puVar19 + -0x1c));
        pcVar9 = pcVar13;
        if ((iVar4 <= iVar14) &&
           (iVar14 = FUN_0013188e(apcStack184[1],*(undefined4 *)(puVar19 + -0x1c),iVar4),
           iVar14 == 0)) {
          pcVar9 = pcVar13 + 1;
          if (0x11 < (int)pcVar13) {
            local_70[(int)pcVar13] = PTR_s_..._00121ac4;
            pcVar13 = pcVar9;
            break;
          }
          local_70[(int)pcVar13] = *(char **)(puVar19 + -0x1c);
        }
        puVar19 = puVar19 + 0x1c;
        pcVar13 = pcVar9;
      }
      goto LAB_0012193e;
    }
    iVar4 = FUN_00121640(apcStack184[1]);
    if ((iVar4 == 0) || (*(code **)(iVar4 + 0x18) == (code *)0x0)) {
      local_70[0] = (char *)0x0;
LAB_0012194e:
      if (pcVar15 != (char *)0x1) {
        return 0;
      }
      goto LAB_00121954;
    }
    pcVar13 = (char *)(**(code **)(iVar4 + 0x18))(pcVar15,apcStack184 + 1,uVar2,0x14,local_70);
    if (pcVar13 == (char *)0x0) goto LAB_0012194e;
  }
  pcVar18 = local_70[0];
  pcVar9 = PTR_s__00121acc;
  if (pcVar13 == (char *)0x1) {
    iVar4 = FUN_001316e0(apcStack184[(int)pcVar15]);
    pcVar18 = local_70[0] + iVar4;
    pcVar5 = (char *)FUN_001316e0(pcVar18);
  }
  else {
    if (((int)pcVar13 < 2) || (local_70[0] == (char *)0x0)) goto LAB_00121a4e;
    pcVar5 = (char *)FUN_001316e0(local_70[0]);
    ppcVar12 = local_70;
    while( true ) {
      ppcVar12 = ppcVar12 + 1;
      pcVar13 = *ppcVar12;
      if (pcVar13 == (char *)0x0) break;
      pcVar13 = pcVar13 + -1;
      pcVar9 = pcVar18;
      do {
        pcVar17 = pcVar9;
        if ((int)pcVar5 <= (int)(pcVar17 + -(int)pcVar18)) break;
        pcVar13 = pcVar13 + 1;
        pcVar9 = pcVar17 + 1;
      } while (*pcVar13 == *pcVar17);
      pcVar5 = pcVar17 + -(int)pcVar18;
    }
    if (pcVar5 == (char *)0x0) goto LAB_00121a4e;
    iVar4 = FUN_001316e0(apcStack184[(int)pcVar15]);
    pcVar5 = pcVar5 + -iVar4;
    if ((int)pcVar5 < 1) goto LAB_00121a4e;
    pcVar18 = local_70[0] + iVar4;
    pcVar9 = pcVar13;
  }
  if (pcVar18 != (char *)0x0) {
    pcVar15 = pcVar5 + (int)pcVar13;
    if ((int)(pcVar15 + (int)pcVar10) < 0x7fe) {
      pcVar7 = (char *)(iVar3 + param_2);
      pcVar17 = pcVar7 + -1;
      pcVar8 = pcVar18;
      while ((int)(pcVar8 + -(int)pcVar18) < (int)pcVar5) {
        pcVar17 = pcVar17 + 1;
        *pcVar17 = *pcVar8;
        pcVar8 = pcVar8 + 1;
      }
      if (-1 < (int)pcVar5) {
        pcVar7 = pcVar7 + (int)pcVar5;
      }
      if (pcVar9 != (char *)0x0) {
        iVar3 = 0;
        while (iVar3 < (int)pcVar13) {
          *pcVar7 = *pcVar9;
          iVar3 = 1;
          pcVar7 = pcVar7 + 1;
        }
      }
      *pcVar7 = '\0';
      FUN_0011f102(pcVar7 + -(int)pcVar15);
      if (pcVar9 == (char *)0x0) {
        FUN_0011f0e6(7);
      }
      *param_3 = pcVar15 + (int)pcVar10;
      *param_4 = pcVar15 + (int)pcVar11;
      return 1;
    }
LAB_00121954:
    FUN_0011f0e6(7);
    return 1;
  }
LAB_00121a4e:
  piVar16 = &iStack116;
  iVar14 = 0x4e;
  iVar3 = FUN_001316e0(PTR_DAT_00121ac8);
  iVar4 = FUN_001316e0(PTR_s__00121acc);
  while (piVar16 = piVar16 + 1, *piVar16 != 0) {
    iVar6 = FUN_001316e0();
    if (iVar14 + iVar6 + iVar4 < 0x4e) {
      FUN_0011f102(PTR_s__00121acc);
    }
    else {
      iVar14 = iVar3 - iVar4;
      FUN_0011f102(PTR_DAT_00121ad0);
      FUN_0011f102(PTR_DAT_00121ac8);
    }
    iVar14 = iVar14 + iVar6 + iVar4;
    FUN_0011f102(*piVar16);
  }
  maybe_error_out(PTR_DAT_00121ad0);
  FUN_0011f102(param_1);
  FUN_0011f102(param_2);
  return 1;
}

```

This function looks to me to output the string `ALPINE_DB>` to the console and then accept commands to execute.  It also looks to me to have tab completion capabilities, as well as the ability to output a list of available commands up to a certain limited number, whereby it then outputs an elipsis (`...`).  By analyzing this function I located the beginning of the list of shell functions (begins at address `0x00160384`).  

I found the `flash_contents_boot_mode_default_app_set` and `flash_contents_boot_mode_default_app_show` commands.  These seems to be commands which allow a shell user to change the bootloader which is loaded by the stage 1 or 2 bootloader (the u-boot image is the third stage bootloader which is referred to as stage 2.5).  If thats the case, I can't think of a better way to easily scaffold up an entirely attacker-controlled boot process - starting at the U-boot level. It also seems like a really easy way to brick a router permanently - past the point of repair via TFTP.  I'm not exactly sure what type of argument the set command expects, but I bet its something similar to the output of the show command.  It might be necessary to first flash the new bootloader to SPI, which I believe is done through the `flash_contents_obj_update` command.  **Warning** messing around with these commands will most likely brick your router.

My take aways from analyzing these two functions and then difference checking the BananaPi version versus the Netgear version is that there are definitely custom modifications to the netgear image that would not appear in a stock U-Boot image.  Furthermore, it seems as though the Netgear U-Boot instance can modify the stage 1 / 2 bootloaders from a U-Boot command shell.

My next steps (oh yes, this blog post will have a part 3 now) are to try to extend modifying the U-boot parameters to the host operating system in order to see if I can execute arbitrary U-Boot shell commands from the Linux OS.  If that is the case, I feel like it might be possible to develop a bootkit for this device without physical access. Of course to actually do so would require a lot of time and a stack of routers to burn through, but most importantly, it will require some dynamic analsys in the form of interacting with the U-Boot instance, even if it is only to read the console output.