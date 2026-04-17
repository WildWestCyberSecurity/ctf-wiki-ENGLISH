# Export Table

DLLs provide information such as exported function names, ordinals, and entry point addresses to the outside world through the export table. From the import perspective, when the Windows loader populates the IAT, it reads the addresses of imported functions from the DLL's export table. The export table typically exists in most DLLs, but it can also be found in a few EXE files.

Calls to exported functions in a DLL can be made either by function name or by the function's index in the export table. After the Windows loader loads the DLLs associated with a process into virtual address space, it traverses the DLL's virtual address space based on the names or ordinals registered in the import table for that DLL, and searches for the export table structure. This determines the starting address (VA) of the exported function in virtual address space, and writes this VA into the corresponding IAT entry.

## EAT

`DataDirectory[0]` stores the RVA of the EXPORT TABLE (i.e., the export table). This RVA points to an `IMAGE_EXPORT_DIRECTORY` structure. A PE file can contain at most one `IMAGE_EXPORT_DIRECTORY` structure. **However, a PE file can have multiple `IMAGE_IMPORT_DESCRIPTOR` structures, because a PE file can import from multiple libraries at once.**

Let's take a look at the `IMAGE_EXPORT_DIRECTORY` structure:

```c
typedef struct _IMAGE_EXPORT_DIRECTORY{
  DWORD    Characteristics;
  DWORD    TimeDateStamp;
  WORD     MajorVersion;
  WORD     MinorVersion;
  DWORD    Name;                     // Address of the library file name
  DWORD    Base;                     // Starting ordinal number for exported functions
  DWORD    NumberOfFunctions;        // Number of exported functions
  DWORD    NumberOfNames;            // Number of exported function names
  DWORD    AddressOfFunctions;       // Array of exported function addresses (number of elements = NumberOfFunctions)
  DWORD    AddressOfNames;           // Array of exported function name addresses (number of elements = NumberOfNames)
  DWORD    AddressOfNameOrdinals;    // Array of exported function ordinals (number of elements = NumberOfNames)
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;
```

The following is a detailed description of the members in this structure:

- **`Name` DWORD. The address stored in this member points to a null-terminated string that records the original file name of the file containing the export table.**
- **`Base` DWORD. The starting ordinal number for exported functions. The ordinal of an exported function = Base + Ordinals.**
- **`NumberOfFunctions` DWORD. The total number of exported functions.**
- **`NumberOfNames` DWORD. In the export table, some functions have defined names while others do not. This member records the count of all exported functions that have defined names. If this value is 0, it means none of the functions have defined names. `NumberOfNames` is always less than or equal to `NumberOfFunctions`.**
- **`AddressOfFunctions` DWORD. Points to the beginning of the exported function address array. The exported function address array holds addresses for `NumberOfFunctions` exported functions.**
- **`AddressOfNames` DWORD. Points to the beginning of the exported function name address array. Each element of the exported function name array points to the address of the corresponding exported function's name string.**
- **`AddressOfNameOrdinals` DWORD. Points to the beginning of the exported function ordinal array. It has a one-to-one correspondence with `AddressOfNames`. Each element of the exported function ordinal array points to the ordinal value of the corresponding exported function.**

Let's learn through a simple example. The example uses version.dll from the Windows system, located in the `C:\Windows\SysWOW64\` directory.
First, let's look at the `IMAGE_EXPORT_DIRECTORY` structure of the example file:

```text
// Example program IMAGE_EXPORT_DIRECTORY
RVA       Value      Description
----------------------------------------------------
00003630  00000000   Characteristicss
00003634  FDB2B236   Time Data Stamp
00003638  0000       Major Version
0000363A  0000       Minor Version
0000363C  00003702   Name RVA
00003640  00000001   Base
00003644  00000011   Number of Functions
00003648  00000011   Number of Names
0000364C  00003658   Address Table RVA
00003650  0000369C   Name Pointer Table RVA
00003654  000036E0   Ordinal Table RVA
----------------------------------------------------
```

Next, let's organize the arrays in the export table:

```text
RVA       Address   Name      Ordinal  Description
---------------------------------------------------------------------
00003658  000014F0  0000370E        0  GetFileVersionInfoA
0000365C  000022E0  00003722        1  GetFileVersionInfoByHandle
00003660  00001F40  0000373D        2  GetFileVersionInfoExA
00003664  00001570  00003753        3  GetFileVersionInfoExW
00003668  00001510  00003769        4  GetFileVersionInfoSizeA
0000366C  00001F60  00003781        5  GetFileVersionInfoSizeExA
00003670  00001590  0000379B        6  GetFileVersionInfoSizeExW
00003674  000015B0  000037B5        7  GetFileVersionInfoSizeW 
00003678  000015D0  000037CD        8  GetFileVersionInfoW
0000357C  00001F80  000037E1        9  VerFindFileA
00003680  00002470  000037EE       10  VerFindFileW
00003684  00001FA0  000037FB       11  VerInstallFileA
00003688  00002F40  0000380B       12  VerInstallFileW
0000368C  0000382C  0000381B       13  VerLanguageNameA
00003690  00003857  00003846       14  VerLanguageNameW
00003694  00001530  00003871       15  VerQueryValueA
00003698  00001550  00003880       16  VerQueryValueW
------------------------------------------------------------------------
```

The Address column corresponds to the actual address of the exported function when loaded into memory, the Name column corresponds to the RVA of the exported function name, and Ordinal is the ordinal number of the exported function.
Here is an additional image showing the string section of the export table, which stores the library file name and exported function names. This can be conveniently viewed using PEview:

![Strings in the Export Table](figure/pe4-eatstrings.png "Figure 10 - Export Table String Section")

The general process for obtaining a function address from the export table is as follows:

1. First, use the `AddressOfNames` member to locate the exported function name array;
2. Then, search for the specified function name by string comparison (strcmp). Once found, use its index as `name_index`;
3. Next, use the `AddressOfOrdinals` member to locate the exported function ordinal array;
4. Then, use `name_index` to locate the corresponding `ordinal` value in the exported function ordinal array;
5. Next, use the `AddressOfFunctions` member to locate the exported function address array, i.e., the `Export Address Table (EAT)`;
6. Finally, use `ordinal` as an index to locate the corresponding entry in the exported function address array and obtain the starting address of the specified function.

For the rare case of exported functions without names, the value obtained by subtracting Base from the Ordinal member is used as an index to locate the corresponding function address in the exported function address array.
