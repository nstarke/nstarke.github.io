---
layout: posts
title:  "SMM Callouts via Notify"
date:   2022-04-05 00:00:00 -0600
categories: uefi smm
author: Nicholas Starke
---

I was recently working on assessing a product's UEFI firmware and I learned something new that didn't seem well documented on the internet.  

Many times, when a Software SMI Handler (SwSmiHandler) is registered in an SMM module, the function **SmmRegisterProtocolNotify** is also called, which registers a function to a Protocol GUID. efiSeek labeled the registered function as a SmiHandler, but I wanted to find out if functions registered with **SmmRegisterProtocolNotify** are called when the swSmiHandler is called.  With some help from [@vincentzimmer](https://twitter.com/vincentzimmer) I was able to trace through the EDK2 source code to pin point how the functions in question are invoked.

For reference, this is what a normal SwSmiHandler registration looks like (in Ghidra, with efiSeek):

```c++
local_18 = 0x35;
local_10a8 = (*EFI_SMM_SW_DISPATCH2_PROTOCOL4->Register)
                        (EFI_SMM_SW_DISPATCH2_PROTOCOL4,swSmiHandler8,local_18,&local_10b0) ;
```

In the above example, **swSmiHandler8** is the function invoked in SMM when `0x35` is passed as the second operand  to the `out` instruction (where `0xb2` is the first operand). When this instruction is executed with these operands, the CPU transitions to System Management Mode (SMM) and then executes **swSmiHandler8** in SMM.

The [SmmRegisterProtocolNotify](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Notify.c#L96) function is defined here.

The function is registered to a *ProtocolNotify* object as a function pointer for its **Function** property [here](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Notify.c#L177)

So where is this **Function**  property function pointer invoked? The actual invocation occurs within [SmmNotifyProtocol](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Notify.c#L29), but if we trace through the call graph a bit we see the **SmmNotifyProtocol** function is called [here](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Handle.c#L326). 

This is within the [SmmInstallProtocolInterfaceNotify](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Handle.c#L208). The **SmmInstallProtocolInterfaceNotify** function is then called [here](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Handle.c#L181) within the [SmmInstallProtocolInterface](https://github.com/tianocore/edk2/blob/7c0ad2c33810ead45b7919f8f8d0e282dae52e71/MdeModulePkg/Core/PiSmmCore/Handle.c#L174).  

Since **SmmInstallProtocolInterface** has the **EFIAPI** qualifier, it is part of the public EFI API that is exposed to and used within modules.

When **SmmInstallProtocolInterface** is called within an SMM Module, the **Function** registered with **SmmRegisterProtocolNotify** the **Function** is called in SMM mode.  There is no context switch.

This becomes interesting when two criteria are met: 

1) When **SmmInstallProtocolInterface** is called within a normally registered SwSmiHandler that can be triggered by writing a byte to Port IO via **out(0xb2, $BYTE)**.

2) When the **Function** registered makes calls to **BootServices** or **RuntimeServices** (from the System Table - not the SMST).

In the case where these two conditions are met, an SMM Callout vulnerability can occur. Note that these two conditions are global to the UEFI image and may be met in different modules.  

This is what a typical **SmmRegisterProtocolNotify** looks like:

```c++
(*gSmst23->SmmRegisterProtocolNotify)(&EFI_EMUL6064TRAP_PROTOCOL_GUID,notify_6ea0f71c,local_18)
```

**notify_6ea0f71c** is the function that is called when **SmmInstallProtocolInterface** is called with the **EFI_EMUL6064TRAP_PROTOCOL_GUID** parameter argument. This generally looks like this:

```c++
(*gSmst9->SmmInstallProtocolInterface)
    (&gHandle23,&EFI_EMUL6064TRAP_PROTOCOL_GUID,EFI_NATIVE_INTERFACE,
        &gEFI_EMUL6064TRAP_PROTOCOL_24);
```

When you see **SmmInstallProtocolInterface** within an SwSmiHandler and if **notify_6ea0f71c** has references to **gBS** or **gRS** (but not the SMM Runtime Services!) then there may be SMM Callout Vulnerabilities!

Thanks to [@vincentzimmer](https://twitter.com/vincentzimmer) and [@hackingthings](https://twitter.com/hackingthings) for answering questions and otherwise just generally being helpful.