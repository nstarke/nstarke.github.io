# Ghidra-firmware-utils Adventure

Published: April 13, 2021

I recently revisited the problem of `ghidra-firmware-utils` ( [https://github.com/al3xtjames/ghidra-firmware-utils](https://github.com/al3xtjames/ghidra-firmware-utils) ) not working on Windows.  If you use the default gradle project in the current master branch, you will be greeted with the following error when you try to open a UEFI filesystem:

```
'byte[] firmware.common.EFIDecompressor.nativeDecompress(byte[])'
java.lang.UnsatisfiedLinkError: 'byte[] firmware.common.EFIDecompressor.nativeDecompress(byte[])'
	at firmware.common.EFIDecompressor.nativeDecompress(Native Method)
	at firmware.common.EFIDecompressor.decompress(EFIDecompressor.java:51)
	at firmware.uefi_fv.FFSCompressedSection.<init>(FFSCompressedSection.java:82)
	at firmware.uefi_fv.FFSSectionFactory.parseSection(FFSSectionFactory.java:64)
	at firmware.uefi_fv.UEFIFFSFile.<init>(UEFIFFSFile.java:147)
	at firmware.uefi_fv.UEFIFirmwareVolumeHeader.<init>(UEFIFirmwareVolumeHeader.java:252)
	at firmware.uefi_fv.UEFIFirmwareVolumeFileSystem.mount(UEFIFirmwareVolumeFileSystem.java:54)
	at firmware.uefi_fv.UEFIFirmwareVolumeFileSystemFactory.create(UEFIFirmwareVolumeFileSystemFactory.java:47)
	at firmware.uefi_fv.UEFIFirmwareVolumeFileSystemFactory.create(UEFIFirmwareVolumeFileSystemFactory.java:29)
	at ghidra.formats.gfilesystem.factory.FileSystemFactoryMgr.mountUsingFactory(FileSystemFactoryMgr.java:176)
	at ghidra.formats.gfilesystem.factory.FileSystemFactoryMgr.probe(FileSystemFactoryMgr.java:360)
	at ghidra.formats.gfilesystem.FileSystemService.probeFileForFilesystem(FileSystemService.java:672)
	at ghidra.formats.gfilesystem.FileSystemService.probeFileForFilesystem(FileSystemService.java:611)
	at ghidra.plugins.fsbrowser.FileSystemBrowserPlugin.doOpenFilesystem(FileSystemBrowserPlugin.java:258)
	at ghidra.plugins.fsbrowser.FileSystemBrowserPlugin.lambda$openFileSystem$0(FileSystemBrowserPlugin.java:117)
	at ghidra.util.task.TaskLauncher$2.run(TaskLauncher.java:117)
	at ghidra.util.task.Task.monitoredRun(Task.java:124)
	at ghidra.util.task.TaskRunner.lambda$startTaskThread$0(TaskRunner.java:104)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:630)
	at java.base/java.lang.Thread.run(Thread.java:832)

---------------------------------------------------
Build Date: 2020-Dec-29 1701 EST
Ghidra Version: 9.2.2
Java Home: C:\Program Files\Java\jdk-15.0.2
JVM Version: Oracle Corporation 15.0.2
OS: Windows 10 10.0 amd64
Workstation: [REDACTED]
```

The exception stems from this code [https://github.com/al3xtjames/ghidra-firmware-utils/blob/bccdc829bab1de8563ccbebe00056f9d57df2b9c/src/main/java/firmware/common/EFIDecompressor.java#L28](https://github.com/al3xtjames/ghidra-firmware-utils/blob/bccdc829bab1de8563ccbebe00056f9d57df2b9c/src/main/java/firmware/common/EFIDecompressor.java#L28):

```java
try {
    JNILibraryLoader.loadLibrary("efidecompress");
} catch (Throwable t) {
    Msg.showError(EFIDecompressor.class, null, "EFI Decompressor",
            "Failed to load libefidecompress native library");
}
```

`JNILibraryLoader.loadLibrary` is defined here ( [https://github.com/al3xtjames/ghidra-firmware-utils/blob/bccdc829bab1de8563ccbebe00056f9d57df2b9c/src/main/java/util/JNILibraryLoader.java#L38](https://github.com/al3xtjames/ghidra-firmware-utils/blob/bccdc829bab1de8563ccbebe00056f9d57df2b9c/src/main/java/util/JNILibraryLoader.java#L38) ) as such:


```java
public static void loadLibrary(String name) throws FileNotFoundException, UnsatisfiedLinkError {
    File libraryPath = Application.getOSFile(System.mapLibraryName(name));
    System.load(libraryPath.getAbsolutePath());
}
```

Initially I thought the call to `System.load` was being passed an incorrect path to `efidecompress.dll`, but I verified that the path was correct by adding an error message to the `JNILibraryLoader.loadLibrary` function.

As a side note, if you are trying to uninstall a Ghidra Plugin on windows, you must delete the corresponding extension entry in `%UserProfile%\.ghidra\.ghidra_9.2.2_PUBLIC\Extensions`.  In this case that is `%UserProfile%\.ghidra\.ghidra_9.2.2_PUBLIC\Extensions\ghidra-firmware-utils`.  

So if it isn't the path to the `.dll` file, wy are we getting an `UnsatisfiedLinkError` exception?

It turns out that if the method name does not perfectly match up with what JNI expects, this type of error will occur.  The exported JNI function names in `efidecompress.dll` should be:

```
Java_firmware_common_EFIDecompressor_nativeDecompress
Java_firmware_common_TianoDecompressor_nativeDecompress
```

This is because of the Java namespace these functions exist in.  In the current master branch with the current gradle build, is this what we have?

From a Visual Studio Command Prompt, we run `dumpbin.exe`:

```
dumpbin /exports efidecompress.dll
Microsoft (R) COFF/PE Dumper Version 14.28.29336.0
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file efidecompress.dll

File Type: DLL

  Section contains the following exports for efidecompress.dll

    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
           2 number of functions
           2 number of names

    ordinal hint RVA      name

          1    0 00002130 _Java_firmware_common_EFIDecompressor_nativeDecompress@12
          2    1 00002340 _Java_firmware_common_TianoDecompressor_nativeDecompress@12

  Summary

        2000 .data
        6000 .rdata
        1000 .reloc
        D000 .text

```

There are two problems here:

1) Each function name begins with an underscore
2) Each function name ends with `@12`

Both of these functions are due to the calling convention and how `cl.exe` compiles the function.  How do we correct both issues?

We need to use `ming-w64`.  I used the installer from [https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win32/Personal%20Builds/mingw-builds/installer/mingw-w64-install.exe](https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win32/Personal%20Builds/mingw-builds/installer/mingw-w64-install.exe) to install the `x86_64` version of `ming-w64`.  This will install the `ming-w64` cross compiler to:

```
C:\Program Files\mingw-w64\x86_64-8.1.0-posix-seh-rt_v6-rev0\mingw64\bin
```

We can fix the leading underscore issue by nature of just using `ming-w64` to compile; it will compile `efidecompress.dll` without the leading underscores in the exported function names.  But what about the suffix `@12`?

Turns out there is a linker flag that we can set to avoid that suffix.  The link flag is:

```
-Wl,--kill-at
```

We additioanlly need to specify the `-shared` linker flag for reasons I don't quite understand.

In addition to fixing the function name irregularities, we also need to build `efidecompress.dll` for the `x86_64` architecture.  The architecture needs to match that of the JVM installation.  I drafted the following `build.gradle` file to create a working `efidecompress.dll` and thus working `ghidra-firmware-utils` extension zip file.  Please note that this version of the gradle file will ONLY work on windows and will most likely cause problems on other architectures.  

```groovy
// Builds a Ghidra Extension for a given Ghidra installation.
//
// An absolute path to the Ghidra installation directory must be supplied either by setting the
// GHIDRA_INSTALL_DIR environment variable or Gradle project property:
//
//     > export GHIDRA_INSTALL_DIR=<Absolute path to Ghidra>
//     > gradle
//
//         or
//
//     > gradle -PGHIDRA_INSTALL_DIR=<Absolute path to Ghidra>
//
// Gradle should be invoked from the directory of the project to build.  Please see the
// application.gradle.version property in <GHIDRA_INSTALL_DIR>/Ghidra/application.properties
// for the correction version of Gradle to use for the Ghidra installation you specify.

import org.apache.tools.ant.taskdefs.condition.Os

//----------------------START "DO NOT MODIFY" SECTION------------------------------
def ghidraInstallDir

if (System.env.GHIDRA_INSTALL_DIR) {
	ghidraInstallDir = System.env.GHIDRA_INSTALL_DIR
}
else if (project.hasProperty("GHIDRA_INSTALL_DIR")) {
	ghidraInstallDir = project.getProperty("GHIDRA_INSTALL_DIR")
}

if (ghidraInstallDir) {
	apply from: new File(ghidraInstallDir).getCanonicalPath() + "/support/buildExtension.gradle"
}
else {
	throw new GradleException("GHIDRA_INSTALL_DIR is not defined!")
}
//----------------------END "DO NOT MODIFY" SECTION-------------------------------

apply plugin: "c"
model {
	platforms {
        x64 {
            architecture "x86_64"
        }
    }
	toolChains {
		gcc(Gcc) {
			path "C:\\Program Files\\mingw-w64\\x86_64-8.1.0-posix-seh-rt_v6-rev0\\mingw64\\bin"
			eachPlatform {
				cCompiler.executable = "gcc.exe"
			}
		}
	}
	components {
		efidecompress(NativeLibrarySpec) {
			targetPlatform "x64"
			sources {
				c {
					source {
						srcDir "src/efidecompress/c"
						include "efidecompress.c"
					}
				}
			}

			binaries.all {
				cCompiler.args "-DCONFIG_JNI"
				if (targetPlatform.operatingSystem.macOsX) {
					cCompiler.args "-I", "${System.properties['java.home']}/include"
					cCompiler.args "-I", "${System.properties['java.home']}/include/darwin"
					cCompiler.args "-mmacosx-version-min=10.9"
					linker.args "-mmacosx-version-min=10.9"
				} else if (targetPlatform.operatingSystem.linux) {
					cCompiler.args "-I", "${System.properties['java.home']}/include"
					cCompiler.args "-I", "${System.properties['java.home']}/include/linux"
					cCompiler.args "-D_FILE_OFFSET_BITS=64"
				} else if (targetPlatform.operatingSystem.windows) {
					cCompiler.args "-I${System.properties['java.home']}\\include"
					cCompiler.args "-I${System.properties['java.home']}\\include\\win32"
					cCompiler.args "-D_JNI_IMPLEMENTATION_"
					linker.args "-Wl,--kill-at"
					linker.args "-shared"
				}
			}
		}
	}
}

repositories {
	mavenCentral()
}

configurations {
	toCopy
}

dependencies {
	toCopy group: "org.tukaani", name: "xz", version: "1.8"
}

task copyLibraries(type: Copy, dependsOn: "efidecompressSharedLibrary") {
	copy {
		from configurations.toCopy into "lib"
	}

	if (Os.isFamily(Os.FAMILY_MAC)) {
		from "$buildDir/libs/efidecompress/shared/libefidecompress.dylib" into "os/osx64"
	} else if (Os.isFamily(Os.FAMILY_UNIX)) {
		from "$buildDir/libs/efidecompress/shared/libefidecompress.so" into "os/linux64"
	} else if (Os.isFamily(Os.FAMILY_WINDOWS)) {
		from "$buildDir/libs/efidecompress/shared/efidecompress.dll" into "os/win64"
	}
}

buildExtension.dependsOn "copyLibraries"

task cleanLibraries(type: Delete) {
	delete fileTree("lib").matching {
		include "*.jar"
	}

	delete fileTree("os").matching {
		include "osx64/*.dylib"
		include "linux64/*.so"
		include "win64/*.dll"
	}
}

clean.dependsOn "cleanLibraries"
```

This `build.gradle` file will compile a 64-bit version of `efidecompress.dll`.  It is necessary to compile a 64-bit version of the library - a 32-bit version will not work on64-bit systems.

I'm not sure how to refactor this `build.gradle` file to work with all architectures so I will create a Github Issue and work with the maintainers to get it in an acceptable state and merged.

[Back](/)