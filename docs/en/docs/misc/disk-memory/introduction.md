# Disk and Memory Analysis

## Common Tools

-   EasyRecovery
-   MedAnalyze
-   FTK
-   [Elcomsoft Forensic Disk Decryptor](https://ctf-wiki.github.io/ctf-tools/misc/#_6)
-   Volatility

## Disk

Common disk partition formats include the following:

-   Windows: FAT12 -> FAT16 -> FAT32 -> NTFS
-   Linux: EXT2 -> EXT3 -> EXT4
-   FAT Main Disk Structure

    ![](./figure/forensic-filesys.jpg)

-   Deleting a file: The first byte of the filename in the directory table becomes `e5`.

## VMDK

A VMDK file is essentially a virtual version of a physical hard disk. It also contains padding areas similar to those in physical hard disk partitions and sectors. We can use these padding areas to hide data we want to conceal. This approach avoids increasing the size of the VMDK file (as would happen if data were directly appended to the end of the file), and also avoids potential virtual machine errors caused by changes in VMDK file size. Moreover, VMDK files are generally quite large, making them suitable for hiding large files.

## Memory

-   Parse Windows / Linux / Mac OS X memory structures
-   Analyze processes and memory data
-   Follow clues and hints from the challenge to extract and analyze specific memory data from designated processes

## Challenges

-   Jarvis OJ - MISC - Forensics 2

## References

-   [Data Hiding Techniques](http://wooyun.jozxing.cc/static/drops/tips-12614.html)
