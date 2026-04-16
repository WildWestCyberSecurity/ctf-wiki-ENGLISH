# Import Table

When an executable uses code or data from an external DLL, the Windows loader needs to record all the functions and data to be imported and load the DLL into the virtual address space of the executable. The loader ensures that all DLLs required by the executable are loaded.

However, the executable cannot determine the location of imported functions in memory, so the Windows loader writes the information needed to locate imported functions into the IAT (Import Address Table) when loading the DLL. When a call to an imported function is encountered during execution, the IAT is used to determine the location of the imported function in memory.

The import table-related data includes `IMAGE_IMPORT_DESCRIPTOR`, `IMAGE_IMPORT_BY_NAME`, and the corresponding string data. The import table is a data section used to fix up and store the actual addresses of the corresponding functions after the DLL is loaded into memory.

## INT and IAT

`DataDirectory[1]` stores the RVA of the IMPORT TABLE (i.e., the import table). This RVA points to an array of `IMAGE_IMPORT_DESCRIPTOR` structures, which record the information needed by the PE file to import library files.

```c
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
  union {
    DWORD Characteristics;
    DWORD OriginalFirstThunk;       // RVA of the Import Name Table `INT`
  };
  DWORD TimeDateStamp;
  DWORD ForwarderChain;
  DWORD Name;                       // RVA of the library name string
  DWORD FirstThunk;                 // RVA of the Import Address Table `IAT`
} IMAGE_IMPORT_DESCRIPTOR;
```

The following is an explanation of the important members in this structure:

- **`OriginalFirstThunk` points to the INT (Import Name Table).**
- **`Name` points to the name of the library file to which the imported function belongs.**
- **`FirstThunk` points to the IAT (Import Address Table).**

`INT` and `IAT` are also collectively referred to as the dual-bridge structure. Each pointer in the `INT` array points to an `IMAGE_IMPORT_BY_NAME` structure, and the same is true for `IAT` in the file. The `IMAGE_IMPORT_BY_NAME` structure records the information needed for an imported function.

```c
typedef struct _IMAGE_IMPORT_BY_NAME {
  WORD Hint;                        // 
  BYTE Name[1];                     // Function name string
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME;
```

- **The `Hint` member represents the ordinal number of the function. Typically, each function in a DLL is assigned an ordinal number. A function can be located either by its name or by its ordinal number.**
- **The `Name[1]` member is a null-terminated ("\0") ANSI string representing the function name.**

Now let's look at the `IMAGE_IMPORT_DESCRIPTOR` structure array in the sample file:

```text
RVA       Data      Description               Value
----------------------------------------------------------
00006000  0000603C  Import Name Table RVA
00006004  00000000  Time Data Stamp
00006008  00000000  Forward Chain
0000600C  000064D8  Name RVA                  KERNEL32.dll
00006010  00006100  Import Address Table RVA  
----------------------------------------------------------
00006014  0000608C  Import Name Table RVA
00006018  00000000  Time Data Stamp
0000601C  00000000  Forward Chain
00006020  00006558  Name RVA                  msvcrt.dll
00006024  00006150  Import Address Table RVA
----------------------------------------------------------
00006028  00000000
0000602C  00000000
00006030  00000000
00006034  00000000
00006038  00000000
----------------------------------------------------------
```

Now let's look at the `INT` and `IAT` of the sample file:

![INT and IAT data of the sample file](figure/pe3-intiat2.png "Figure 7 - INT and IAT data section of the sample file")

As you can see, although the two point to different locations, the data they store is exactly the same. Why are two copies of the identical structure saved? To understand this, we first need to understand the roles of `INT` and `IAT` and the relationship between them. Let's first look at the relationship between `INT` and `IAT` in the file.

![Layout of INT and IAT in the file](figure/pe3-intiatinfile.png "Figure 8 - INT and IAT in the file")

Although they are different pointers, their contents are exactly the same, and they ultimately point to the same structure array. In other words, to locate a function in a library file, you can use either `INT` or `IAT`.
When a program is loaded into memory, the addresses of the imported functions are written into the `IAT` for easy reference. The process of updating the IAT address values is as follows:

1. Read the `Name` member of `IMAGE_IMPORT_DESCRIPTOR` to get the library name string "KERNEL32.dll"
2. Load the corresponding library -> `LoadLibrary["KERNEL32.dll"]`
3. Read the `OriginalFirstThunk` member of `IMAGE_IMPORT_DESCRIPTOR` to get the `INT` address
4. Read the values in the `INT` array to get the address of the corresponding `IMAGE_IMPORT_BY_NAME` structure
5. Read the `Hint` or `Name` member of `IMAGE_IMPORT_BY_NAME` to get the starting address of the corresponding function -> `GetProcAddress('DeleteCriticalSection')`
6. Read the `FirstThunk` member of `IMAGE_IMPORT_DESCRIPTOR` to get the `IAT` address
7. Write the function address obtained in step 5 into the corresponding entry in the `IAT` array
8. Repeat steps 4 - 7 until the `INT` ends (i.e., when a NULL is encountered)

Now let's look at the relationship between `INT` and `IAT` in memory.

![Layout of INT and IAT in memory](figure/pe3-intiatinmemory.png "Figure 8 - INT and IAT in memory")

**In memory, the `INT` can be used to find the function name or ordinal number, and the `IAT` can be used to find the actual address of the function's instruction code in the memory space.**
Let's examine the program's IAT in x32dbg:

![IAT values in x32dbg](figure/pe3-iatinx32dbg.png "Figure 9 - IAT in x32dbg")

At this point, all pointers in the `IAT` have been replaced with the actual addresses of the functions in memory.

## Bound Import

Bound import is a technique to improve PE loading speed. It only affects the loading process and does not affect the final loading result or the execution result of the PE. If a PE file has many functions to import, a certain amount of time will be spent on function importing during loading, which increases the PE loading time. **Bound import moves the IAT address fix-up work to before loading. This is done either manually by the user or by a dedicated binding tool; the bound import data is then declared in the PE file to tell the loader that it does not need to repeat the loading process.**

However, the base addresses of dynamic link libraries differ across different Windows systems, which can cause the bound addresses to be incorrect and prevent the program from running. This is easy to resolve. **Assuming that the IAT fix-ups before PE loading are all correct, the fix-up step is skipped at runtime. At the same time, the PE loading has an error-checking mechanism — if an error is detected, the PE loader will re-fix the IAT during loading.**

**In summary, when Windows loads the dynamic link libraries associated with the target PE file, it first checks whether these addresses are correct and valid, including checking whether the DLL version on the current system matches the version number described in the bound import structure. If the version does not match or the DLL needs to be relocated, the loader will traverse the array pointed to by OriginalFirstThunk to calculate new addresses and write the new addresses into the IAT.**
