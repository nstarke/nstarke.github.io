---
layout: posts
title:  "Is it JS or Tar? A Polyglot Story"
date:   2022-03-02 00:00:00 -0600
categories: javascript tar polyglot
author: Nicholas Starke
---

![Polyglot](/images/03022022/polyglot.png "Polyglot screenshot")

I've been taking a break from firmware hacking for a bit, and so as part of that I figured out how to make a polyglot file that is both a valid, executable JavaScript file and a GNU Tar Archive. Basically I did this by abusing multiline JavaScript strings. In JavaScript, everything that is not executable appears as a multiline strings.  These multiline strings can be divided up: there is a first multiline string, then the executable payload, then the second multiline string.

This small shell script will take a path to a JavaScript file as a payload as a first argument, and then a file path for output as the second argument.

The first thing I do after taking care of the positional script arguments is load the JavaScript payload into a bash variable (`$PAYLOAD_SCRIPT`).  I then touch a file named **; `**. This file will serve as the beginning of the first string encapulation.

Next I configure the payload contents to begin with **\` ;**. I use `echo -n` to emit this string because the `-n` flag echoes out without a trailing newline.  This string is redirected to the `$OUTPUT_FILE` path. This output becomes the end of the first multiline string.

The next line then echoes out the contents of the payload file (`$PAYLOAD_SCRIPT`), again using echoes `-n` flag to omit the trailing newline character. This inserts the JavaScript I want to execute.

After echoing out the payload contents, I echo out the strings **; \`** to the `$OUTPUT_FILE` file path. This serves as the beginning of the second multiline string.  

Now that the payload has been created successfully, I need to tar up the two files I created above.  I tar up the two files without compression.

Lastly, I add a single **\`** character to the end of the file. This is to end the second multiline string. GNU Tar will ignore this trailing content when performing operations over the the crafted `.tar.js` file.  This trailer character is absolutely necessary as the JavaScript will not execute without it.

## Automation Script

```bash
#!/bin/bash
#
# Bash script to generate polyglot JavaScript/GNU Tar file
# Author: Nicholas Starke
#

PAYLOAD_FILE="$1"
OUTPUT_FILE="$2"
PAYLOAD_SCRIPT="$(cat $PAYLOAD_FILE)"
touch '; `'
echo -n '` ; ' > $OUTPUT_FILE
echo -n $PAYLOAD_SCRIPT >> $OUTPUT_FILE
echo -n '; `' >> $OUTPUT_FILE
tar -c -f "$OUTPUT_FILE.tar.js" '; `' "$OUTPUT_FILE"
echo -n '`' >> "$OUTPUT_FILE.tar.js"
# clean up the temporary output file.
rm "$OUTPUT_FILE"
exit 0
```

## Hexdump

```
00000000: 3b20 6000 0000 0000 0000 0000 0000 0000  ; `.............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 3030 3030 3634 3400 3030 3031  ....0000644.0001
00000070: 3735 3000 3030 3031 3735 3000 3030 3030  750.0001750.0000
00000080: 3030 3030 3030 3000 3134 3230 3737 3633  0000000.14207763
00000090: 3737 3300 3031 3030 3133 0020 3000 0000  773.010013. 0...
000000a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000100: 0075 7374 6172 2020 006e 6963 6b00 0000  .ustar  .nick...
00000110: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000120: 0000 0000 0000 0000 006e 6963 6b00 0000  .........nick...
00000130: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000140: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000150: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000160: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000170: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000180: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000190: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000200: 6f75 7470 7574 0000 0000 0000 0000 0000  output..........
00000210: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000220: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000230: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000240: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000250: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000260: 0000 0000 3030 3030 3634 3400 3030 3031  ....0000644.0001
00000270: 3735 3000 3030 3031 3735 3000 3030 3030  750.0001750.0000
00000280: 3030 3030 3034 3400 3134 3230 3737 3633  0000044.14207763
00000290: 3737 3300 3031 3130 3131 0020 3000 0000  773.011011. 0...
000002a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000300: 0075 7374 6172 2020 006e 6963 6b00 0000  .ustar  .nick...
00000310: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000320: 0000 0000 0000 0000 006e 6963 6b00 0000  .........nick...
00000330: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000340: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000350: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000360: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000370: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000380: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000390: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000400: 6020 3b20 616c 6572 7428 274a 532f 5441  ` ; alert('JS/TA
00000410: 5220 706f 6c79 676c 6f74 2058 5353 2729  R polyglot XSS')
00000420: 3b3b 2060 0000 0000 0000 0000 0000 0000  ;; `............
00000430: 0000 0000 0000 0000 0000 0000 0000 0000  ................
[...]
00002790: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000027f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00002800: 60                                       `
```