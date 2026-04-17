# DUMP and IAT Reconstruction

## Principle

After finding the program's OEP, we need to dump the program and rebuild the `IAT`. `IAT` stands for `Import Address Table`, whose entries point to the actual addresses of functions.

## Example

For instance, as shown below, we have found the OEP and reached the program's true entry point. Now we need to dump the program. We right-click and select `"Dump debugged process with OllyDump"` (though you can also use `LoadPE` to perform the dump):

![right_click.jpg](./figure/fix_iat/right_click.jpg)

A window pops up. Check whether the addresses are correct — mainly verify that the `Entry Point Address` is selected correctly. Then uncheck `Rebuild Import Table`.

![dump.png](./figure/fix_iat/dump.png)

Name the dumped file — in my case I named it `dump.exe`. Let's try running `dump.exe`. You'll find that the program cannot run normally. For some simple packers, if you dump the file and find it cannot run properly, and you have confirmed that you found the correct OEP and the decompiled results in `IDA` look fine, then your first thought should be that there is a problem with the program's `IAT`. You need to rebuild the `IAT`.

We need to use `ImportREC` to help repair the import table.

Open `ImportREC` and select the running process `原版.exe` (`原版.exe` is the process I am debugging in OD, where `EIP` is currently at the `OEP` position — do not close this process after using `OllyDump`). `ImportREC` needs to know the `OEP` to repair the import table entry point, which means you need to enter it in the `OEP` input field on the right side of the window.

![importrec.png](./figure/fix_iat/importrec.png)

As we know, in OllyDbg the current entry point of the program is `0049C25C`, and the image base is `00400000`.

Therefore, the `OEP` we need to fill in here is `0009C25C`.

We modify the `OEP` in `ImportREC` to `0009C25C` and click `AutoSearch`. A prompt pops up saying "Found possible original IAT address".

![auto_search.png](./figure/fix_iat/auto_search.png)

We click the `"Get Imports"` button to rebuild the `IAT`. The left side will display the addresses of each imported function in the `IAT` and whether they are valid. As shown in the figure, `ImportREC` has found the location of the `IAT` in memory and detected that all functions are valid.

![get_imports.png](./figure/fix_iat/get_imports.png)

We click `Fix Dump`, then open the file previously dumped using the `OllyDump` plugin, which is the `dump.exe` file.

Then `ImportREC` will help restore the import table and generate a `dump_.exe` file. `dump_.exe` can run normally.
