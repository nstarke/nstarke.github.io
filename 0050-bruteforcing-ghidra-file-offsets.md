# Bruteforcing Ghidra File Offsets

Published: June 6, 2021

Let's say you know the base address of a raw binary, but you don't know where in the binary the base address executes the first instruction.  You need a way of bruteforcing the file offset.  I wrote a small shell script to do just that.

This script relies on [this ghidra script](https://gist.github.com/nstarke/ea83d6e8aba9a8b028a94cc14f5ff00d) (included in Appendix A) being located in the `$USER_HOME/ghidra_scripts` dir.  This script counts the number of referenced strings.

The script takes four arguments:

1) the GHIDRA_PROJECTS directory path
2) the filename of the binary you wish to bruteforce
3) the Ghidra Language ID for the binary (for example, `ARM:LE:32:v7`)
4) the base load address of the binary.

The script uses `analyeHeadless` to check for referenced strings at file offsets in increments of 4 bytes. The referenced string count and total string count are written to `analysis.log`.  Generally speaking, the offset that has the highest referenced string count ( or the string count that most closesly matches the total count) is the correct file offset.

The script will create a project with a random name and delete the project after analysis.

I've used this script several times to bruteforce boot loader file offsets and thought it might help someone else.

```bash
#!/bin/bash
GHIDRA_PROJECTS=$1
FILENAME=$2
LANGID=$3
BASE_ADDR=$4
TMP_PROJECT_NAME=$RANDOM
CMD="$GHIDRA_HOME/support/analyzeHeadless \"$GHIDRA_PROJECTS\" \"$TMP_PROJECT_NAME\" -import \"$FILENAME\" -postScript CountReferencedStrings.java -processor \"$LANGID\" -deleteProject -loader BinaryLoader -loader-baseAddr \"$BASE_ADDR\" -loader-fileOffset"
COUNTER=0
for i in {0..128}
do
        echo "-----" >> analysis.log
        printf '0x%x\n' $COUNTER >> analysis.log
        $CMD $(printf '0x%x' $COUNTER) 2>&1 | grep "CountReferencedStrings.java>" >> analysis.log
        echo "-----" >> analysis.log
        COUNTER=$((COUNTER+4))
done
```

## Appendix A: CountReferencedStrings.java

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
//This script counts the references to existing strings.
//@category Analysis

import java.util.*;

import ghidra.app.script.GhidraScript;
import ghidra.program.model.address.Address;
import ghidra.program.model.mem.Memory;
import ghidra.program.util.DefinedDataIterator;
import ghidra.program.model.listing.Data;

public class CountReferencedStrings extends GhidraScript {

	@Override
	public void run() throws Exception {

		monitor.setMessage("Finding Strings with References");
		int referencedCount = 0;
		int totalCount = 0;
		for (Data nextData: DefinedDataIterator.definedStrings(currentProgram) ) {
			Address strAddr = nextData.getMinAddress();
			int refCount = currentProgram.getReferenceManager().getReferenceCountTo(strAddr);
			totalCount++;
			if (  refCount > 0 ) {
				referencedCount++;
			}
		}

		println("Number of referenced strings found: " + referencedCount);
		println("Total number of strings found: " + totalCount);
	}
}
```

[Back](/)