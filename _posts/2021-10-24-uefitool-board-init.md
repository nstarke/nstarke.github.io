---
layout: posts
title:  "UEFITool and Reset Vector"
date:   2021-10-24 00:00:00 -0600
categories: uefi binary ghidra
author: Nicholas Starke
---

The class in reference in this post is OpenSecurityTraining2's Arch4001.  I will update this post with a link once the class is public.

**Update**
The link to the course is [https://p.ost2.fyi/courses/course-v1:OpenSecurityTraining2+Arch4001_x86-64_RVF+2021_v1/course/](https://p.ost2.fyi/courses/course-v1:OpenSecurityTraining2+Arch4001_x86-64_RVF+2021_v1/course/).  I **highly** recommend this course.

I recently took a class on the x86 reset vector, and one of the things I learned was that the CPU starts executing at `0xfffffff0`, which is the last 16 bytes of the 32-bit address space.  The class I took recommended using a Dell Optiplex 7010, so I purchased that model of PC with Windows 10 built into it.  I dumped the firmware using Chipsec, and loaded the dumped binary file into UEFITool. The firmware version was `A29`, which is the latest as of this writing.  I wanted to see where in the UEFI image the reset vector code was located.  I found it as the last item after expanding the PEI phase modules. 

![UEFITool Reset Vector Code](/images/10242021/uefitool-sec.PNG "UEFITool Reset Vector Code")

Note the `Data Address` value of `0xffffffc0` in the image.

There are two components here:

* A Raw binary stub
* A PE32 file

Let's look at both in more detail.

## Raw Binary Stub

![UEFITool Reset Vector Bytes](/images/10242021/hex-view.PNG "UEFITool Reset Vector Bytes")

Here is the contents of the Raw Binary Stub:

```
00000000: 4400 0019 ffff ffff ffff ffff 0000 0000  D...............
00000010: 0000 0000 bf50 41eb 1d00 0000 0000 0000  .....PA.........
00000020: 0000 0000 042b f3ff 0000 0000 0000 0000  .....+..........
00000030: 0000 0000 9090 e93b f800 0000 0000 0000  .......;........
00000040: 0000 f0ff                                ....
```

The last 16 bytes correspond to the x86 reset vector.  Let's disassemble those bytes:

```
$ objdump -D -b binary -mi386 --adjust-vma=0xfffffff0 reset-tail.bin

reset-tail.bin:     file format binary


Disassembly of section .data:

fffffff0 <.data>:
fffffff0:       90                      nop
fffffff1:       90                      nop
fffffff2:       e9 3b f8 00 00          jmp    0xf832
fffffff7:       00 00                   add    %al,(%eax)
fffffff9:       00 00                   add    %al,(%eax)
fffffffb:       00 00                   add    %al,(%eax)
fffffffd:       00 f0                   add    %dh,%al
ffffffff:       ff                      .byte 0xff
```

This code is executed in 16-bit real mode.  The first two bytes are NOPs (`0x90`) and then a jump to `0xf832`.  In reality, this will jump to `0xfffff832` because of how the code segment (`cs`) register is set at reset.  So what exists at `0xfffff832`? If you look at the offsets for the PE32 file that is part of SEC CORE, you will see the following metadata (on the right of the screenshot):

![UEFITool SEC CORE PE32 Metadata](/images/10242021/pe32-metadata.PNG "UEFITool SEC CORE PE32 Metadata")

Specifically, we are interested in `Image Base` (`0xFFFFF4FC`) and `Full Size` (`0xAC4`).  Add these together and what do we get?

```
>>> hex(0xFFFFF4FC + 0xAC4)
'0xffffffc0'
```

`0xffffffc0` is the `Data Address` Value from the first Raw Binary image metadata above.  That means the PE32 file will be directly before the Raw Binary image in memory.  Since `0xfffff832` is less than `0xffffffc0` but more than `0xFFFFF4FC` we know that the reset vector code is jumping into the PE32 program directly before it in memory.

## PE32 File

I right click in UEFITool on the PE32 image section in SEC CORE and select `Save body`.  This exports the PE32 image section as a file with a PE32 header but without the UEFI module header.  Let's open the exported PE32 file in Ghidra.

Once we have loaded the file and run the auto analysis, we have some work to do.  The first thing is that Ghidra maps the base address to `0x80000000` when we know from UEFITool that the base address is `0xFFFFF4FC`. So, we need to change the memory map to set the base address to `0xFFFFF4FC`.

![Ghidra Memory Map](/images/10242021/memory-map.PNG "Ghidra Memory Map")

Once this is set, we need to disassemble consecutive bytes that are not the values `0xff` or `0xcc`.  I do not know why this is but I suspect it is some sort of padding mechanism.

Once we have the majority of the PE32 disassembled in Ghidra, let's look at `0xfffff832` which is where the Raw Binary jumps to:

```
                             *************************************************************
                             *                           FUNCTION                          
                             *************************************************************
                             EFI_STATUS  __stdcall  ModuleEntryPoint (EFI_HANDLE  ImageH
             EFI_STATUS        EAX:4          <RETURN>
             EFI_HANDLE        Stack[0x4]:4   ImageHandle
             EFI_SYSTEM_TAB    Stack[0x8]:4   SystemTable
                             ModuleEntryPoint                                XREF[2]:     Entry Point (*) , fffff5dc (*)   
        fffff830 db  e3           FNINIT
        fffff832 0f  6e  c0       MOVD       MM0 ,EAX
        fffff835 fa              CLI
        fffff836 b8  00  f0       MOV        EAX ,0xd88ef000
                 8e  d8
        fffff83b be  f0  ff       MOV        ESI ,0x3c80fff0
                 80  3c
        fffff840 ea  75  05       JMPF       0xe0 :0x5bea0575 =>LAB_5be8fa71
                 ea  5b  e0  00
        fffff847 f0              LOCK
        fffff848 66  bb  84  f9    MOV        BX,0xf984

```

Looks like this jumps to two bytes after the `ModuleEntryPoint` that Ghidra identified.  A few values are moved into `EAX` and `ESI`, interrupts are cleared (`cli`) and then there is a far jump `jmpf` at `fffff840`.  Without analyzing the rest of the UEFI image it's hard to know where exactly this goes to.  However, the disassembly continues further down.  The full disassembly is at the bottom of this document, but it looks like this module is responsible for exiting real mode and setting up the initial PCIe configuration using Port IO.  At this point in early boot, there really aren't functions yet available because there is no stack yet. The one "function" that Ghidra defines is the module entry point, which is derived from the metadata in the UEFI header.


## Full Disassembly (Ghidra)

```
                             //
                             // Headers 
                             // ram:fffff4fc-ram:fffff6fb
                             //
             assume DF = 0x0  (Default)
                             IMAGE_DOS_HEADER_fffff4fc.e_program[0]          XREF[0,6]:   fffffb27 (*) , fffffe50 (*) , 
                             IMAGE_DOS_HEADER_fffff4fc.e_program[8]                       fffffe6a (*) , fffffe83 (*) , 
                             IMAGE_DOS_HEADER_fffff4fc.e_program[32]                      fffffe95 (*) , fffffea7 (*)   
                             IMAGE_DOS_HEADER_fffff4fc.e_program[40]
        fffff4fc 4d  5a  00       IMAGE_DO                                                    Magic number
                 00  00  00 
                 00  00  00 
        fffff5b4 50  45  00       IMAGE_NT                                                    = 
                 00  4c  01 
                 02  00  e2 
        fffff6ac 53  54  41       IMAGE_SE                                                    STARTUP_
                 52  54  55 
                 50  5f  93 
        fffff6d4 2e  72  65       IMAGE_SE                                                    .reloc
                 6c  6f  63 
                 00  00  20 
                             //
                             // STARTUP_ 
                             // ram:fffff6fc-ram:ffffff9b
                             //
                             CPU_MICROCODE_FILE_GUID                         XREF[3]:     fffff5e0 (*) , fffff6b8 (*) , 
                                                                                          fffff769 (R)   
        fffff6fc 72  85  08       EFI_GUID
                 17  7f  37 
                 ef  44  8f 
           fffff6fc 72  85  08  17    UINT32    17088572h               Data1                             XREF[3]:     fffff5e0 (*) , fffff6b8 (*) , 
                                                                                                                     fffff769 (R)   
           fffff700 7f  37           UINT16    377Fh                   Data2
           fffff702 ef  44           UINT16    44EFh                   Data3
           fffff704 8f  4e  b0  9f  ff  UINT8[8]  8Fh,"N",B0h,9Fh,FFh,"F  Data4
                    46  a0  70
                             AMI_EARLY_BIST_PPI_GUID
        fffff70c 72  ce  e2       EFI_GUID
                 a7  32  dc 
                 c0  4b  9e 
                             LAB_fffff71c                                    XREF[3]:     fffff8bb (*) , fffff8c7 (R) , 
                                                                                          fffff930 (R)   
        fffff71c 10  00           ADC        byte ptr [EAX ],AL
        fffff71e 00  80  0c       ADD        byte ptr [EAX  + 0xfffff70c ],AL
                 f7  ff  ff
        fffff724 00  00           ADD        byte ptr [EAX ],AL
        fffff726 00  00           ADD        byte ptr [EAX ],AL
                             LAB_fffff728                                    XREF[1]:     fffffae3 (j)   
        fffff728 bb  00  00       MOV        EBX ,0xffa50000
                 a5  ff
        fffff72d 81  7b  28       CMP        dword ptr [EBX  + 0x28 ]=>DAT_7fa4f524 ,0x4856465f
                 5f  46  56  48
        fffff734 0f  85  ef       JNZ        LAB_fffff829
                 00  00  00
        fffff73a 8b  d3           MOV        EDX ,EBX
        fffff73c 8b  43  30       MOV        EAX ,dword ptr [EBX  + 0x30 ]=>DAT_7fa4f52c
        fffff73f 25  ff  ff       AND        EAX ,0xffff
                 00  00
        fffff744 03  d8           ADD        EBX ,EAX
        fffff746 0f  82  dd       JC         LAB_fffff829
                 00  00  00
        fffff74c 03  52  20       ADD        EDX ,dword ptr [EDX  + 0x20 ]=>DAT_7fa4f51c
        fffff74f 74  06           JZ         LAB_fffff757
        fffff751 0f  82  d2       JC         LAB_fffff829
                 00  00  00
                             LAB_fffff757                                    XREF[1]:     fffff74f (j)   
        fffff757 4a              DEC        EDX
                             LAB_fffff758                                    XREF[1]:     fffff785 (j)   
        fffff758 83  3b  ff       CMP        dword ptr [EBX ],-0x1
        fffff75b 74  2a           JZ         LAB_fffff787
        fffff75d b9  04  00       MOV        ECX ,0x4
                 00  00
        fffff762 8b  f3           MOV        ESI ,EBX
        fffff764 bf  fc  f6       MOV        EDI ,0xfffff6fc
                 ff  ff
        fffff769 f3  a7           CMPSD.RE   ES:EDI =>DAT_7fffebf8 ,ESI                        = 
        fffff76b 74  1f           JZ         LAB_fffff78c
        fffff76d 8b  43  14       MOV        EAX ,dword ptr [EBX  + 0x14 ]
        fffff770 25  ff  ff       AND        EAX ,0xffffff
                 ff  00
        fffff775 03  d8           ADD        EBX ,EAX
        fffff777 83  c3  07       ADD        EBX ,0x7
        fffff77a 0f  82  a9       JC         LAB_fffff829
                 00  00  00
        fffff780 83  e3  f8       AND        EBX ,0xfffffff8
        fffff783 3b  da           CMP        EBX ,EDX
        fffff785 72  d1           JC         LAB_fffff758
                             LAB_fffff787                                    XREF[1]:     fffff75b (j)   
        fffff787 e9  9d  00       JMP        LAB_fffff829
                 00  00
                             LAB_fffff78c                                    XREF[1]:     fffff76b (j)   
        fffff78c 8b  fb           MOV        EDI ,EBX
        fffff78e 8b  43  14       MOV        EAX ,dword ptr [EBX  + 0x14 ]
        fffff791 25  ff  ff       AND        EAX ,0xffffff
                 ff  00
        fffff796 03  f8           ADD        EDI ,EAX
        fffff798 0f  82  8b       JC         LAB_fffff829
                 00  00  00
        fffff79e 83  c3  18       ADD        EBX ,0x18
        fffff7a1 0f  82  82       JC         LAB_fffff829
                 00  00  00
        fffff7a7 8b  f3           MOV        ESI ,EBX
        fffff7a9 b8  01  00       MOV        EAX ,0x1
                 00  00
        fffff7ae 0f  a2           CPUID
        fffff7b0 8b  d8           MOV        EBX ,EAX
        fffff7b2 b9  17  00       MOV        ECX ,0x17
                 00  00
        fffff7b7 0f  32           RDMSR
        fffff7b9 c1  ea  12       SHR        EDX ,0x12
        fffff7bc 80  e2  07       AND        DL,0x7
        fffff7bf 8a  ca           MOV        CL,DL
        fffff7c1 b2  01           MOV        DL,0x1
        fffff7c3 d2  e2           SHL        DL,CL
        fffff7c5 87  de           XCHG       ESI ,EBX
                             LAB_fffff7c7                                    XREF[1]:     fffff827 (j)   
        fffff7c7 3b  df           CMP        EBX ,EDI
        fffff7c9 73  5e           JNC        LAB_fffff829
        fffff7cb 83  3b  01       CMP        dword ptr [EBX ],0x1
        fffff7ce 75  59           JNZ        LAB_fffff829
        fffff7d0 b9  00  08       MOV        ECX ,0x800
                 00  00
        fffff7d5 83  7b  1c  00    CMP        dword ptr [EBX  + 0x1c ],0x0
        fffff7d9 74  03           JZ         LAB_fffff7de
        fffff7db 8b  4b  20       MOV        ECX ,dword ptr [EBX  + 0x20 ]
                             LAB_fffff7de                                    XREF[1]:     fffff7d9 (j)   
        fffff7de 3b  73  0c       CMP        ESI ,dword ptr [EBX  + 0xc ]
        fffff7e1 75  07           JNZ        LAB_fffff7ea
        fffff7e3 8b  c3           MOV        EAX ,EBX
        fffff7e5 84  53  18       TEST       byte ptr [EBX  + 0x18 ],DL=>DAT_7fa4f52c
        fffff7e8 75  41           JNZ        LAB_fffff82b
                             LAB_fffff7ea                                    XREF[1]:     fffff7e1 (j)   
        fffff7ea 8b  6b  20       MOV        EBP ,dword ptr [EBX  + 0x20 ]
        fffff7ed 8b  43  1c       MOV        EAX ,dword ptr [EBX  + 0x1c ]
        fffff7f0 83  c0  30       ADD        EAX ,0x30
        fffff7f3 3b  e8           CMP        EBP ,EAX
        fffff7f5 76  20           JBE        LAB_fffff817
        fffff7f7 8b  0c  18       MOV        ECX ,dword ptr [EAX  + EBX *0x1 ]
        fffff7fa 83  f9  14       CMP        ECX ,0x14
        fffff7fd 73  2a           JNC        LAB_fffff829
        fffff7ff 8d  6c  18  14    LEA        EBP ,[EAX  + EBX *0x1  + 0x14 ]
                             LAB_fffff803                                    XREF[1]:     fffff812 (j)   
        fffff803 39  75  00       CMP        dword ptr [EBP ],ESI
        fffff806 75  07           JNZ        LAB_fffff80f
        fffff808 8b  c3           MOV        EAX ,EBX
        fffff80a 84  55  04       TEST       byte ptr [EBP  + 0x4 ],DL
        fffff80d 75  1c           JNZ        LAB_fffff82b
                             LAB_fffff80f                                    XREF[1]:     fffff806 (j)   
        fffff80f 83  c5  0c       ADD        EBP ,0xc
        fffff812 e2  ef           LOOP       LAB_fffff803
        fffff814 8b  4b  20       MOV        ECX ,dword ptr [EBX  + 0x20 ]
                             LAB_fffff817                                    XREF[1]:     fffff7f5 (j)   
        fffff817 81  c1  ff       ADD        ECX ,0x7ff
                 07  00  00
        fffff81d 81  e1  00       AND        ECX ,0xfffff800
                 f8  ff  ff
        fffff823 03  d9           ADD        EBX ,ECX
        fffff825 72  02           JC         LAB_fffff829
        fffff827 eb  9e           JMP        LAB_fffff7c7
                             LAB_fffff829                                    XREF[11]:    fffff734 (j) , fffff746 (j) , 
                                                                                          fffff751 (j) , fffff77a (j) , 
                                                                                          fffff787 (j) , fffff798 (j) , 
                                                                                          fffff7a1 (j) , fffff7c9 (j) , 
                                                                                          fffff7ce (j) , fffff7fd (j) , 
                                                                                          fffff825 (j)   
        fffff829 33  c0           XOR        EAX ,EAX
                             LAB_fffff82b                                    XREF[2]:     fffff7e8 (j) , fffff80d (j)   
        fffff82b e9  b8  02       JMP        LAB_fffffae8
                 00  00
                             *************************************************************
                             *                           FUNCTION                          
                             *************************************************************
                             EFI_STATUS  __stdcall  ModuleEntryPoint (EFI_HANDLE  ImageH
             EFI_STATUS        EAX:4          <RETURN>
             EFI_HANDLE        Stack[0x4]:4   ImageHandle
             EFI_SYSTEM_TAB    Stack[0x8]:4   SystemTable
                             ModuleEntryPoint                                XREF[2]:     Entry Point (*) , fffff5dc (*)   
        fffff830 db  e3           FNINIT
        fffff832 0f  6e  c0       MOVD       MM0 ,EAX
        fffff835 fa              CLI
        fffff836 b8  00  f0       MOV        EAX ,0xd88ef000
                 8e  d8
        fffff83b be  f0  ff       MOV        ESI ,0x3c80fff0
                 80  3c
        fffff840 ea  75  05       JMPF       0xe0 :0x5bea0575 =>LAB_5be8fa71
                 ea  5b  e0  00
        fffff847 f0              LOCK
        fffff848 66  bb  84  f9    MOV        BX,0xf984
        fffff84c ff              ??         FFh
        fffff84d ff              ??         FFh
        fffff84e 66  2e  0f       LGDT       word ptr CS:[EDI ]
                 01  17
        fffff853 0f  20  c0       MOV        EAX ,CR0
        fffff856 0c  01           OR         AL,0x1
        fffff858 0f  22  c0       MOV        CR0 ,EAX
        fffff85b fc              CLD
        fffff85c b8  08  00       MOV        EAX ,0xd88e0008
                 8e  d8
        fffff861 8e  c0           MOV        ES,AX
        fffff863 8e  d0           MOV        SS,AX
        fffff865 8e  e0           MOV        FS,AX
        fffff867 8e  e8           MOV        GS,AX
        fffff869 66  ea  71       JMPF       offset  LAB_8010ed5d
                 f8  ff  ff
        fffff86f 10  00           ADC        byte ptr [EAX ],AL
        fffff871 b9  a0  01       MOV        ECX ,0x1a0
                 00  00
        fffff876 0f  32           RDMSR
        fffff878 0f  ba  f0  16    BTR        EAX ,0x16
        fffff87c 73  02           JNC        LAB_fffff880
        fffff87e 0f  30           WRMSR
                             LAB_fffff880                                    XREF[1]:     fffff87c (j)   
        fffff880 b9  1b  00       MOV        ECX ,0x1b
                 00  00
        fffff885 0f  32           RDMSR
        fffff887 83  e2  f0       AND        EDX ,0xfffffff0
        fffff88a 25  ff  0f       AND        EAX ,0xfff
                 00  00
        fffff88f 0d  00  00       OR         EAX ,0xfee00000
                 e0  fe
        fffff894 0f  30           WRMSR
        fffff896 0f  20  e0       MOV        EAX ,CR4
        fffff899 66  0d  00  06    OR         AX,0x600
        fffff89d 0f  22  e0       MOV        CR4 ,EAX
        fffff8a0 e9  c7  06       JMP        LAB_ffffff6c
                 00  00
                             LAB_fffff8a5                                    XREF[1]:     ffffff8a (j)   
        fffff8a5 0f  7e  c0       MOVD       EAX ,MM0
        fffff8a8 50              PUSH       EAX =>DAT_ff91fffc
        fffff8a9 b8  20  00       MOV        EAX ,0xfee00020
                 e0  fe
        fffff8ae 8b  18           MOV        EBX ,dword ptr [EAX ]=>DAT_fee00020
        fffff8b0 c1  eb  18       SHR        EBX ,0x18
        fffff8b3 53              PUSH       EBX =>DAT_ff91fff8
        fffff8b4 6a  01           PUSH       offset  DAT_ff91fff4
        fffff8b6 8b  dc           MOV        EBX ,ESP
        fffff8b8 83  ec  0c       SUB        ESP ,0xc
        fffff8bb be  1c  f7       MOV        ESI ,LAB_fffff71c
                 ff  ff
        fffff8c0 8b  fc           MOV        EDI ,ESP
        fffff8c2 b9  03  00       MOV        ECX ,0x3
                 00  00
        fffff8c7 f3  a5           MOVSD.REP  ES:EDI =>DAT_ff91ffe8 ,ESI =>LAB_fffff71c
        fffff8c9 89  5c  24  08    MOV        dword ptr [ESP  + 0x8 ]=>DAT_ff91fff0 ,EBX =>DAT_f
        fffff8cd 8b  dc           MOV        EBX ,ESP
        fffff8cf 68  7f  03       PUSH       offset  DAT_ff91ffe4
                 00  00
        fffff8d4 d9  2c  24       FLDCW      word ptr [ESP ]=>DAT_ff91ffe4
        fffff8d7 83  c4  04       ADD        ESP ,0x4
        fffff8da 68  00  00       PUSH       offset  DAT_ff91ffe4
                 90  ff
        fffff8df 53              PUSH       EBX =>DAT_ff91ffe0
        fffff8e0 68  00  00       PUSH       offset  DAT_ff91ffdc
                 02  00
        fffff8e5 ff  35  fc       PUSH       dword ptr [DAT_fffffffc ]
                 ff  ff  ff
        fffff8eb ff  15  e0       CALL       dword ptr [DAT_ffffffe0 ]
                 ff  ff  ff
        fffff8f1 8d  a4  24       LEA        ESP ,[ESP ]
                 00  00  00  00
        fffff8f8 8d  64  24  00    LEA        ESP ,[ESP ]
        fffff8fc ff  02           INC        dword ptr [EDX ]
        fffff8fe 00  02           ADD        byte ptr [EDX ],AL
        fffff900 01  02           ADD        dword ptr [EDX ],EAX
        fffff902 02  02           ADD        AL,byte ptr [EDX ]
        fffff904 03  02           ADD        EAX ,dword ptr [EDX ]
        fffff906 04  02           ADD        AL,0x2
        fffff908 05  02  06       ADD        EAX ,0x7020602
                 02  07
        fffff90d 02  08           ADD        CL,byte ptr [EAX ]
        fffff90f 02  09           ADD        CL,byte ptr [ECX ]
        fffff911 02  0a           ADD        CL,byte ptr [EDX ]
        fffff913 02  0b           ADD        CL,byte ptr [EBX ]=>DAT_ff91ffe8
        fffff915 02  0c  02       ADD        CL,byte ptr [EDX  + EAX *0x1 ]
        fffff918 0d  02  0e       OR         EAX ,0xf020e02
                 02  0f
        fffff91d 02  50  02       ADD        DL,byte ptr [EAX  + 0x2 ]
        fffff920 58              POP        EAX
        fffff921 02  59  02       ADD        BL,byte ptr [ECX  + 0x2 ]
        fffff924 68  02  69       PUSH       0x6a026902
                 02  6a
        fffff929 02  6b  02       ADD        CH,byte ptr [EBX  + 0x2 ]
        fffff92c 6c              INSB       ES:EDI =>DAT_ff91ffe8 ,DX
        fffff92d 02  6d  02       ADD        CH,byte ptr [EBP  + 0x2 ]
        fffff930 6e              OUTSB      DX,ESI =>LAB_fffff71c
        fffff931 02  6f  02       ADD        CH,byte ptr [EDI  + 0x2 ]=>DAT_ff91ffeb
        fffff934 8d  a4  24       LEA        ESP ,[ESP ]
                 00  00  00  00
        fffff93b 90              NOP
        fffff93c 00  00           ADD        byte ptr [EAX ],AL
        fffff93e 00  00           ADD        byte ptr [EAX ],AL
        fffff940 00  00           ADD        byte ptr [EAX ],AL
        fffff942 00  00           ADD        byte ptr [EAX ],AL
        fffff944 ff              ??         FFh
        fffff945 ff              ??         FFh
        fffff946 00  00           ADD        byte ptr [EAX ],AL
        fffff948 00  93  cf       ADD        byte ptr [EBX  + 0xffff00cf ],DL
                 00  ff  ff
        fffff94e 00  00           ADD        byte ptr [EAX ],AL
        fffff950 00  9b  cf       ADD        byte ptr [EBX  + 0xffff00cf ],BL
                 00  ff  ff
        fffff956 00  00           ADD        byte ptr [EAX ],AL
        fffff958 00  93  cf       ADD        byte ptr [EBX  + 0xffff00cf ],DL
                 00  ff  ff
        fffff95e 00  00           ADD        byte ptr [EAX ],AL
        fffff960 00  9b  cf       ADD        byte ptr [EBX  + 0xcf ],BL
                 00  00  00
        fffff966 00  00           ADD        byte ptr [EAX ],AL
        fffff968 00  00           ADD        byte ptr [EAX ],AL
        fffff96a 00  00           ADD        byte ptr [EAX ],AL
        fffff96c ff              ??         FFh
        fffff96d ff              ??         FFh
        fffff96e 00  00           ADD        byte ptr [EAX ],AL
        fffff970 00  93  cf       ADD        byte ptr [EBX  + 0xffff00cf ],DL
                 00  ff  ff
        fffff976 00  00           ADD        byte ptr [EAX ],AL
        fffff978 00  9b  af       ADD        byte ptr [EBX  + 0xaf ],BL
                 00  00  00
        fffff97e 00  00           ADD        byte ptr [EAX ],AL
        fffff980 00  00           ADD        byte ptr [EAX ],AL
        fffff982 00  00           ADD        byte ptr [EAX ],AL
        fffff984 47              INC        EDI
        fffff985 00  3c  f9       ADD        byte ptr [ECX  + EDI *0x8 ],BH
        fffff988 ff              ??         FFh
        fffff989 ff              ??         FFh
        fffff98a cc              ??         CCh
        fffff98b cc              ??         CCh
        fffff98c 0f  6e  cb       MOVD       MM1 ,EBX
        fffff98f 0f  6e  d1       MOVD       MM2 ,ECX
        fffff992 0f  6e  da       MOVD       MM3 ,EDX
        fffff995 66  8b  d8       MOV        BX,AX
        fffff998 66  c1  eb  0c    SHR        BX,0xc
        fffff99c 66  83  fb  08    CMP        BX,0x8
        fffff9a0 0f  87  dd       JA         LAB_fffffa83
                 00  00  00
        fffff9a6 66  83  fb  00    CMP        BX,0x0
        fffff9aa 0f  84  d3       JZ         LAB_fffffa83
                 00  00  00
        fffff9b0 66  8b  c8       MOV        CX,AX
        fffff9b3 66  83  e1  07    AND        CX,0x7
        fffff9b7 66  25  ff  0f    AND        AX,0xfff
        fffff9bb 66  c1  e8  03    SHR        AX,0x3
        fffff9bf 86  e0           XCHG       AL,AH
        fffff9c1 b0  0d           MOV        AL,0xd
        fffff9c3 0c  80           OR         AL,0x80
        fffff9c5 e6  70           OUT        0x70 ,AL
        fffff9c7 eb  00           JMP        LAB_fffff9c9
                             LAB_fffff9c9                                    XREF[1]:     fffff9c7 (j)   
        fffff9c9 eb  00           JMP        LAB_fffff9cb
                             LAB_fffff9cb                                    XREF[1]:     fffff9c9 (j)   
        fffff9cb eb  00           JMP        LAB_fffff9cd
                             LAB_fffff9cd                                    XREF[1]:     fffff9cb (j)   
        fffff9cd eb  00           JMP        LAB_fffff9cf
                             LAB_fffff9cf                                    XREF[1]:     fffff9cd (j)   
        fffff9cf e4  71           IN         AL,0x71
        fffff9d1 a8  80           TEST       AL,0x80
        fffff9d3 75  09           JNZ        LAB_fffff9de
        fffff9d5 66  b8  12  00    MOV        AX,0x12
        fffff9d9 e9  a9  00       JMP        LAB_fffffa87
                 00  00
                             LAB_fffff9de                                    XREF[1]:     fffff9d3 (j)   
        fffff9de 86  e0           XCHG       AL,AH
        fffff9e0 c1  c9  10       ROR        ECX ,0x10
        fffff9e3 66  33  c9       XOR        CX,CX
        fffff9e6 eb  09           JMP        LAB_fffff9f1
                             LAB_fffff9e8                                    XREF[1]:     fffff9f5 (j)   
        fffff9e8 66  d1  e1       SHL        CX,1
        fffff9eb 66  83  c9  01    OR         CX,0x1
        fffff9ef 66  4b           DEC        BX
                             LAB_fffff9f1                                    XREF[1]:     fffff9e6 (j)   
        fffff9f1 66  83  fb  00    CMP        BX,0x0
        fffff9f5 77  f1           JA         LAB_fffff9e8
        fffff9f7 66  8b  d9       MOV        BX,CX
        fffff9fa c1  c9  10       ROR        ECX ,0x10
        fffff9fd 66  d3  e3       SHL        BX,CL
        fffffa00 c1  c9  10       ROR        ECX ,0x10
        fffffa03 66  8b  cb       MOV        CX,BX
        fffffa06 0a  d2           OR         DL,DL
        fffffa08 75  28           JNZ        LAB_fffffa32
        fffffa0a c1  c9  10       ROR        ECX ,0x10
        fffffa0d 66  0f  b6  de    MOVZX      BX,DH
        fffffa11 66  d3  e3       SHL        BX,CL
        fffffa14 0a  ff           OR         BH,BH
        fffffa16 74  06           JZ         LAB_fffffa1e
        fffffa18 66  b8  14  00    MOV        AX,0x14
        fffffa1c eb  69           JMP        LAB_fffffa87
                             LAB_fffffa1e                                    XREF[1]:     fffffa16 (j)   
        fffffa1e d2  e6           SHL        DH,CL
        fffffa20 c1  c9  10       ROR        ECX ,0x10
        fffffa23 66  f7  d1       NOT        CX
        fffffa26 84  f1           TEST       CL,DH
        fffffa28 74  06           JZ         LAB_fffffa30
        fffffa2a 66  b8  14  00    MOV        AX,0x14
        fffffa2e eb  57           JMP        LAB_fffffa87
                             LAB_fffffa30                                    XREF[1]:     fffffa28 (j)   
        fffffa30 8a  de           MOV        BL,DH
                             LAB_fffffa32                                    XREF[1]:     fffffa08 (j)   
        fffffa32 66  ba  71  00    MOV        DX,0x71
        fffffa36 c1  ca  10       ROR        EDX ,0x10
        fffffa39 66  ba  70  00    MOV        DX,0x70
        fffffa3d 0f  6e  c2       MOVD       MM0 ,EDX
        fffffa40 0c  80           OR         AL,0x80
        fffffa42 ee              OUT        DX,AL
        fffffa43 eb  00           JMP        LAB_fffffa45
                             LAB_fffffa45                                    XREF[1]:     fffffa43 (j)   
        fffffa45 eb  00           JMP        LAB_fffffa47
                             LAB_fffffa47                                    XREF[1]:     fffffa45 (j)   
        fffffa47 eb  00           JMP        LAB_fffffa49
                             LAB_fffffa49                                    XREF[1]:     fffffa47 (j)   
        fffffa49 eb  00           JMP        LAB_fffffa4b
                             LAB_fffffa4b                                    XREF[1]:     fffffa49 (j)   
        fffffa4b 86  e0           XCHG       AL,AH
        fffffa4d c1  ca  10       ROR        EDX ,0x10
        fffffa50 ec              IN         AL,DX
        fffffa51 0f  7e  da       MOVD       EDX ,MM3
        fffffa54 0a  d2           OR         DL,DL
        fffffa56 75  1d           JNZ        LAB_fffffa75
        fffffa58 0f  7e  c2       MOVD       EDX ,MM0
        fffffa5b 86  e0           XCHG       AL,AH
        fffffa5d ee              OUT        DX,AL
        fffffa5e eb  00           JMP        LAB_fffffa60
                             LAB_fffffa60                                    XREF[1]:     fffffa5e (j)   
        fffffa60 eb  00           JMP        LAB_fffffa62
                             LAB_fffffa62                                    XREF[1]:     fffffa60 (j)   
        fffffa62 eb  00           JMP        LAB_fffffa64
                             LAB_fffffa64                                    XREF[1]:     fffffa62 (j)   
        fffffa64 eb  00           JMP        LAB_fffffa66
                             LAB_fffffa66                                    XREF[1]:     fffffa64 (j)   
        fffffa66 86  c4           XCHG       AH,AL
        fffffa68 22  c1           AND        AL,CL
        fffffa6a 0a  c3           OR         AL,BL
        fffffa6c c1  ca  10       ROR        EDX ,0x10
        fffffa6f ee              OUT        DX,AL
        fffffa70 f8              CLC
        fffffa71 eb  15           JMP        LAB_fffffa88
        fffffa73 eb              ??         EBh
        fffffa74 0e              ??         0Eh
                             LAB_fffffa75                                    XREF[1]:     fffffa56 (j)   
        fffffa75 22  c1           AND        AL,CL
        fffffa77 c1  c9  10       ROR        ECX ,0x10
        fffffa7a d2  e8           SHR        AL,CL
        fffffa7c 66  0f  b6  c0    MOVZX      AX,AL
        fffffa80 f8              CLC
        fffffa81 eb  05           JMP        LAB_fffffa88
                             LAB_fffffa83                                    XREF[2]:     fffff9a0 (j) , fffff9aa (j)   
        fffffa83 66  b8  13  00    MOV        AX,0x13
                             LAB_fffffa87                                    XREF[3]:     fffff9d9 (j) , fffffa1c (j) , 
                                                                                          fffffa2e (j)   
        fffffa87 f9              STC
                             LAB_fffffa88                                    XREF[2]:     fffffa71 (j) , fffffa81 (j)   
        fffffa88 0f  7e  cb       MOVD       EBX ,MM1
        fffffa8b 0f  7e  d1       MOVD       ECX ,MM2
        fffffa8e 0f  7e  da       MOVD       EDX ,MM3
        fffffa91 ff  e7           JMP        EDI
                             LAB_fffffa93                                    XREF[1]:     ffffff7b (j)   
        fffffa93 e9  e8  04       JMP        LAB_ffffff80
                 00  00
        fffffa98 cc              ??         CCh
        fffffa99 cc              ??         CCh
        fffffa9a cc              ??         CCh
        fffffa9b cc              ??         CCh
                             LAB_fffffa9c                                    XREF[1]:     fffffd23 (j)   
        fffffa9c b0  02           MOV        AL,0x2
        fffffa9e e6  80           OUT        0x80 ,AL
        fffffaa0 be  aa  fa       MOV        ESI ,0xfffffaaa
                 ff  ff
        fffffaa5 0f  6e  fe       MOVD       MM7 ,ESI
        fffffaa8 eb  53           JMP        LAB_fffffafd
        fffffaaa b0  07           MOV        AL,0x7
        fffffaac e6  80           OUT        0x80 ,AL
        fffffaae be  b8  fa       MOV        ESI ,LAB_fffffab8
                 ff  ff
        fffffab3 0f  6e  fe       MOVD       MM7 ,ESI
        fffffab6 eb  2b           JMP        LAB_fffffae3
                             LAB_fffffab8                                    XREF[2]:     fffffaae (*) , fffffafb (j)   
        fffffab8 be  c5  fa       MOV        ESI ,LAB_fffffac5
                 ff  ff
        fffffabd 0f  6e  fe       MOVD       MM7 ,ESI
        fffffac0 e9  8d  00       JMP        LAB_fffffb52
                 00  00
                             LAB_fffffac5                                    XREF[2]:     fffffab8 (*) , fffffb5d (j)   
        fffffac5 b0  03           MOV        AL,0x3
        fffffac7 e6  80           OUT        0x80 ,AL
        fffffac9 b0  09           MOV        AL,0x9
        fffffacb e6  80           OUT        0x80 ,AL
        fffffacd be  da  fa       MOV        ESI ,LAB_fffffada
                 ff  ff
        fffffad2 0f  6e  fe       MOVD       MM7 ,ESI
        fffffad5 e9  85  00       JMP        LAB_fffffb5f
                 00  00
                             LAB_fffffada                                    XREF[2]:     fffffacd (*) , fffffce1 (j)   
        fffffada b0  0b           MOV        AL,0xb
        fffffadc e6  80           OUT        0x80 ,AL
        fffffade e9  45  02       JMP        LAB_fffffd28
                 00  00
                             LAB_fffffae3                                    XREF[1]:     fffffab6 (j)   
        fffffae3 e9  40  fc       JMP        LAB_fffff728
                 ff  ff
                             LAB_fffffae8                                    XREF[1]:     fffff82b (j)   
        fffffae8 0b  c0           OR         EAX ,EAX
        fffffaea 74  0c           JZ         LAB_fffffaf8
        fffffaec b9  79  00       MOV        ECX ,0x79
                 00  00
        fffffaf1 33  d2           XOR        EDX ,EDX
        fffffaf3 83  c0  30       ADD        EAX ,0x30
        fffffaf6 0f  30           WRMSR
                             LAB_fffffaf8                                    XREF[1]:     fffffaea (j)   
        fffffaf8 0f  7e  fe       MOVD       ESI ,MM7
        fffffafb ff  e6           JMP        ESI =>LAB_fffffab8
                             LAB_fffffafd                                    XREF[1]:     fffffaa8 (j)   
        fffffafd b8  0b  00       MOV        EAX ,0xb
                 00  00
        fffffb02 b9  01  00       MOV        ECX ,0x1
                 00  00
        fffffb07 0f  a2           CPUID
        fffffb09 8b  f0           MOV        ESI ,EAX
        fffffb0b b8  01  00       MOV        EAX ,0x1
                 00  00
        fffffb10 0f  a2           CPUID
        fffffb12 0f  cb           BSWAP      EBX
        fffffb14 0f  b6  c3       MOVZX      EAX ,BL
        fffffb17 0f  b6  db       MOVZX      EBX ,BL
        fffffb1a c1  e0  18       SHL        EAX ,0x18
        fffffb1d 66  8b  ce       MOV        CX,SI
        fffffb20 d2  eb           SHR        BL,CL
        fffffb22 0b  c3           OR         EAX ,EBX
        fffffb24 0f  6e  c8       MOVD       MM1 ,EAX
        fffffb27 b8  60  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffb2c 66  ba  f8  0c    MOV        DX,0xcf8
        fffffb30 ef              OUT        DX,EAX
        fffffb31 66  ba  fc  0c    MOV        DX,0xcfc
        fffffb35 b8  04  00       MOV        EAX ,0x4
                 00  00
        fffffb3a ef              OUT        DX,EAX
        fffffb3b ed              IN         EAX ,DX
        fffffb3c 0d  01  00       OR         EAX ,0xf8000001
                 00  f8
        fffffb41 ef              OUT        DX,EAX
        fffffb42 0f  7e  c8       MOVD       EAX ,MM1
        fffffb45 25  ff  ff       AND        EAX ,0xfff3ffff
                 f3  ff
        fffffb4a 0f  6e  c8       MOVD       MM1 ,EAX
        fffffb4d 0f  7e  fe       MOVD       ESI ,MM7
        fffffb50 ff  e6           JMP        ESI
                             LAB_fffffb52                                    XREF[1]:     fffffac0 (j)   
        fffffb52 0f  7e  c8       MOVD       EAX ,MM1
        fffffb55 b4  01           MOV        AH,0x1
        fffffb57 0f  6e  c8       MOVD       MM1 ,EAX
        fffffb5a 0f  7e  fe       MOVD       ESI ,MM7
        fffffb5d ff  e6           JMP        ESI =>LAB_fffffac5
                             LAB_fffffb5f                                    XREF[1]:     fffffad5 (j)   
        fffffb5f bf  00  03       MOV        EDI ,0xfee00300
                 e0  fe
        fffffb64 b8  00  45       MOV        EAX ,0xc4500
                 0c  00
        fffffb69 89  07           MOV        dword ptr [EDI ]=>DAT_fee00300 ,EAX
                             LAB_fffffb6b                                    XREF[1]:     fffffb71 (j)   
        fffffb6b 8b  07           MOV        EAX ,dword ptr [EDI ]=>DAT_fee00300
        fffffb6d 0f  ba  e0  0c    BT         EAX ,0xc
        fffffb71 72  f8           JC         LAB_fffffb6b
        fffffb73 b9  fe  00       MOV        ECX ,0xfe
                 00  00
        fffffb78 0f  32           RDMSR
        fffffb7a 0f  b6  d8       MOVZX      EBX ,AL
        fffffb7d d1  e3           SHL        EBX ,1
        fffffb7f 33  c0           XOR        EAX ,EAX
        fffffb81 33  d2           XOR        EDX ,EDX
                             LAB_fffffb83                                    XREF[1]:     fffffb90 (j)   
        fffffb83 83  c3  fe       ADD        EBX ,-0x2
        fffffb86 2e  0f  b7       MOVZX      ECX ,word ptr CS:[EBX  + 0xfffffce3 ]=>LAB_fffffc
                 8b  e3  fc 
                 ff  ff
        fffffb8e 0f  30           WRMSR
        fffffb90 75  f1           JNZ        LAB_fffffb83
        fffffb92 b9  ff  02       MOV        ECX ,0x2ff
                 00  00
        fffffb97 0f  32           RDMSR
        fffffb99 25  00  f3       AND        EAX ,0xfffff300
                 ff  ff
        fffffb9e 0f  30           WRMSR
        fffffba0 33  ff           XOR        EDI ,EDI
        fffffba2 33  c9           XOR        ECX ,ECX
        fffffba4 0f  6e  d1       MOVD       MM2 ,ECX
                             LAB_fffffba7                                    XREF[1]:     fffffbf8 (j)   
        fffffba7 0f  7e  d1       MOVD       ECX ,MM2
        fffffbaa b8  04  00       MOV        EAX ,0x4
                 00  00
        fffffbaf 0f  a2           CPUID
        fffffbb1 8b  f0           MOV        ESI ,EAX
        fffffbb3 24  1f           AND        AL,0x1f
        fffffbb5 0a  c0           OR         AL,AL
        fffffbb7 74  41           JZ         LAB_fffffbfa
        fffffbb9 66  8b  c6       MOV        AX,SI
        fffffbbc 24  e0           AND        AL,0xe0
        fffffbbe 3c  60           CMP        AL,0x60
        fffffbc0 75  2f           JNZ        LAB_fffffbf1
        fffffbc2 8b  f3           MOV        ESI ,EBX
        fffffbc4 8b  c3           MOV        EAX ,EBX
        fffffbc6 c1  e8  0b       SHR        EAX ,0xb
        fffffbc9 25  ff  03       AND        EAX ,0x3ff
                 00  00
        fffffbce 40              INC        EAX
        fffffbcf 81  e3  ff       AND        EBX ,0xfff
                 0f  00  00
        fffffbd5 43              INC        EBX
        fffffbd6 f7  e3           MUL        EBX
        fffffbd8 8b  de           MOV        EBX ,ESI
        fffffbda c1  eb  16       SHR        EBX ,0x16
        fffffbdd 81  e3  ff       AND        EBX ,0x3ff
                 03  00  00
        fffffbe3 43              INC        EBX
        fffffbe4 f7  e3           MUL        EBX
        fffffbe6 8b  d9           MOV        EBX ,ECX
        fffffbe8 43              INC        EBX
        fffffbe9 f7  e3           MUL        EBX
        fffffbeb 8b  df           MOV        EBX ,EDI
        fffffbed 03  c3           ADD        EAX ,EBX
        fffffbef 8b  f8           MOV        EDI ,EAX
                             LAB_fffffbf1                                    XREF[1]:     fffffbc0 (j)   
        fffffbf1 0f  7e  d0       MOVD       EAX ,MM2
        fffffbf4 40              INC        EAX
        fffffbf5 0f  6e  d0       MOVD       MM2 ,EAX
        fffffbf8 eb  ad           JMP        LAB_fffffba7
                             LAB_fffffbfa                                    XREF[1]:     fffffbb7 (j)   
        fffffbfa b8  08  00       MOV        EAX ,0x80000008
                 00  80
        fffffbff 0f  a2           CPUID
        fffffc01 2c  20           SUB        AL,0x20
        fffffc03 0f  b6  c0       MOVZX      EAX ,AL
        fffffc06 33  f6           XOR        ESI ,ESI
        fffffc08 0f  ab  c6       BTS        ESI ,EAX
        fffffc0b 4e              DEC        ESI
        fffffc0c b8  06  00       MOV        EAX ,0xff900006
                 90  ff
        fffffc11 33  d2           XOR        EDX ,EDX
        fffffc13 b9  00  02       MOV        ECX ,0x200
                 00  00
        fffffc18 0f  30           WRMSR
        fffffc1a b8  00  08       MOV        EAX ,0xfffe0800
                 fe  ff
        fffffc1f 8b  d6           MOV        EDX ,ESI
        fffffc21 b9  01  02       MOV        ECX ,0x201
                 00  00
        fffffc26 0f  30           WRMSR
        fffffc28 8b  df           MOV        EBX ,EDI
        fffffc2a 81  fb  00       CMP        EBX ,0x180000
                 00  18  00
        fffffc30 73  1e           JNC        LAB_fffffc50
        fffffc32 b8  05  00       MOV        EAX ,0xfff00005
                 f0  ff
        fffffc37 33  d2           XOR        EDX ,EDX
        fffffc39 b9  02  02       MOV        ECX ,0x202
                 00  00
        fffffc3e 0f  30           WRMSR
        fffffc40 b8  00  08       MOV        EAX ,0xfffc0800
                 fc  ff
        fffffc45 8b  d6           MOV        EDX ,ESI
        fffffc47 b9  03  02       MOV        ECX ,0x203
                 00  00
        fffffc4c 0f  30           WRMSR
        fffffc4e eb  1c           JMP        LAB_fffffc6c
                             LAB_fffffc50                                    XREF[1]:     fffffc30 (j)   
        fffffc50 b8  05  00       MOV        EAX ,0xfff00005
                 f0  ff
        fffffc55 33  d2           XOR        EDX ,EDX
        fffffc57 b9  02  02       MOV        ECX ,0x202
                 00  00
        fffffc5c 0f  30           WRMSR
        fffffc5e b8  00  08       MOV        EAX ,0xfff00800
                 f0  ff
        fffffc63 8b  d6           MOV        EDX ,ESI
        fffffc65 b9  03  02       MOV        ECX ,0x203
                 00  00
        fffffc6a 0f  30           WRMSR
                             LAB_fffffc6c                                    XREF[1]:     fffffc4e (j)   
        fffffc6c b9  ff  02       MOV        ECX ,0x2ff
                 00  00
        fffffc71 0f  32           RDMSR
        fffffc73 0d  00  08       OR         EAX ,0x800
                 00  00
        fffffc78 0f  30           WRMSR
        fffffc7a 0f  20  c0       MOV        EAX ,CR0
        fffffc7d 25  ff  ff       AND        EAX ,0x9fffffff
                 ff  9f
        fffffc82 0f  08           INVD
        fffffc84 0f  22  c0       MOV        CR0 ,EAX
        fffffc87 b9  e0  02       MOV        ECX ,0x2e0
                 00  00
        fffffc8c 0f  32           RDMSR
        fffffc8e 83  c8  01       OR         EAX ,0x1
        fffffc91 0f  30           WRMSR
        fffffc93 bf  00  00       MOV        EDI ,0xff900000
                 90  ff
        fffffc98 b9  00  08       MOV        ECX ,0x800
                 00  00
        fffffc9d b8  a5  a5       MOV        EAX ,0xa5a5a5a5
                 a5  a5
                             LAB_fffffca2                                    XREF[1]:     fffffcaa (j)   
        fffffca2 89  07           MOV        dword ptr [EDI ]=>DAT_ff900000 ,EAX
        fffffca4 0f  ae  f8       SFENCE
        fffffca7 83  c7  40       ADD        EDI ,0x40
        fffffcaa e2  f6           LOOP       LAB_fffffca2
        fffffcac b9  e0  02       MOV        ECX ,0x2e0
                 00  00
        fffffcb1 0f  32           RDMSR
        fffffcb3 83  c8  02       OR         EAX ,0x2
        fffffcb6 0f  30           WRMSR
        fffffcb8 fc              CLD
        fffffcb9 bf  00  00       MOV        EDI ,0xff900000
                 90  ff
        fffffcbe b9  00  80       MOV        ECX ,0x8000
                 00  00
        fffffcc3 b8  5a  5a       MOV        EAX ,0x5a5a5a5a
                 5a  5a
                             LAB_fffffcc8                                    XREF[1]:     fffffcce (j)   
        fffffcc8 ab              STOSD      ES:EDI =>DAT_ff900000
        fffffcc9 3b  47  fc       CMP        EAX ,dword ptr [EDI  + -0x4 ]=>DAT_ff900000
        fffffccc 75  04           JNZ        LAB_fffffcd2
        fffffcce e2  f8           LOOP       LAB_fffffcc8
        fffffcd0 eb  0c           JMP        LAB_fffffcde
                             LAB_fffffcd2                                    XREF[1]:     fffffccc (j)   
        fffffcd2 b0  d0           MOV        AL,0xd0
        fffffcd4 e6  80           OUT        0x80 ,AL
                             LAB_fffffcd6                                    XREF[1]:     fffffcd6 (j)   
        fffffcd6 eb  fe           JMP        LAB_fffffcd6
        fffffcd8 b0  d1           MOV        AL,0xd1
        fffffcda e6  80           OUT        0x80 ,AL
                             LAB_fffffcdc                                    XREF[1]:     fffffcdc (j)   
        fffffcdc eb  fe           JMP        LAB_fffffcdc
                             LAB_fffffcde+1                                  XREF[1,1]:   fffffcd0 (j) , fffffb86 (R)   
                             LAB_fffffcde
        fffffcde 0f  7e  fe       MOVD       ESI ,MM7
                             LAB_fffffce1                                    XREF[1]:     fffffb86 (R)   
        fffffce1 ff  e6           JMP        ESI =>LAB_fffffada
        fffffce3 ff              ??         FFh
        fffffce4 02  50  02       ADD        DL,byte ptr [EAX  + 0x2 ]
        fffffce7 58              POP        EAX
        fffffce8 02  59  02       ADD        BL,byte ptr [ECX  + 0x2 ]
        fffffceb 68  02  69       PUSH       0x6a026902
                 02  6a
        fffffcf0 02  6b  02       ADD        CH,byte ptr [EBX  + 0x2 ]
        fffffcf3 6c              INSB       ES:EDI ,DX
        fffffcf4 02  6d  02       ADD        CH,byte ptr [EBP  + 0x2 ]
        fffffcf7 6e              OUTSB      DX,ESI
        fffffcf8 02  6f  02       ADD        CH,byte ptr [EDI  + 0x2 ]
        fffffcfb 00  02           ADD        byte ptr [EDX ],AL
        fffffcfd 01  02           ADD        dword ptr [EDX ],EAX
        fffffcff 02  02           ADD        AL,byte ptr [EDX ]
        fffffd01 03  02           ADD        EAX ,dword ptr [EDX ]
        fffffd03 04  02           ADD        AL,0x2
        fffffd05 05  02  06       ADD        EAX ,0x7020602
                 02  07
        fffffd0a 02  08           ADD        CL,byte ptr [EAX ]
        fffffd0c 02  09           ADD        CL,byte ptr [ECX ]
        fffffd0e 02  0a           ADD        CL,byte ptr [EDX ]
        fffffd10 02  0b           ADD        CL,byte ptr [EBX ]
        fffffd12 02  0c  02       ADD        CL,byte ptr [EDX  + EAX *0x1 ]
        fffffd15 0d  02  0e       OR         EAX ,0xf020e02
                 02  0f
        fffffd1a 02  10           ADD        DL,byte ptr [EAX ]
        fffffd1c 02  11           ADD        DL,byte ptr [ECX ]
        fffffd1e 02  12           ADD        DL,byte ptr [EDX ]
        fffffd20 02  13           ADD        DL,byte ptr [EBX ]
        fffffd22 02              ??         02h
                             LAB_fffffd23                                    XREF[1]:     ffffff76 (j)   
        fffffd23 e9  74  fd       JMP        LAB_fffffa9c
                 ff  ff
                             LAB_fffffd28                                    XREF[1]:     fffffade (j)   
        fffffd28 bc  00  00       MOV        ESP ,0xff920000
                 92  ff
        fffffd2d e9  49  02       JMP        LAB_ffffff7b
                 00  00
        fffffd32 cc              ??         CCh
        fffffd33 cc              ??         CCh
        fffffd34 cc              ??         CCh
        fffffd35 cc              ??         CCh
        fffffd36 cc              ??         CCh
        fffffd37 cc              ??         CCh
        fffffd38 cc              ??         CCh
        fffffd39 cc              ??         CCh
        fffffd3a cc              ??         CCh
        fffffd3b cc              ??         CCh
                             LAB_fffffd3c                                    XREF[1]:     ffffff6c (j)   
        fffffd3c b0  04           MOV        AL,0x4
        fffffd3e e6  80           OUT        0x80 ,AL
        fffffd40 b0  ff           MOV        AL,0xff
        fffffd42 e6  21           OUT        0x21 ,AL
        fffffd44 e6  ed           OUT        0xed ,AL
        fffffd46 e6  a1           OUT        0xa1 ,AL
        fffffd48 e6  ed           OUT        0xed ,AL
        fffffd4a b0  00           MOV        AL,0x0
        fffffd4c e6  92           OUT        0x92 ,AL
        fffffd4e 66  ba  f8  0c    MOV        DX,0xcf8
        fffffd52 b8  f0  f8       MOV        EAX ,0x8000f8f0
                 00  80
        fffffd57 ef              OUT        DX,EAX
        fffffd58 b2  fc           MOV        DL,0xfc
        fffffd5a b8  01  c0       MOV        EAX ,0xfed1c001
                 d1  fe
        fffffd5f ef              OUT        DX,EAX
        fffffd60 b2  f8           MOV        DL,0xf8
        fffffd62 b8  dc  f8       MOV        EAX ,0x8000f8dc
                 00  80
        fffffd67 ef              OUT        DX,EAX
        fffffd68 b2  fc           MOV        DL,0xfc
        fffffd6a ed              IN         EAX ,DX
        fffffd6b 83  c8  08       OR         EAX ,0x8
        fffffd6e ef              OUT        DX,EAX
        fffffd6f b2  f8           MOV        DL,0xf8
        fffffd71 b8  40  f8       MOV        EAX ,0x8000f840
                 00  80
        fffffd76 ef              OUT        DX,EAX
        fffffd77 b2  fc           MOV        DL,0xfc
        fffffd79 66  b8  00  04    MOV        AX,0x400
        fffffd7d 66  ef           OUT        DX,AX
        fffffd7f b2  f8           MOV        DL,0xf8
        fffffd81 b8  44  f8       MOV        EAX ,0x8000f844
                 00  80
        fffffd86 ef              OUT        DX,EAX
        fffffd87 b2  fc           MOV        DL,0xfc
        fffffd89 b0  80           MOV        AL,0x80
        fffffd8b ee              OUT        DX,AL
        fffffd8c b2  f8           MOV        DL,0xf8
        fffffd8e b8  48  f8       MOV        EAX ,0x8000f848
                 00  80
        fffffd93 ef              OUT        DX,EAX
        fffffd94 b2  fc           MOV        DL,0xfc
        fffffd96 66  b8  00  05    MOV        AX,0x500
        fffffd9a 66  ef           OUT        DX,AX
        fffffd9c b2  f8           MOV        DL,0xf8
        fffffd9e b8  4c  f8       MOV        EAX ,0x8000f84c
                 00  80
        fffffda3 ef              OUT        DX,EAX
        fffffda4 b2  fc           MOV        DL,0xfc
        fffffda6 b0  10           MOV        AL,0x10
        fffffda8 ee              OUT        DX,AL
        fffffda9 b2  f8           MOV        DL,0xf8
        fffffdab b8  d8  f8       MOV        EAX ,0x8000f8d8
                 00  80
        fffffdb0 ef              OUT        DX,EAX
        fffffdb1 b2  fd           MOV        DL,0xfd
        fffffdb3 b0  ff           MOV        AL,0xff
        fffffdb5 ee              OUT        DX,AL
        fffffdb6 66  ba  68  04    MOV        DX,0x468
        fffffdba 66  ed           IN         AX,DX
        fffffdbc e6  ed           OUT        0xed ,AL
        fffffdbe 80  cc  08       OR         AH,0x8
        fffffdc1 66  ef           OUT        DX,AX
        fffffdc3 e6  ed           OUT        0xed ,AL
        fffffdc5 66  ba  6a  04    MOV        DX,0x46a
        fffffdc9 66  ed           IN         AX,DX
        fffffdcb e6  ed           OUT        0xed ,AL
        fffffdcd 24  f9           AND        AL,0xf9
        fffffdcf 66  ef           OUT        DX,AX
        fffffdd1 e6  ed           OUT        0xed ,AL
        fffffdd3 66  ba  66  04    MOV        DX,0x466
        fffffdd7 66  ed           IN         AX,DX
        fffffdd9 e6  ed           OUT        0xed ,AL
        fffffddb 24  fe           AND        AL,0xfe
        fffffddd 0c  02           OR         AL,0x2
        fffffddf 24  fe           AND        AL,0xfe
        fffffde1 66  ef           OUT        DX,AX
        fffffde3 e6  ed           OUT        0xed ,AL
        fffffde5 be  00  f4       MOV        ESI ,0xfed1f400
                 d1  fe
        fffffdea c6  06  04       MOV        byte ptr [ESI ]=>DAT_7ed1e8fc ,0x4
        fffffded be  10  f4       MOV        ESI ,0xfed1f410
                 d1  fe
        fffffdf2 c6  06  60       MOV        byte ptr [ESI ]=>DAT_7ed1e90c ,0x60
        fffffdf5 66  ba  f8  0c    MOV        DX,0xcf8
        fffffdf9 b8  e4  f8       MOV        EAX ,0x8000f8e4
                 00  80
        fffffdfe ef              OUT        DX,EAX
        fffffdff b2  fc           MOV        DL,0xfc
        fffffe01 b8  00  00       MOV        EAX ,0x0
                 00  00
        fffffe06 ef              OUT        DX,EAX
        fffffe07 66  ba  00  05    MOV        DX,0x500
        fffffe0b ec              IN         AL,DX
        fffffe0c 0c  02           OR         AL,0x2
        fffffe0e ee              OUT        DX,AL
        fffffe0f 66  ba  04  05    MOV        DX,0x504
        fffffe13 ec              IN         AL,DX
        fffffe14 0c  02           OR         AL,0x2
        fffffe16 ee              OUT        DX,AL
        fffffe17 66  ba  0c  05    MOV        DX,0x50c
        fffffe1b ec              IN         AL,DX
        fffffe1c a8  02           TEST       AL,0x2
        fffffe1e e6  80           OUT        0x80 ,AL
        fffffe20 75  12           JNZ        LAB_fffffe34
        fffffe22 66  ba  f8  0c    MOV        DX,0xcf8
        fffffe26 b8  e8  f8       MOV        EAX ,0x8000f8e8
                 00  80
        fffffe2b ef              OUT        DX,EAX
        fffffe2c b2  fc           MOV        DL,0xfc
        fffffe2e ed              IN         EAX ,DX
        fffffe2f 83  e0  02       AND        EAX ,0x2
        fffffe32 74  08           JZ         LAB_fffffe3c
                             LAB_fffffe34                                    XREF[1]:     fffffe20 (j)   
        fffffe34 bf  18  f4       MOV        EDI ,0xfed1f418
                 d1  fe
        fffffe39 83  0f  02       OR         dword ptr [EDI ]=>DAT_7ed1e914 ,0x2
                             LAB_fffffe3c                                    XREF[1]:     fffffe32 (j)   
        fffffe3c e9  30  01       JMP        LAB_ffffff71
                 00  00
        fffffe41 cc              ??         CCh
        fffffe42 cc              ??         CCh
        fffffe43 cc              ??         CCh
        fffffe44 cc              ??         CCh
        fffffe45 cc              ??         CCh
        fffffe46 cc              ??         CCh
        fffffe47 cc              ??         CCh
        fffffe48 cc              ??         CCh
        fffffe49 cc              ??         CCh
        fffffe4a cc              ??         CCh
        fffffe4b cc              ??         CCh
                             LAB_fffffe4c                                    XREF[1]:     ffffff71 (j)   
        fffffe4c b0  03           MOV        AL,0x3
        fffffe4e e6  80           OUT        0x80 ,AL
        fffffe50 b8  60  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffe55 66  ba  f8  0c    MOV        DX,0xcf8
        fffffe59 ef              OUT        DX,EAX
        fffffe5a b2  fc           MOV        DL,0xfc
        fffffe5c ec              IN         AL,DX
        fffffe5d 24  01           AND        AL,0x1
        fffffe5f 74  09           JZ         LAB_fffffe6a
        fffffe61 b0  06           MOV        AL,0x6
        fffffe63 66  ba  f9  0c    MOV        DX,0xcf9
        fffffe67 ee              OUT        DX,AL
                             LAB_fffffe68                                    XREF[1]:     fffffe68 (j)   
        fffffe68 eb  fe           JMP        LAB_fffffe68
                             LAB_fffffe6a                                    XREF[1]:     fffffe5f (j)   
        fffffe6a b8  60  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffe6f 66  ba  f8  0c    MOV        DX,0xcf8
        fffffe73 ef              OUT        DX,EAX
        fffffe74 b2  fc           MOV        DL,0xfc
        fffffe76 b8  04  00       MOV        EAX ,0x4
                 00  00
        fffffe7b ef              OUT        DX,EAX
        fffffe7c ed              IN         EAX ,DX
        fffffe7d 0d  01  00       OR         EAX ,0xf8000001
                 00  f8
        fffffe82 ef              OUT        DX,EAX
        fffffe83 b8  40  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffe88 66  ba  f8  0c    MOV        DX,0xcf8
        fffffe8c ef              OUT        DX,EAX
        fffffe8d b2  fc           MOV        DL,0xfc
        fffffe8f b8  01  90       MOV        EAX ,0xfed19001
                 d1  fe
        fffffe94 ef              OUT        DX,EAX
        fffffe95 b8  48  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffe9a 66  ba  f8  0c    MOV        DX,0xcf8
        fffffe9e ef              OUT        DX,EAX
        fffffe9f b2  fc           MOV        DL,0xfc
        fffffea1 b8  01  00       MOV        EAX ,0xfed10001
                 d1  fe
        fffffea6 ef              OUT        DX,EAX
        fffffea7 b8  68  00       MOV        EAX ,offset  IMAGE_DOS_HEADER_fffff4fc.e_program  = null
                 00  80
        fffffeac 66  ba  f8  0c    MOV        DX,0xcf8
        fffffeb0 ef              OUT        DX,EAX
        fffffeb1 b2  fc           MOV        DL,0xfc
        fffffeb3 b8  01  80       MOV        EAX ,0xfed18001
                 d1  fe
        fffffeb8 ef              OUT        DX,EAX
        fffffeb9 e9  b8  00       JMP        LAB_ffffff76
                 00  00
        fffffebe cc              ??         CCh
        fffffebf cc              ??         CCh
        fffffec0 cc              ??         CCh
        fffffec1 cc              ??         CCh
        fffffec2 cc              ??         CCh
        fffffec3 cc              ??         CCh
        fffffec4 cc              ??         CCh
        fffffec5 cc              ??         CCh
        fffffec6 cc              ??         CCh
        fffffec7 cc              ??         CCh
        fffffec8 cc              ??         CCh
        fffffec9 cc              ??         CCh
        fffffeca cc              ??         CCh
        fffffecb cc              ??         CCh
                             LAB_fffffecc                                    XREF[1]:     ffffff80 (j)   
        fffffecc e9  b4  00       JMP        LAB_ffffff85
                 00  00
        fffffed1 cc              ??         CCh
        fffffed2 cc              ??         CCh
        fffffed3 cc              ??         CCh
        fffffed4 cc              ??         CCh
        fffffed5 cc              ??         CCh
        fffffed6 cc              ??         CCh
        fffffed7 cc              ??         CCh
        fffffed8 cc              ??         CCh
        fffffed9 cc              ??         CCh
        fffffeda cc              ??         CCh
        fffffedb cc              ??         CCh
                             LAB_fffffedc                                    XREF[1]:     ffffff85 (j)   
        fffffedc 66  ba  f8  0c    MOV        DX,0xcf8
        fffffee0 b8  80  f8       MOV        EAX ,0x8000f880
                 00  80
        fffffee5 ef              OUT        DX,EAX
        fffffee6 66  ba  fc  0c    MOV        DX,0xcfc
        fffffeea ed              IN         EAX ,DX
        fffffeeb 0d  00  00       OR         EAX ,0x24000000
                 00  24
        fffffef0 0d  00  00       OR         EAX ,0x8000000
                 00  08
        fffffef5 ef              OUT        DX,EAX
        fffffef6 66  ba  f8  0c    MOV        DX,0xcf8
        fffffefa b8  84  f8       MOV        EAX ,0x8000f884
                 00  80
        fffffeff ef              OUT        DX,EAX
        ffffff00 b2  fc           MOV        DL,0xfc
        ffffff02 b8  82  06       MOV        EAX ,0x7c0682
                 7c  00
        ffffff07 ef              OUT        DX,EAX
        ffffff08 66  ba  f8  0c    MOV        DX,0xcf8
        ffffff0c b8  88  f8       MOV        EAX ,0x8000f888
                 00  80
        ffffff11 ef              OUT        DX,EAX
        ffffff12 b2  fc           MOV        DL,0xfc
        ffffff14 b8  01  09       MOV        EAX ,0x7c0901
                 7c  00
        ffffff19 ef              OUT        DX,EAX
        ffffff1a 66  ba  f8  0c    MOV        DX,0xcf8
        ffffff1e b8  8c  f8       MOV        EAX ,0x8000f88c
                 00  80
        ffffff23 ef              OUT        DX,EAX
        ffffff24 b2  fc           MOV        DL,0xfc
        ffffff26 b8  e1  07       MOV        EAX ,0x3c07e1
                 3c  00
        ffffff2b ef              OUT        DX,EAX
        ffffff2c 66  ba  f8  0c    MOV        DX,0xcf8
        ffffff30 b8  90  f8       MOV        EAX ,0x8000f890
                 00  80
        ffffff35 ef              OUT        DX,EAX
        ffffff36 b2  fc           MOV        DL,0xfc
        ffffff38 b8  01  09       MOV        EAX ,0x1c0901
                 1c  00
        ffffff3d ef              OUT        DX,EAX
        ffffff3e 66  ba  f8  0c    MOV        DX,0xcf8
        ffffff42 b8  a4  f8       MOV        EAX ,0x8000f8a4
                 00  80
        ffffff47 ef              OUT        DX,EAX
        ffffff48 66  ba  fc  0c    MOV        DX,0xcfc
        ffffff4c 66  ed           IN         AX,DX
        ffffff4e 66  a9  04  00    TEST       AX,0x4
        ffffff52 74  0e           JZ         LAB_ffffff62
        ffffff54 66  ba  70  00    MOV        DX,0x70
        ffffff58 b0  6f           MOV        AL,0x6f
        ffffff5a 0c  80           OR         AL,0x80
        ffffff5c ee              OUT        DX,AL
        ffffff5d 66  42           INC        DX
        ffffff5f 32  c0           XOR        AL,AL
        ffffff61 ee              OUT        DX,AL
                             LAB_ffffff62                                    XREF[1]:     ffffff52 (j)   
        ffffff62 e9  23  00       JMP        LAB_ffffff8a
                 00  00
        ffffff67 cc              ??         CCh
        ffffff68 cc              ??         CCh
        ffffff69 cc              ??         CCh
        ffffff6a cc              ??         CCh
        ffffff6b cc              ??         CCh
                             LAB_ffffff6c                                    XREF[1]:     fffff8a0 (j)   
        ffffff6c e9  cb  fd       JMP        LAB_fffffd3c
                 ff  ff
                             LAB_ffffff71                                    XREF[1]:     fffffe3c (j)   
        ffffff71 e9  d6  fe       JMP        LAB_fffffe4c
                 ff  ff
                             LAB_ffffff76                                    XREF[1]:     fffffeb9 (j)   
        ffffff76 e9  a8  fd       JMP        LAB_fffffd23
                 ff  ff
                             LAB_ffffff7b                                    XREF[1]:     fffffd2d (j)   
        ffffff7b e9  13  fb       JMP        LAB_fffffa93
                 ff  ff
                             LAB_ffffff80                                    XREF[1]:     fffffa93 (j)   
        ffffff80 e9  47  ff       JMP        LAB_fffffecc
                 ff  ff
                             LAB_ffffff85                                    XREF[1]:     fffffecc (j)   
        ffffff85 e9  52  ff       JMP        LAB_fffffedc
                 ff  ff
                             LAB_ffffff8a                                    XREF[1]:     ffffff62 (j)   
        ffffff8a e9  16  f9       JMP        LAB_fffff8a5
                 ff  ff
```