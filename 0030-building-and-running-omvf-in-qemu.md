# Building and Running OVMF in Qemu

I built EDK2 and OVMF from source using the instructions here: [https://github.com/tianocore/tianocore.github.io/wiki/How-to-run-OVMF](https://github.com/tianocore/tianocore.github.io/wiki/How-to-run-OVMF)

The instructions are helpful in getting the build tooling configured to build edk2, but I consistently ran into a problem when I built the `DEBUG` version of OVMF. I would run:

```
$ qemu-system-x86_64 -bios ../edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd
```

And then I would get the standard QEMU menu which states:

```
Guest has not initialized the display yet
```

I finally figured out that this has something to do with the `DEBUG` build; maybe the QEMU machine is waiting for a debugger to attach.  I checked this by running:

```bash
$ qemu-system-x86_64 -bios ../edk2/./Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd -gdb tcp::1234
```

And then attaching gdb to the qemu stub.  Sure enough, the qemu instance was waiting for gdb!

However, even after continuing the processor execution, the `DEBUG` build takes a really long time to initialize the display and otherwise "get going".

I recommend setting the `TARGET` option in `Conf/target.txt` to `RELEASE`.  This will build the production version of OVMF that doesn't contain all the extraneous debugging utilities that make running the `DEBUG` version so slow.

After I had the `RELEASE` version built I was dropped into the UEFI shell within 20 second of boot under qemu. Most of that time is spent waiting for PXE to time out.  The command to do this is:

```bash
$ qemu-system-x86_64 -bios ../edk2/Build/OvmfX64/RELEASE_GCC5/FV/OVMF.fd -nographic
```

I needed to run with the `-nographic` flag because I was on a remote server over ssh and thus running headlessly.

[Back](https://nstarke.github.io/)