# PE File Format

## Introduction to PE

The full name of a PE file is Portable Executable, meaning a portable executable file. Common file types such as EXE, DLL, OCX, SYS, and COM are all PE files. PE files are program files on the Microsoft Windows operating system, and may be executed indirectly (e.g., DLL).
The layout of a 32-bit PE file is shown in the figure below:

```text
+-------------------------------+ \
|     MS-DOS MZ header          |  |
+-------------------------------+  |
| MS-DOS Real-Mode Stub program |  |
+-------------------------------+  |
|     PE Signature              |  | -> PE file header
+-------------------------------+  |
|     IMAGE_FILE_HEADER         |  |
+-------------------------------+  |
|     IMAGE_OPTIONAL_HEADER     |  |
+-------------------------------+ /
|     section header #1         | 
+-------------------------------+ 
|     section header #2 
+------------------------- 
: 
: 
 
+------------------------------+ 
|        section #1            | 
+------------------------------+ 
|        section #2 
+-------------------- 
: 
: 
```

Next, we will use a 32-bit PE file as a specimen to introduce the PE file format.

```c
// Example code test.c
#include <stdio.h>

int main(){
  printf("Hello, PE!\n");

  return 0;
}
```

**The file is compiled using the `TDM-GCC 4.9.2 32-bit Release` option in `Devcpp` to generate `test.exe`, which serves as the example file.**

### Common Terms and Their Meanings

- **`Image file`: Since PE files usually need to be loaded into memory for execution, serving as an image in memory, PE files are also called image files.**
- **`RVA`: Relative Virtual Address, the offset of the image file in virtual memory relative to the load base address.**
- **`VA`: Virtual Address, the address of the image file in virtual memory.**
- **`FOA`: File Offset Address, the offset of the image file on disk relative to the beginning of the file.**

Because regardless of whether data is on disk or in virtual memory, the relative offset of data with respect to its containing section is fixed, this allows conversion between RVA and FOA, i.e., `RVA - Section RVA = FOA - Section FOA`.

Suppose a piece of data belonging to the .data section has an RVA of 0x3100, and the .data section's Section RVA is 0x3000, then the data's relative offset from the .data section is 0x100. If the .data section's Section FOA is 0x1C00, then the data's FOA on disk is 0x1D00. The complete formula is: `FOA = Section FOA + (RVA - Section RVA)`. If the image file's load base address is 0x40000000, then the data's VA is 0x40003100.

## PE File Header

The very beginning of a PE file is the PE file header, which consists of the `MS-DOS file header` and the `IMAGE_NT_HEADERS` structure.

### MS-DOS File Header

The `MS-DOS file header` contains two parts: `IMAGE_DOS_HEADER` and `DOS Stub`.

The definition of the `IMAGE_DOS_HEADER` structure is as follows:

```c
typedef struct _IMAGE_DOS_HEADER
{
     WORD e_magic;              // "MZ"
     WORD e_cblp;
     WORD e_cp;
     WORD e_crlc;
     WORD e_cparhdr;
     WORD e_minalloc;
     WORD e_maxalloc;
     WORD e_ss;
     WORD e_sp;
     WORD e_csum;
     WORD e_ip;
     WORD e_cs;
     WORD e_lfarlc;
     WORD e_ovno;
     WORD e_res[4];
     WORD e_oemid;
     WORD e_oeminfo;
     WORD e_res2[10];
     LONG e_lfanew;             // Offset of the NT header relative to the beginning of the file
} IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;
```

The `IMAGE_DOS_HEADER` structure has 2 important members:
- **`e_magic`: WORD. DOS signature "4D5A", i.e., ASCII value "MZ". All PE files begin with the DOS signature.**
- **`e_lfanew`: WORD. Offset of `IMAGE_NT_HEADER` relative to the beginning of the file.**

The `IMAGE_DOS_HEADER` of the example program is shown in Figure 2:

![IMAGE_DOS_HEADER](figure/pe2-imagedosheader.png "Figure 2 - IMAGE_DOS_HEADER")

Immediately following the `IMAGE_DOS_HEADER` structure is the `DOS Stub`. Its function is simple: when the system is in an MS-DOS environment, it outputs `This program cannot be run in DOS mode.` and exits the program, indicating that the program cannot run in the MS-DOS environment. This makes all PE files compatible with the MS-DOS environment. By leveraging this feature, it is possible to create a program that runs in both MS-DOS and Windows environments, executing 16-bit MS-DOS code in MS-DOS and 32-bit Windows code in Windows.

The `DOS Stub` of the example program is shown in Figure 3:

![DOS Stub](figure/pe2-imagedosstub.png "Figure 3 - DOS Stub")

### IMAGE_NT_HEADERS

The `IMAGE_NT_HEADERS` structure, commonly known as the NT header, follows immediately after the `DOS Stub`. Its definition is as follows:

```c
typedef struct _IMAGE_NT_HEADERS {
  DWORD                   Signature;         /* +0000h PE Signature */
  IMAGE_FILE_HEADER       FileHeader;        /* +0004h PE Standard Header */
  IMAGE_OPTIONAL_HEADER32 OptionalHeader;    /* +0018h PE Optional Header */
} IMAGE_NT_HEADERS32, *PIMAGE_NT_HEADERS32;
```

The `IMAGE_NT_HEADERS` of the example program is shown in Figure 4:

![NT Header](figure/pe2-imagentheader.png "Figure 4 - IMAGE_NT_HEADERS")

Next, let's discuss the NT header in detail.

#### PE Signature

The first member of the NT header is the `PE Signature`, which is a 4-byte ASCII string `PE\0\0`, used to indicate that the current file is a PE format image file. Its position can be determined by the value of the `e_lfanew` member of `IMAGE_DOS_HEADER`.

#### IMAGE_FILE_HEADER

Immediately following the `PE Signature` is the `IMAGE_FILE_HEADER` structure, also known as the `COFF header (Common Object File Format header)`. Its definition is as follows:

```c
typedef struct _IMAGE_FILE_HEADER {
  WORD  Machine;                    /* +0004h Target machine type */
  WORD  NumberOfSections;           /* +0006h Number of sections in the PE */
  DWORD TimeDateStamp;              /* +0008h Timestamp */
  DWORD PointerToSymbolTable;       /* +000ch Pointer to the symbol table */
  DWORD NumberOfSymbols;            /* +0010h Number of symbols in the symbol table */
  WORD  SizeOfOptionalHeader;       /* +0012h Size of the optional header */
  WORD  Characteristics;            /* +0014h File attribute flags */
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
```

The following is a description of each field:

- **`Machine`: WORD. Specifies the CPU type. For details on supported CPU types, refer to [Microsoft PE Format COFF File Header Machine Types](https://docs.microsoft.com/zh-cn/windows/win32/debug/pe-format?redirectedfrom=MSDN#machine-types).**
- **`NumberOfSections`: WORD. The number of sections present in the file. PE files categorize and store code, data, and resources in different sections based on their attributes.**
- `TimeDateStamp`: DWORD. The low 32 bits represent the number of seconds elapsed from January 1, 1970, 00:00 to the time the file was created.
- `PointerToSymbolTable`: DWORD. The file offset of the symbol table. If no symbol table exists, the value is 0.
- `NumberOfSymbols`: DWORD. This field indicates the number of symbols in the symbol table. Since the string table immediately follows the symbol table, this value can be used to locate the string table.
- **`SizeOfOptionalHeader`: WORD. Indicates the size of the optional header. The default is 0x00E0 on 32-bit machines and 0x00F0 on 64-bit machines.**
- **`Characteristics`: WORD. Used to identify file attributes, combined using bit OR.** The following are some defined file attribute flags:

```c
// File attribute flags
#define IMAGE_FILE_RELOCS_STRIPPED          0x0001    // File does not contain relocation info and can only be loaded at its preferred base address. If the preferred base is unavailable, the loader reports an error
#define IMAGE_FILE_EXECUTABLE_IMAGE         0x0002    // File is executable. If this bit is not set, it indicates a linker error
#define IMAGE_FILE_LINE_NUMS_STRIPPED       0x0004    // Line number information is not present
#define IMAGE_FILE_LOCAL_SYMS_STRIPPED      0x0008    // Symbol information is not present
#define IMAGE_FILE_AGGRESSIVE_WS_TRIM       0x0010    // Deprecated
#define IMAGE_FILE_LARGE_ADDRESS_AWARE      0x0020    // Application can handle addresses larger than 2GB
#define IMAGE_FILE_BYTES_REVERSED_LO        0x0080    // Little-endian. Deprecated
#define IMAGE_FILE_32BIT_MACHINE            0x0100    // Based on 32-bit architecture
#define IMAGE_FILE_DEBUG_STRIPPED           0x0200    // Debug information is not present
#define IMAGE_FILE_REMOVABLE_RUN_FROM_SWAP  0x0400    // If the image file is on removable media, fully load and copy to the swap file
#define IMAGE_FILE_NET_RUN_FROM_SWAP        0x0800    // If the image file is on network media, fully load and copy to the swap file
#define IMAGE_FILE_SYSTEM                   0x1000    // Image file is a system file
#define IMAGE_FILE_DLL                      0x2000    // Image file is a dynamic-link library file
#define IMAGE_FILE_UP_SYSTEM_ONLY           0x4000    // File can only run on a uniprocessor machine
#define IMAGE_FILE_BYTES_REVERSED_HI        0x8000    // Big-endian (deprecated)
```

The `IMAGE_FILE_HEADER` of the example program is as follows:

```text
// Example program IMAGE_FILE_HEADER
RVA       Value      Description
----------------------------------------------------
00000084  014C       Machine type
00000086  000F       Number of sections
00000088  5D88E2A6   Timestamp
0000008c  00012C00   Symbol table offset
00000090  000004E4   Number of symbols
00000094  00E0       Optional header size
00000096  0107       File attributes
                     0001  IMAGE_FILE_RELOCS_STRIPPED
                     0002  IMAGE_FILE_EXECUTABLE_IMAGE
                     0004  IMAGE_FILE_LINE_NUMS_STRIPPED
                     0100  IMAGE_FILE_32BIT_MACHINE
```

#### IMAGE_OPTIONAL_HEADER

The reason `IMAGE_OPTIONAL_HEADER` is called the optional header is that for object files, it serves no purpose and only increases the file size; however, for image files, it provides information essential for loading. Its definition is as follows:

```c
typedef struct _IMAGE_OPTIONAL_HEADER {
  WORD                 Magic;                            /* +0018h Magic number */
  BYTE                 MajorLinkerVersion;               /* +001ah Linker major version number */
  BYTE                 MinorLinkerVersion;               /* +001bh Linker minor version number */
  DWORD                SizeOfCode;                       /* +001ch Total size of all code sections */
  DWORD                SizeOfInitializedData;            /* +0020h Total size of all initialized data sections */
  DWORD                SizeOfUninitializedData;          /* +0024h Total size of all uninitialized data sections */
  DWORD                AddressOfEntryPoint;              /* +0028h Entry point RVA */
  DWORD                BaseOfCode;                       /* +002ch Code section start RVA */
  DWORD                BaseOfData;                       /* +0030h Data section start RVA */
  DWORD                ImageBase;                        /* +0034h Preferred load address of the image file */
  DWORD                SectionAlignment;                 /* +0038h Section alignment granularity in memory */
  DWORD                FileAlignment;                    /* +003ch Section alignment granularity in the file */
  WORD                 MajorOperatingSystemVersion;      /* +0040h OS major version number */
  WORD                 MinorOperatingSystemVersion;      /* +0042h OS minor version number */
  WORD                 MajorImageVersion;                /* +0044h Image major version number */
  WORD                 MinorImageVersion;                /* +0046h Image minor version number */
  WORD                 MajorSubsystemVersion;            /* +0048h Subsystem major version number */
  WORD                 MinorSubsystemVersion;            /* +004ah Subsystem minor version number */
  DWORD                Win32VersionValue;                /* +004ch Reserved. Set to 0 */
  DWORD                SizeOfImage;                      /* +0050h Size of the image file in memory */
  DWORD                SizeOfHeaders;                    /* +0054h Total size of all headers + section table */
  DWORD                CheckSum;                         /* +0058h Image file checksum */
  WORD                 Subsystem;                        /* +005ch Subsystem required to run the image */
  WORD                 DllCharacteristics;               /* +005eh DLL attributes of the image file */
  DWORD                SizeOfStackReserve;               /* +0060h Stack size reserved at initialization */
  DWORD                SizeOfStackCommit;                /* +0064h Stack size actually committed at initialization */
  DWORD                SizeOfHeapReserve;                /* +0068h Heap size reserved at initialization */
  DWORD                SizeOfHeapCommit;                 /* +006ch Heap size actually committed at initialization */
  DWORD                LoaderFlags;                      /* +0070h Deprecated */
  DWORD                NumberOfRvaAndSizes;              /* +0074h Number of data directory entries */
  IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];     /* +0078h Pointer to the first IMAGE_DATA_DIRECTORY structure in the data directory */
} IMAGE_OPTIONAL_HEADER32, *PIMAGE_OPTIONAL_HEADER32;
```

- **`Magic`: WORD. Specifies the type of the image file. `0x0107h` indicates a ROM image; `0x010B` indicates PE32; `0x020B` indicates PE32+, i.e., a 64-bit PE file.**
- `MajorLinkerVersion`: BYTE. Specifies the linker major version number.
- `MinorLinkerVersion`: BYTE. Specifies the linker minor version number.
- `SizeOfCode`: DWORD. Total size of all sections containing code. **The size here refers to the size after file alignment. Whether a section contains code is determined by checking if the section attributes include the `IMAGE_SCN_CNT_CODE` flag.**
- `SizeOfInitializedData`: DWORD. Total size of all sections containing initialized data.
- `SizeOfUninitializedData`: DWORD. Total size of all sections containing uninitialized data.
- **`AddressOfEntryPoint`: DWORD. The offset of the pointer to the entry point function relative to the image file's load base address. For executable files, this is the startup address; for device drivers, this is the address of the initialization function; the entry point function is optional for DLL files — if no entry point exists, this member must be set to 0.**
- `BaseOfCode`: DWORD. The RVA of the code section, i.e., the offset of the start of the code section relative to the image file's load base address. Typically, the code section follows immediately after the PE header, with the section name ".text".
- `BaseOfData`: DWORD. The RVA of the data section, i.e., the offset of the start of the data section relative to the image file's load base address. Typically, the data section is located at the end of the file, with the section name ".data".
- **`ImageBase`: DWORD. The preferred load address of the image file; the value must be a multiple of 64KB.** The default value for applications is 0x00400000; the default for DLLs is 0x10000000. **When a program uses multiple DLL files, the PE loader will adjust the DLL load addresses to ensure all DLL files can be loaded correctly.**
- **`SectionAlignment`: DWORD. Section alignment granularity in memory. The value of this member must be no less than the value of the `FileAlignment` member. The default value equals the system's page size.**
- **`FileAlignment`: DWORD. Alignment granularity for raw data in the image file. The value must be a power of 2 in the range of 512 to 64K. The default value is 512, but if the `SectionAlignment` member value is less than the system page size, then `FileAlignment` and `SectionAlignment` must have the same value.**
- `MajorOperatingSystemVersion`: WORD. OS major version number.
- `MinorOperatingSystemVersion`: WORD. OS minor version number.
- `MajorImageVersion`: WORD. Image major version number.
- `MinorImageVersion`: WORD. Image minor version number.
- `MajorSubsystemVersion`: WORD. Subsystem major version number.
- `MinorSubsystemVersion`: WORD. Subsystem minor version number.
- `Win32VersionValue`: DWORD. Reserved. Set to 0.
- **`SizeOfImage`: DWORD. The size of the image file in virtual memory. The value must be a multiple of `SectionAlignment`.**
- **`SizeOfHeaders`: DWORD. The total size of the PE file header and all section headers, aligned to `FileAlignment`. The first section begins at the file offset `SizeOfHeaders`.**
- `CheckSum`: DWORD. The checksum of the image file. Files that need to be verified at load time include all drivers, any DLLs loaded at startup, and any DLLs loaded into critical system processes.
- **`Subsystem`: WORD. The subsystem required to run the image file. The defined subsystem flags are as follows:**

```c
// Subsystem flags
#define IMAGE_SUBSYSTEM_UNKNOWN                      0  // Unknown subsystem
#define IMAGE_SUBSYSTEM_NATIVE                       1  // No subsystem required. Device drivers and native system processes
#define IMAGE_SUBSYSTEM_WINDOWS_GUI                  2  // Windows graphical user interface (GUI) subsystem
#define IMAGE_SUBSYSTEM_WINDOWS_CUI                  3  // Windows character-mode user interface (CUI) subsystem
#define IMAGE_SUBSYSTEM_OS2_CUI                      5  // OS/2 CUI subsystem
#define IMAGE_SUBSYSTEM_POSIX_CUI                    7  // POSIX CUI subsystem
#define IMAGE_SUBSYSTEM_WINDOWS_CE_GUI               9  // Windows CE system
#define IMAGE_SUBSYSTEM_EFI_APPLICATION             10  // Extensible Firmware Interface (EFI) application
#define IMAGE_SUBSYSTEM_EFI_BOOT_SERVEICE_DRIVER    11  // EFI driver with boot services
#define IMAGE_SUBSYSTEM_EFI_RUNTIME_DRIVER          12  // EFI driver with runtime services
#define IMAGE_SUBSYSTEM_EFI_ROM                     13  // EFI ROM image
#define IMAGE_SUBSYSTEM_XBOX                        14  // XBOX system
#define IMAGE_SUBSYSTEM_WINDOWS_BOOT_APPLICATION    16  // Boot application
```

- **`DllCharacteristics`: WORD. DLL attributes of the image file, combined using bit OR. The meanings of each flag bit are as follows:**

```c
// DLL attribute flags
// 0x0001 0x0002 0x0004 0x0008 are reserved; values must be 0.
#define IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE             0x0040  // DLL can be relocated at load time
#define IMAGE_DLLCHARACTERISTICS_FORCE_INTEGRITY          0x0080  // Force code integrity checks
#define IMAGE_DLLCHARACTERISTICS_NX_COMPAT                0x0100  // Image is compatible with Data Execution Prevention (DEP)
#define IMAGE_DLLCHARACTERISTICS_NO_ISOLATION             0x0200  // Image can be isolated but should not be
#define IMAGE_DLLCHARACTERISTICS_NO_SEH                   0x0400  // Image does not use Structured Exception Handling (SEH)
#define IMAGE_DLLCHARACTERISTICS_NO_BIND                  0x0800  // Do not bind the image
//#define IMAGE_DLLCHARACTERISTICS_APPCONTAINER           0x1000  // Reserved in 32-bit; in 64-bit indicates the image must execute within an AppContainer
#define IMAGE_DLLCHARACTERISTICS_WDM_DRIVER               0x2000  // WDM driver
//#define IMAGE_DLLCHARACTERISTICS_GUARD_CF               0x4000  // Reserved in 32-bit; in 64-bit indicates the image supports Control Flow Guard
#define IMAGE_DLLCHARACTERISTICS_TERMINAL_SERVER_AWARE    0x8000  // Image is Terminal Server aware
```

- `SizeOfStackReserve`: DWORD. The size of stack memory reserved at initialization; the default value is 1MB. Specifically, it is the size of virtual memory reserved for the stack at initialization, but not all reserved virtual memory can be directly used as the stack. The actual stack size committed at initialization is specified by the `SizeOfStackCommit` member.
- `SizeOfStackCommit`: DWORD. The actual stack memory size committed at initialization.
- `SizeOfHeapReserve`: DWORD. The size of heap memory reserved at initialization; the default value is 1MB. Every process has at least one default process heap, which is created when the process starts and is not deleted during the process's lifetime.
- `SizeOfHeapCommit`: DWORD. The actual heap memory size committed at initialization; the default size is 1 page. The initial reserved heap memory size and the actual committed heap memory size can be specified through the linker's "-heap" parameter.
- `LoaderFlags`: This member is deprecated.
- **`NumberOfRvaAndSizes`: DWORD. The number of data directory entries. Typically 0x00000010, i.e., 16.**
- **`DataDirectory`: Structure. An array composed of `IMAGE_DATA_DIRECTORY` structures, where each element has a defined value. The structure definition is as follows:**

```c
typedef struct _IMAGE_DATA_DIRECTORY {
  DWORD VirtualAddress;      /* RVA of the data directory */
  DWORD Size;                /* Size of the data directory */
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

The array entries are as follows:

```c
// Data directory
DataDirectory[0] =   EXPORT Directory           // Export table RVA and size
DataDirectory[1] =   IMPORT Directory           // Import table RVA and size
DataDirectory[2] =   RESOURCE Directory         // Resource table RVA and size
DataDirectory[3] =   EXCEPTION Directory        // Exception table RVA and size
DataDirectory[4] =   CERTIFICATE Directory      // Certificate table FOA and size
DataDirectory[5] =   BASE RELOCATION Directory  // Base relocation table RVA and size
DataDirectory[6] =   DEBUG Directory            // Debug information RVA and size
DataDirectory[7] =   ARCH DATA Directory        // Architecture-specific information RVA and size
DataDirectory[8] =   GLOBALPTR Directory        // Global pointer register RVA
DataDirectory[9] =   TLS Directory              // Thread Local Storage table RVA and size
DataDirectory[10] =  LOAD CONFIG Directory      // Load configuration table RVA and size
DataDirectory[11] =  BOUND IMPORT Directory     // Bound import table RVA and size
DataDirectory[12] =  `IAT` Directory              // Import Address Table RVA and size
DataDirectory[13] =  DELAY IMPORT Directory     // Delay import descriptor RVA and size
DataDirectory[14] =  CLR Directory              // CLR data RVA and size
DataDirectory[15] =  Reserverd                  // Reserved
```

The `IMAGE_OPTIONAL_HEADER` of the example program is shown below:
![Optional Header Upper](figure/pe2-imageoptionalheader1.png "Figure 5 - IMAGE_OPTIONAL_HEADER Structure")
![Optional Header Lower](figure/pe2-imageoptionalheader2.png "Figure 6 - DataDirectory Member")

## PE Data Body

The PE data body includes the `Section Header` and all sections.

### Section Header

Immediately following the optional header is the `Section Header`, also known as the section table. The attributes of all sections in a PE file are defined in the section table. The section table consists of a series of `IMAGE_SECTION_HEADER` structures, each 40 bytes in size. Each structure describes the information of one section, defined as follows:

```c
typedef struct _IMAGE_SECTION_HEADER {
  BYTE  Name[IMAGE_SIZEOF_SHORT_NAME];    /* Section name */
  union {
    DWORD PhysicalAddress;                /* Physical address */
    DWORD VirtualSize;                    /* Section size in virtual memory */
  } Misc;
  DWORD VirtualAddress;                   /* Section RVA in virtual memory */
  DWORD SizeOfRawData;                    /* Section size on disk */
  DWORD PointerToRawData;                 /* Section FOA on disk */
  DWORD PointerToRelocations;             /* Pointer to the relocation table */
  DWORD PointerToLinenumbers;             /* Pointer to the line number table */
  WORD  NumberOfRelocations;              /* Number of relocation entries */
  WORD  NumberOfLinenumbers;              /* Number of line numbers */
  DWORD Characteristics;                  /* Section attributes */
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
```

- `Name`: Section name string. Maximum length is 8 bytes.
- `Misc`
  - `PhysicalAddress`: DWORD. File address.
  - `VirtualSize`: DWORD. The size of the section in virtual memory.
- `VirtualAddress`: DWORD. The section's RVA in virtual memory.
- `SizeOfRawData`: DWORD. For image files, it represents the size of initialized data on disk, and the value must be a multiple of `FileAlignment`; for object files, it represents the size of the section.
- `PointerToRawData`: DWORD. The FOA of the section's beginning on disk. The value must be a multiple of `FileAlignment`.
- `PointerToRelocations`: DWORD. Used in object files; pointer to the relocation table.
- `PointerToLinenumbers`: DWORD. Position of line number information (for debugging). Set to 0 if there is no line number information; also, since the use of COFF debug information is discouraged, it should be set to 0 in image files.
- `NumberOfRelocations`: WORD. The number of relocation entries; set to 0 in image files.
- `NumberOfLinenumbers`: WORD. The number of line numbers (for debugging). Since the use of COFF debug information is discouraged, it should be set to 0 in image files.
- **`Characteristics`: DWORD. Section attributes, combined using bit OR. The meanings of each flag bit are as follows:**

```c
// Section attributes
#define IMAGE_SCN_CNT_CODE                0x00000020  // Section contains code
#define IMAGE_SCN_CNT_INITIALIZED_DATA    0x00000040  // Section contains initialized data
#define IMAGE_SCN_CNT_UNINITIALIZED_DATA  0x00000080  // Section contains uninitialized data
#define IMAGE_SCN_ALIGN_1BYTES            0x00100000  // 1-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_2BYTES            0x00200000  // 2-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_4BYTES            0x00300000  // 4-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_8BYTES            0x00400000  // 8-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_16BYTES           0x00500000  // 16-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_32BYTES           0x00600000  // 32-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_64BYTES           0x00700000  // 64-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_128BYTES          0x00800000  // 128-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_256BYTES          0x00900000  // 256-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_512BYTES          0x00A00000  // 512-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_1024BYTES         0x00B00000  // 1024-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_2048BYTES         0x00C00000  // 2048-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_4096BYTES         0x00D00000  // 4096-byte alignment. Used only for object files
#define IMAGE_SCN_ALIGN_8192BYTES         0x00E00000  // 8192-byte alignment. Used only for object files
#define IMAGE_SCN_LNK_NRELOC_OVFL         0x01000000  // Section contains extended relocation entries
#define IMAGE_SCN_MEM_DISCARDABLE         0x02000000  // Section can be discarded as needed, e.g., .reloc is discarded after the process starts
#define IMAGE_SCN_MEM_NOT_CACHED          0x04000000  // Section will not be cached
#define IMAGE_SCN_MEM_NOT_PAGED           0x08000000  // Section is not pageable
#define IMAGE_SCN_MEM_SHARED              0x10000000  // Section can be shared among different processes
#define IMAGE_SCN_MEM_EXECUTE             0x20000000  // Section can be executed as code
#define IMAGE_SCN_MEM_READ                0x40000000  // Section is readable
#define IMAGE_SCN_MEM_WRITE               0x80000000  // Section is writable
```

The section headers of the example file are as follows:

```text
No.  Name    VirtualSize  VirtualOffset  RawSize   RawOffset  Characteristics
--------------------------------------------------------------------------
01   .text   00001670     00001000       00001800  00000400   60500020  R-X  Contains executable code
02   .data   0000002C     00003000       00000200  00001C00   C0300040  RW-  Contains initialized data
03   .rdata  00000168     00004000       00000600  00001E00   40300040  R--  Contains initialized data
04   .bss    00000450     00005000       00000000  00000000   C0700080  RW-  Contains uninitialized data
05   .idata  00000564     00006000       00000600  00002400   C0300040  RW-  Contains initialized data
06   .CRT    00000034     00007000       00000200  00002A00   C0300040  RW-  Contains initialized data
07   .tls    00000020     00008000       00000200  00002C00   C0300040  RW-  Contains initialized data
08   /4      000002D8     00009000       00000400  00002E00   42400040  R--  Contains initialized data
09   /19     0000A6D5     0000A000       0000A800  00003200   42100040  R--  Contains initialized data
0A   /31     0000199E     00015000       00001A00  0000DA00   42100040  R--  Contains initialized data
0B   /45     000018F3     00017000       00001A00  0000F400   42100040  R--  Contains initialized data
0C   /57     00000780     00019000       00000800  00010E00   42300040  R--  Contains initialized data
0D   /70     000002F2     0001A000       00000400  00011600   42100040  R--  Contains initialized data
0E   /81     00000D1E     0001B000       00000E00  00012800   42100040  R--  Contains initialized data
0F   /92     00000230     0001C000       00000400  00012C00   42100040  R--  Contains initialized data
```

### Sections

Immediately following the `Section Header` are the individual sections. A PE file generally requires at least two sections: a code section .text for storing executable data, and a data section .data for storing data. The purpose of a section can be guessed from its name, but the section name is not the determining factor for the section's purpose — it is only a reference. For example, the code section's name can be changed to .data without affecting program execution. Here are the common sections and their purposes:

```text
 .text  Default code section. Used to store executable code.
 .data  Default read/write data section. Used to store initialized global variables and static variables.
.rdata  Default read-only data section.
.idata  Used to store import table information. Contains the IAT, INT, imported function names, and imported DLL names, etc.
.edata  Used to store export table information.
 .rsrc  Used to store resource table information.
  .bss  Used to store uninitialized data.
  .tls  Used to store TLS (Thread Local Storage) information.
.reloc  Used to store relocation table information.
```

Among these, some sections require special attention, such as the .idata section which stores library import-related data, or the .tls section related to Thread Local Storage, etc. Analyzing these important sections is the main focus of subsequent study.
