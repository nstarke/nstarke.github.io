# A script to grab Iowa House Sessions and concatenate them together

Published: March 22, 2016

```bash
#!/bin/bash
# Example of Base URL: http://sg001-vod.sliq.net/00285-vod/_definst_/2016/03/House%20in%20Session_2016-03-22-13.58.50_2461_2.mp4
BASEURL=$1

# MAX only works up to 999 because of "seq -f "%03g".  Change "%03g" as your order of magnitude increases.
MAX=$2

for i in $(seq -f "%03g" 0 $MAX); do
    wget "$BASEURL/media_$i.ts" -O /tmp/video-$i.mp4
done

# change the ulimit -n 1024 to a higher number if your MAX ever gets higher than 900 or so.
ulimit -n 1024

# Must have FFMPEG installed for this to work
ffmpeg -i concat:"$(ls /tmp/*.mp4 | tr '\n' '|')" -codec copy -bsf:a aac_adtstoasc output.mp4
```

[Back](https://nstarke.github.io/)