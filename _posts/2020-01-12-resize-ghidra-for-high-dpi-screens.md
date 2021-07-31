---
layout: post
title:  "Resize Ghidra for High DPI screens"
date:   2020-01-12 00:00:00 -0600
categories: ghidra hidpi configuration
author: Nicholas Starke
---

If you run Ghidra on a high DPI screen, you will probably find the GUI to be scaled down so small to be almost of no use.  

There is a setting that you can adjust to scale the Ghidra GUI:

in `$GHIDRA_ROOT/support` is a file named `launch.properties`.  In this `launch.properties` file is the following configuration key:

```
VMARGS_LINUX=-Dsun.java2d.uiScale=1
```

Change this line to:

```
VMARGS_LINUX=-Dsun.java2d.uiScale=2
```

Then launch ghidra and you should be good to go!