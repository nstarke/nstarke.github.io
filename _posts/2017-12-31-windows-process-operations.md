---
layout: posts
title:  "Windows Process Token Bitwise AND To Get Real Value"
date:   2017-12-31 00:00:00 -0600
categories: windows kernel
author: Nicholas Starke
---

The value stored in the `_EPROCESS` object must be AND'd against a certain value to produce the actual token.
For more information, see [http://mcdermottcybersecurity.com/articles/x64-kernel-privilege-escalation](http://mcdermottcybersecurity.com/articles/x64-kernel-privilege-escalation)

## Windows 32-bit:
```
kd> ?[TOKEN] & 0xFFFFFFF8
```

## Windows 64-bit:
```
kd> ?[TOKEN] & 0xFFFFFFFFFFFFFFF0
```
