# Extract Netgear .chk Firmware

I recently ran into a situation where `binwalk -M -e $FIRMWARE` failed me.  This was for a Netgear firmware image that ended in a `.chk`extension.

The firmware file name was `R7960P-V1.0.1.34_1.0.20.chk`.

This is the output when I ran `binwalk R7960P-V1.0.1.34_1.0.20.chk`:

```bash
$ binwalk R7960P-V1.0.1.34_1.0.20.chk

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
58            0x3A            JFFS2 filesystem, little endian
4063290       0x3E003A        UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000

```

We need the UBIFS portion of the `.chk` file, so either use `binwalk -M -e R7960P-V1.0.1.34_1.0.20.chk` or use `dd` to manually extract that portion of the image.

If you use `binwalk` you will see a folder with a file in it: `_R7960P-V1.0.1.34_1.0.20.chk.extracted/3E003A.ubi`.

Now this UBIFS blob contains a squahsfs image.  We need to extract that image instead of trying to extract files from the ubifs itself.

To accomplis this we will need [https://github.com/jrspruitt/ubi_reader](https://github.com/jrspruitt/ubi_reader), which can be installed via pip: `sudo pip install ubi_reader`.

Execute `$ ubireader_extract_images _R7960P-V1.0.1.34_1.0.20.chk.extracted/3E003A.ubi`

This will create a directory with a file in it: `ubifs-root/3E003A.ubi/img-1531400273_vol-rootfs_squashfs.ubifs`.

Now all we have to do is use `binwalk` to extract the squashfs filesystem:

```bash
$ binwalk -e ubifs-root/3E003A.ubi/img-1531400273_vol-rootfs_squashfs.ubifs

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0, compression:xz, size: 36995096 bytes, 2192 inodes, blocksize: 131072 bytes, created: 2019-02-24 22:58:06

```

That leave you with an extract squashfs file system:

```bash
ls ubifs-root/3E003A.ubi/_img-1531400273_vol-rootfs_squashfs.ubifs.extracted/squashfs-root 
bin/     data/   dev/  lib/    misc3/  mnt/  proc/  share/  tmp@  var/   www/
bootfs/  debug@  etc/  lib64/  misc5/  opt/  sbin/  sys/    usr/  webs/
```

[Back](https://nstarke.github.io/)
