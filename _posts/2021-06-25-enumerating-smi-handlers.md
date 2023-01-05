---
layout: posts
title:  "Enumerating SMI Handlers"
date:   2021-06-25 00:00:00 -0600
categories: smi uefi ghidra
author: Nicholas Starke
---

SMI Handlers make interesting targets when auditing UEFI implementations because they execute with the highest privilege level available on the platform (ring -2, System Management Mode).  As an interrupt, they are often small, self contained functions meant to execute quickly.  There have been a [few](https://github.com/Cr4sh/Aptiocalypsis) [vulnerabilities](https://github.com/tandasat/SmmExploit) found in SMI Handlers.

This write up will document my technique for enumerating SMI Handlers using Ghidra.  During this write up I will use the following firmware image as an example:

[https://dlcdnets.asus.com/pub/ASUS/nb/UX360CA/UX360CAAS304.zip](https://dlcdnets.asus.com/pub/ASUS/nb/UX360CA/UX360CAAS304.zip)

Unzipping this file results in a file named `UX360CA-AS.304`.  This is a UEFI update capsule.


## Ghidra Extensions

There are two ghidra extensions I use for this task:

* [EfiSeek](https://github.com/DSecurity/efiSeek)
* [Ghidra-firmware-utils](https://github.com/al3xtjames/ghidra-firmware-utils)

## EfiSeek

Efiseek will run as Ghidra Extension as part of the ghidra auto analysis engine.  You do not have to do anything special, once installed, to run this extension.  The extension can be built using [gradle](https://gradle.org/releases/) on Linux, MacOS, and Windows. It will run with analyzeHeadless without any problems or modifications.  The primary function EfiSeek provides is the renaming of SmiHandler functions according to UEFI protocol analysis.

## Ghidra-firmware-utils

Ghidra-firmware-utils is useful here because it can extract the UEFI filesystem into individual PE32 binaries and maintain the structure of the filesystem hierarchy.  We will use this to enumerate all binaries in a UEFI implementation, analyze them, and then use a custom Ghidra Script (different than a Ghidra Extension) to search for the functions that were renamed by `EfiSeek`.  

Ghidra-firmware-utils will not run headlessly without a few changes.  I have a small diff provided here that I use to run this extension headlessly.  You will need to apply this diff in order to be able to use `analyzeHeadless` with Ghidra-firmware-utils:

```diff
diff --git a/ghidra_scripts/UEFIHelper.java b/ghidra_scripts/UEFIHelper.java
index 5d96cc2..07ae7fb 100644
--- a/ghidra_scripts/UEFIHelper.java
+++ b/ghidra_scripts/UEFIHelper.java
@@ -30,6 +30,7 @@ import ghidra.app.util.bin.format.pe.NTHeader;
 import ghidra.app.util.bin.format.pe.PeSubsystem;
 import ghidra.app.util.bin.format.pe.PortableExecutable;
 import ghidra.app.util.opinion.PeLoader;
+import ghidra.app.plugin.core.analysis.AutoAnalysisManager;
 import ghidra.app.services.DataTypeManagerService;
 import ghidra.program.model.address.Address;
 import ghidra.program.model.data.*;
@@ -71,7 +72,9 @@ public class UEFIHelper extends GhidraScript {
        private DataTypeManager loadDataTypeLibrary(String name) throws Duplicat
eIdException,
                        IOException {
                // Check if the data type library was already loaded.
-               DataTypeManagerService service = state.getTool().getService(Data
TypeManagerService.class);
+               //DataTypeManagerService service = state.getTool().getService(Da
taTypeManagerService.class);
+               AutoAnalysisManager aam = AutoAnalysisManager.getAnalysisManager(currentProgram);
+               DataTypeManagerService service = aam.getDataTypeManagerService();
                DataTypeManager[] managers = service.getDataTypeManagers();
                for (DataTypeManager manager : managers) {
                        if (manager.getName().equals(name)) {
@@ -80,7 +83,7 @@ public class UEFIHelper extends GhidraScript {
                }

                // Load the data type library from the plugin's data directory.
-               return service.openArchive(loadDataFile(name), false).getDataTypeManager();
+               return service.openDataTypeArchive(name);
        }

        /**
```

Additionally, if you wish to compile on Windows, you will need to patch the following diff to successfully build on 64-bit Windows platforms:

```diff
diff --git a/build.gradle b/build.gradle
index 6e52ff8..a0a8b58 100644
--- a/build.gradle
+++ b/build.gradle
@@ -36,8 +36,14 @@ else {

 apply plugin: "c"
 model {
+       platforms {
+        x64 {
+            architecture "x86_64"
+        }
+    }
        components {
                efidecompress(NativeLibrarySpec) {
+                       targetPlatform "x64"
                        sources {
                                c {
                                        source {
```

The extension can be built using [gradle](https://gradle.org/releases/) on Linux, MacOS, and Windows.

## Adding Extensions
Once you have used gradle to compile the two extensions mentioned above, you must add them to the ghidra extension list.  This can be done in the Ghidra Project menu by going to `File->Install Extensions`

![Install Extensions](/images/0053/install-extensions.PNG "Install Extensions Screenshot")

You will then need to restart Ghidra for these extensions to take effect.

## Opening UEFI image

Next we will open up the Ghidra CodeBrowser, and import our UEFI image.  When opening the file we downloaded above (`UX360CA-AS.304`), you will be presented with a `Container File` Dialog:

![Container File Dialog](/images/0053/container-file.PNG "Container File Dialog Screenshot")

Select `File system` from the list of options.  Ghidra-firmware-utils will then import the UEFI filesystem into Ghidra.  

Some firmware images, especially those pulled directly off chip, will require you select between the Intel File Descriptor format and the UEFI image.  IF you see this option, select `UEFI Image`.

Now you will be presented with a list of files:

![File Browser](/images/0053/file-browser.PNG "File Browser Dialog Screenshot")

Next we need to batch import the PE32 files from the UEFI image to the ghidra project.  Select the Root node on the File Browser dialog and right click on it.  Navigate to `Batch Import` in the context menu.

![Batch Import](/images/0053/batch-import.PNG "Batch Import Dialog Screenshot")

Deselect `PCI Option Rom`.  Ghidra can almost never import these for some reason and the failure to import prevents all the other modules from being loaded into ghidra successfully. If you see something like this following image, you have imported the PE32 components of the UEFI image successfully:

![Batch Import Success](/images/0053/batch-import-success.PNG "Batch Import Success Dialog Screenshot")

At this point, we need to close out of Ghidra entirely.  We are going to run the headless analyzer over the entire project.

## Analyze Headless

Open up a command prompt and cd into $GHIDRA_INSTALL_DIR/support.  This is the location of the analyzeHeadless scripts.  We want to run this script over our nearly created UEFI image ghidra project.  You will need to know

* The location of your ghidra-projects directory (Usually this is $HOME/ghidra-projects)
* The name of your Ghidra project

For example:

```
analyzeHeadless.bat c:\Users\stark\ghidra-projects asus-304 -process -recursive -postScript UEFIHelper.java
```

Where `c:\Users\stark\ghidra-projects` is the ghidra-project directory, and `asus-304` is the ghidra project name.  Additioanlly, `UEFIHElper.java` is the analysis entry point for ghidra-firmware-utils.  State differently, running `UEFIHelper.java` is how we analyze a given UEFI module using ghidra-firmware-utils.  You can optionally leave off this argument (run the aforementioned command WITHOUT `-postScript UEFIHelper.java`) if you do not wish to use the ghidra-firmware-utils analysis.  Running both the EfiSeek and ghidra-firmware-utils analysis will result in duplicate protocol bookmarks in the individual binaries, but I find this is not such a big deal.

This analysis task will take a long time and depends on how big the firmware image is; how many UEFI modules are present.  On this particular UEFI image, the analysis took 2 hours on a 16-core system.

Once completed, we can begin enumerated SMI Handlers.

## Searching for Handlers

After the analysis phase, open up the Ghidra UI and load the project we just analyzed.

During the analysis phase, EfiSeek should have identified SMI Handler functions based off the UEFI Protocol GUID for SMI Handlers and renamed the functions accordingly.  Thus, if we are able to do a search for function names containing the string `SmiHandler`, we should be able to list all the functions that handle SMI.  Unfortunately, by default, there is no Ghidra Script that can do a "contains" search on function names.  We must build a script that can do this.  Fortunately, there is an exact match function name script that can be easily modified to do a "contains" search.  I present the "contains" search module here:

```java
/* ###
 * IP: GHIDRA
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 * 
 *      http://www.apache.org/licenses/LICENSE-2.0
 * 
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
//Opens all programs under a chosen domain folder, scans them for functions
//that contain a user supplied name, and prints info about the match.
//@category FunctionID
import ghidra.app.script.GhidraScript;
import ghidra.feature.fid.service.FidService;
import ghidra.framework.model.DomainFile;
import ghidra.framework.model.DomainFolder;
import ghidra.program.database.ProgramContentHandler;
import ghidra.program.model.listing.*;
import ghidra.util.Msg;
import ghidra.util.exception.CancelledException;
import ghidra.util.exception.VersionException;

import java.io.IOException;
import java.util.ArrayList;

public class FindContainsFunction extends GhidraScript {

	FidService service;

	@Override
	protected void run() throws Exception {
		service = new FidService();

		DomainFolder folder =
			askProjectFolder("Please select a project folder to RECURSIVELY look for a named contains function:");
		String name =
			askString("Please enter function name substring",
				"Please enter the function name substring you're looking for:");

		ArrayList<DomainFile> programs = new ArrayList<DomainFile>();
		findPrograms(programs, folder);
		findFunction(programs, name);
	}

	private void findFunction(ArrayList<DomainFile> programs, String name) {
		for (DomainFile domainFile : programs) {
			if (monitor.isCancelled()) {
				return;
			}
			Program program = null;
			try {
				program = (Program) domainFile.getDomainObject(this, false, false, monitor);
				FunctionManager functionManager = program.getFunctionManager();
				FunctionIterator functions = functionManager.getFunctions(true);
				for (Function function : functions) {
					if (monitor.isCancelled()) {
						return;
					}
					if (function.getName().contains(name)) {
						println("found " + function.getName() + " in " + domainFile.getPathname());
					}
				}
			}
			catch (Exception e) {
				Msg.warn(this, "problem looking at " + domainFile.getName(), e);
			}
			finally {
				if (program != null) {
					program.release(this);
				}
			}
		}
	}

	private void findPrograms(ArrayList<DomainFile> programs, DomainFolder folder)
			throws VersionException, CancelledException, IOException {
		DomainFile[] files = folder.getFiles();
		for (DomainFile domainFile : files) {
			if (monitor.isCancelled()) {
				return;
			}
			if (domainFile.getContentType().equals(ProgramContentHandler.PROGRAM_CONTENT_TYPE)) {
				programs.add(domainFile);
			}
		}
		DomainFolder[] folders = folder.getFolders();
		for (DomainFolder domainFolder : folders) {
			if (monitor.isCancelled()) {
				return;
			}
			findPrograms(programs, domainFolder);
		}
	}
}

```

Save this file in your `ghidra_scripts` directory (usually in `$HOME`) as `FindContainsFunction.java`.  The file name is important as it has to match the Class name (`FindContainsFunction`).

Now we can run this script in the Ghidra Script manager to enumerate our SmiHandlers.  The console output will write out in the Ghidra console window.

The script takes two pieces of input.  The first is where in the imported directory tree you would like to do the search.  Everything under your selected entry point will be searched for the string you provide as the second piece of input.  The second piece of input will be compared with all function names in the directory structure, and substring matches will be reported through the console output.

**Script Manager**

![Script Manager](/images/0053/script-manager.PNG "Script Manager Dialog Screenshot")

**Script Input 1**

![Script Input 1](/images/0053/script-input1.PNG "Script Input 1 Screenshot")

**Script Input 2**

![Script Input 2](/images/0053/script-input2.PNG "Script Input 2 Screenshot")

**Script Output**

![Script Output](/images/0053/script-output.PNG "Script Output Screenshot")

## Script Output

This is the script output for this file:

```
FindContainsFunction.java> Running...
FindContainsFunction.java> found swSmiHandler37 in /UX360CA-AS.304/Volume 001 - 7dccf422-45f6-4951-a557-cc421da31599/File 000 - WinflashDxeSmi/PE32 Image Section
FindContainsFunction.java> found swSmiHandler10 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 012 - CpuSpSMI/PE32 Image Section
FindContainsFunction.java> found swSmiHandler11 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 012 - CpuSpSMI/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler9 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 153 - NvramSmi/PE32 Image Section
FindContainsFunction.java> found swSmiHandler7 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 156 - PiSmmCommunicationSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler12 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 157 - NbSmi/PE32 Image Section
FindContainsFunction.java> found swSmiHandler10 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 161 - AcpiModeEnable/PE32 Image Section
FindContainsFunction.java> found swSmiHandler11 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 161 - AcpiModeEnable/PE32 Image Section
FindContainsFunction.java> found swSmiHandler7 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 162 - SmramSaveInfoHandlerSmm/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler21 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 166 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler19 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 166 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler18 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 166 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler16 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 166 - PowerMgmtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler11 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 173 - MicrocodeUpdate/PE32 Image Section
FindContainsFunction.java> found swSmiHandler9 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 175 - BootScriptHideSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler8 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 175 - BootScriptHideSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler23 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 178 - UsbRtSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler9 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 180 - CmosSmm/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler7 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 181 - FlashSmiSmm/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler20 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found swSmiHandler17 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler16 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler21 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler18 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler19 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 182 - SmmHddSecurity/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler6 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 184 - NvmeSmm/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler10 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 185 - SmmPcieSataController/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler10 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 186 - SdioSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler11 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 186 - SdioSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler21 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 187 - SecSMIFlash/PE32 Image Section
FindContainsFunction.java> found swSmiHandler15 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 188 - SmbiosDmiEdit/PE32 Image Section
FindContainsFunction.java> found swSmiHandler22 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 189 - SmiFlash/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler4 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 190 - SmmLockBox/PE32 Image Section
FindContainsFunction.java> found swSmiHandler5 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 192 - TcgSmm/PE32 Image Section
FindContainsFunction.java> found swSmiHandler31 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 195 - AsusSmiEntry/PE32 Image Section
FindContainsFunction.java> found swSmiHandler33 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 195 - AsusSmiEntry/PE32 Image Section
FindContainsFunction.java> found ChildSmiHandler12 in /UX360CA-AS.304/Volume 002 - 4f1c52d3-d824-4d2a-a2f0-ec40c23c5916/File 002 - 9e21fd93-9c72-4c15-8c4b-e77f1db2d792/GUID-Defined Section - LzmaCustomDecompressGuid/Firmware Volume Image Section/Volume 000 - 5c60f367-a505-419a-859e-2a4ff6ca6fe5/File 229 - PchSmiDispatcher/PE32 Image Section
FindContainsFunction.java> Finished!
```