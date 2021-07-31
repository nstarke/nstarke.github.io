---
layout: posts
title:  "Python Find Information Scripts"
date:   2019-08-29 00:00:00 -0600
categories: python reverse-engineering
author: Nicholas Starke
---

I developed two scripts for finding information in binary files.  

The first one measures the string entropy and displays the results:

```python
#!/usr/bin/env python

#
# find-entropy.py
#
# A simple Utility to measure entropy of strings.
# Usage should be something like this:
#
# $ strings file.txt | python find-entropy.py
#
# The preset criteria of 3.75 should weed out most non-human created strings
#

import math
import string
import sys

def shannon_entropy(data):
    """
    Adapted from http://blog.dkbza.org/2007/05/scanning-data-for-entropy-anomalies.html
    by way of truffleHog (https://github.com/dxa4481/truffleHog)
    """
    if not data:
        return 0
    entropy = 0
    for x in string.printable:
        p_x = float(data.count(x)) / len(data)
        if p_x > 0:
            entropy += - p_x * math.log(p_x, 2)
    return entropy

for line in sys.stdin:
    entropy = shannon_entropy(line)
    if entropy > 3.75:
        print (line[:-1], entropy)
```

The second script searches for compressed data that doesn't have a header attached to it.  If you have embedded compressed data with headers, you are better off using [binwalk](https://github.com/refirmlabs/binwalk).

```python
#!/usr/bin/env python

#
# find-data.py
#
# A small script to bruteforce embedded compressed data that might not have a header
# Useful for raw binary firmware images that do not contain a standard
# binary header (ELF, PE, MACH-O).
#
# I included a limt on size at 16KB because this has a tendency to create
# lots of small files, which are generally false positives.
# 
# I usually run this over every firmware image I need to analyze.
#
# Usage: python find-data.py "filename.bin"
#

import zlib
import sys
import lzma
import bz2
import zipfile

LIMIT = 1024 * 16

with open(sys.argv[1], 'r') as compressed_data:
    compressed_data = compressed_data.read()
    for i in range(len(compressed_data)):
        try:
            unzipped = zlib.decompress(compressed_data[i:], -zlib.MAX_WBITS)
            if len(unzipped) > LIMIT:
                print ('GZIP: Offset found', i)
                with open('./result-gz-' + str(i) + '.bin', 'w') as result:
                    result.write(unzipped);
                    result.close()
        except Exception as ex:
            pass

        try: 
            unzipped = lzma.decompress(compressed_data[i:])
            if len(unzipped) > LIMIT:
                print ('LZMA: Offset Found', i)
		with open('./result-lzma-' + str(i) + '.bin', 'w') as result:
                    result.write(unzipped);
                    result.close()
	except Exception as ex:
	    pass
        try: 
            unzipped = bz2.decompress(compressed_data[i:])
            if len(unzipped) > LIMIT:
                print ('BZ2: Offset Found', i)
		with open('./result-bz2-' + str(i) + '.bin', 'w') as result:
                    result.write(unzipped);
                    result.close()
	except Exception as ex:
	    pass
```