---
layout: post
title:  "Search for Regex with Regex"
date:   2016-02-23 00:00:00 -0600
categories: regex security search grep
author: Nicholas Starke
---

This is a command I developed to search for regular expressions using egrep:

```bash
egrep -r -e "[\^]?(\.\*)|(\[\w+*\-\w+*\])|(\{\d+[\,\d+]?\})[\$]?" .
```