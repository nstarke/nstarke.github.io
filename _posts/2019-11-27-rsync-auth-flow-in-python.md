---
layout: posts
title:  "Rsync Protocol Auth Flow in Python"
date:   2019-11-27 00:00:00 -0600
categories: python rsync authentication
author: Nicholas Starke
---

This is an implementation of the authentication flow in the rsync protocol.

```python
#!/usr/bin/env python
from Crypto.Hash import MD4

import socket
import base64
import os
import random
import time
import sys

BUF_SIZE = 4096

HOST = sys.argv[1]
PORT = 873

USERNAME = "admin"
PASSWORD = "password"

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))
s.send("@RSYNCD: 29\n")
print(s.recv(BUF_SIZE))
s.send("#list\n")
l = s.recv(BUF_SIZE).rstrip()
s.close()
# Second Connection
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, 873))
s.send("@RSYNCD: 29\n")
print(s.recv(BUF_SIZE))
s.send(l + "\n")
auth = s.recv(BUF_SIZE)
print(auth)
auth = auth.rstrip().split(' ')[2]
h = MD4.new("\x00" * 4) # Gag me with a giant spoon
h.update(PASSWORD)
h.update(auth)
digest = base64.b64encode(h.digest())
digest = digest.replace("=", "")    
s.send(USERNAME + ' ' + digest + '\n')
print(s.recv(BUF_SIZE))
s.close()
```