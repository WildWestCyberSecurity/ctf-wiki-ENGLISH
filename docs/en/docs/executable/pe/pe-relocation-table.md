# Base Relocation Table

When the linker generates a PE file, it assumes the PE file will be loaded at the default base address during execution, so it writes the absolute addresses of code and data into the PE file. If the PE file is indeed loaded at the default base address, no relocation is needed. However, if the PE file is loaded at a different location, all the absolute addresses in the file become invalid, because the load base address has changed and the actual addresses of code and data have changed accordingly. In this case, relocation is needed to fix the absolute addresses so they all point to the correct locations.

For EXE files, each file uses an independent virtual address space during execution, meaning it can always be loaded at the default base address, so relocation information is not needed. However, multiple DLLs may exist in the same virtual address space, and some DLLs may face a situation where the default base address is already occupied, so DLLs need relocation information.

## Relocation Structure

In a PE file, all addresses that may need relocation are placed in an array, namely the base relocation table. If the load address changes, all addresses in the array will be corrected. The base relocation table is located within the .reloc section, but the correct way to find it is through `DataDirectory[5]`, i.e., the BASE RELOCATION TABLE entry.

The organization of base relocation data uses a page-splitting method, where relocation data for different pages is stored separately. The relocation data for each page forms a relocation block, and all relocation blocks together form the relocation table. Each relocation block holds relocation information for a 4KB-sized page, and the size of each relocation block must be DWORD-aligned. A relocation block starts with an `IMAGE_BASE_RELOCATION` structure, defined as follows:

```c
typedef struct _IMAGE_BASE_RELOCATION {
    DWORD    VirtualAddress;  // RVA of the relocation page
    DWORD    SizeOfBlock;     // Size of the relocation block
     WORD    TypeOffset;      // Type and offset of the relocation entry

} _IMAGE_BASE_RELOCATION;
typedef IMAGE_BASE_RELOCATION UNALIGNED * PIMAGE_BASE_RELOCATION;
```

The following is a detailed explanation of each member in the structure:

- **VirtualAddress: RVA of the relocation page. The sum of the image load base address plus the page RVA is used as the addend, and adding the offset corresponding to the relocation entry gives the actual VA in memory. A VirtualAddress field is also appended at the end of the last relocation block as a termination marker.**
- **SizeOfBlock: Size of the base relocation block. This includes the size of VirtualAddress, SizeOfBlock, and the subsequent TypeOffset entries.**
- **TypeOffset:** An array. Each element in the array is 2 bytes in size, i.e., 16 bits.
  - **type: The upper 4 bits indicate the relocation type.**
  - **offset: The lower 12 bits indicate the offset of the relocation data position relative to the page RVA. Adding this to VirtualAddress gives the pointer to the relocation data to be modified, and adding the image load base address gives the corrected pointer.**

## Relocation Process

**Use the relocation table to locate the addresses that need to be modified.** For example, in me.dll, the beginning of the relocation table is as follows:

```text
RVA       Data      Description
00005000  00001000  Page RVA          // page RVA = 0x1000
00005004  00000118  Relocation block size
00005008      3013  Type|Offset       //   offset = 0x013
...
```

From 0x1000 + 0x013, we calculate that the data to be relocated is at file offset 0x1013. Adding the default ImageBase gives 0x10001013. As shown below:

```x86asm
.text:10001012 68 9C 20 00 10        push 1000209C
```

That is, the value 1000209C at file offset 0x1013 may need relocation.

**Modify the data to be relocated.** After the program runs, me.dll is loaded at 0x633C0000:

![DLL load location](figure/pe5-relocdll.png)

Calculate the corrected value for the data to be relocated, then write the corrected value to the relocation address:

```
Calculate the address of the data to be relocated:
(0x10001013 - DefaultImageBase) + ImageBase 
i.e. (0x10001013 - 0x10000000) + 0x633C0000 => 0x633C1013

Calculate the corrected value of the data to be relocated:
(0x1000209C - DefaultImageBase) + ImageBase 
i.e. (0x1000209C - 0x10000000) + 0x633C0000 => 0x633C209C

Finally: *0x633C1013 = 0x633C209C
```

Check the actual value in memory:

![Address after relocation](figure/pe5-relocdata.png)

> A question to ponder: when would an EXE need relocation?
