# Dynamic Sections

## .interp

This section contains the interpreter corresponding to the program.

Executable files that participate in dynamic linking will have a program header element of type PT_INTERP, which is used to load segments in the program. During the exec (BA_OS) process, the system extracts the interpreter's path from this section and creates the initial program image based on the segments of the interpreter file. In other words, the system does not use the image of the given executable file, but first constructs an independent memory image for the interpreter. Therefore, the interpreter needs to obtain control from the system and then provide an execution environment for the application.

The interpreter may obtain control in two ways.

1. It can receive a file descriptor pointing to the file header in order to read the executable file. It can use this file descriptor to read and map the segments of the executable file into memory.

2. Sometimes, depending on the executable file format, the system may not give the file descriptor to the interpreter, but instead directly loads the executable file into memory.

The interpreter itself may not need another interpreter. The interpreter itself may be a shared object or an executable file.

- A shared object (in normal cases) is loaded as position-independent. That is, for different processes, its address will differ. The system uses mmap (KE_OS) and some related operations to create the contents of the dynamic segment. Therefore, the address of a shared object usually does not conflict with the original address of the executable file.
- An executable file is generally loaded at a fixed address. The system creates the corresponding segments through the virtual addresses in the program header table. Therefore, the virtual address of an executable file's interpreter may conflict with the first executable file. The interpreter is responsible for resolving the corresponding conflicts.

## .dynamic

If an object file participates in the dynamic linking process, its program header table will contain an element of type PT_DYNAMIC. This segment contains the .dynamic section. ELF uses the _DYNAMIC symbol to mark this section. Its structure is as follows:

```c
/* Dynamic section entry.  */
typedef struct
{
    Elf32_Sword d_tag; /* Dynamic entry type */
    union
    {
        Elf32_Word d_val; /* Integer value */
        Elf32_Addr d_ptr; /* Address value */
    } d_un;
} Elf32_Dyn;
extern Elf32_Dyn_DYNAMIC[];
```

Here, the value of d_tag determines how d_un should be interpreted.

-   d_val
    -   This field represents an integer value, which can have multiple meanings.
-   d_ptr
    -   This field represents the virtual address of the program. As mentioned earlier, the virtual address of a file may not match the virtual address in memory during execution. When parsing addresses in the dynamic section, the dynamic linker calculates the actual address based on the original file value and the memory base address. For consistency, the file does not contain relocation entries to "correct" addresses in the dynamic structure.

As can be seen, this section is actually composed of several key-value pairs.

**The following table summarizes the d_tag requirements in executable files and shared object files.** If a tag is marked as "mandatory", then for a TIS ELF conforming file, its dynamic linking array must contain the corresponding entry type. Similarly, "optional" means it may or may not be present.

| Name                  | Value                  | d_un        | Executable | Shared Object | Description                                                  |
| --------------------- | ---------------------- | ----------- | ---------- | ------------- | ------------------------------------------------------------ |
| DT_NULL               | 0                      | ignored     | mandatory  | mandatory     | Marks the end of the _DYNAMIC array.                         |
| DT_NEEDED             | 1                      | d_val       | optional   | optional      | Contains the string table offset of a NULL-terminated string that gives the name of a needed library. The index used is a subscript into DT_STRTAB. The dynamic array can contain many entries of this type. The relative order of entries of this type is significant, but their order relative to other tags does not matter. The corresponding section is .gnu.version_r. |
| DT_PLTRELSZ           | 2                      | d_val       | optional   | optional      | Gives the total size of relocation entries associated with the procedure linkage table. If a DT_JMPREL type entry exists, then DT_PLTRELSZ must also exist. |
| DT_PLTGOT             | 3                      | d_ptr       | optional   | optional      | Gives the address associated with the procedure linkage table or the global offset table. The corresponding section is .got.plt. |
| DT_HASH               | 4                      | d_ptr       | mandatory  | mandatory     | This type of entry contains the address of the symbol hash table. This hash table refers to the symbol table referenced by DT_SYMTAB. |
| DT_STRTAB             | 5                      | d_ptr       | mandatory  | mandatory     | This type of entry contains the address of the dynamic string table. Symbol names, library names, and other strings are all contained in this table. The corresponding section name should be .dynstr. |
| DT_SYMTAB             | 6                      | d_ptr       | mandatory  | mandatory     | This type of entry contains the address of the dynamic symbol table. For 32-bit files, the entries in this symbol table are of type Elf32_Sym. |
| DT_RELA               | 7                      | d_ptr       | mandatory  | optional      | This type of entry contains the address of a relocation table. The elements in this table contain explicit addends, such as Elf32_Rela in 32-bit files. An object file may have multiple relocation sections. When creating a relocation table for an executable file or shared object file, the link editor concatenates these sections to form a single table. Although these sections are independent in the object file, the dynamic linker treats them as a single table. When the dynamic linker creates a process image for an executable file or adds a shared object to a process image, it reads the relocation table and performs the related actions. If this element exists, the dynamic structure must also contain DT_RELASZ and DT_RELAENT elements. If relocation is mandatory for a file, then either DT_RELA or DT_REL may be present. |
| DT_RELASZ             | 8                      | d_val       | mandatory  | optional      | This type of entry contains the total byte size of the DT_RELA relocation table. |
| DT_RELAENT            | 9                      | d_val       | mandatory  | optional      | This type of entry contains the byte size of a DT_RELA relocation entry. |
| DT_STRSZ              | 10                     | d_val       | mandatory  | mandatory     | This type of entry gives the byte size of the string table, measured in bytes. |
| DT_SYMENT             | 11                     | d_val       | mandatory  | mandatory     | This type of entry gives the byte size of a symbol table entry. |
| DT_INIT               | 12                     | d_ptr       | optional   | optional      | This type of entry gives the address of the initialization function. |
| DT_FINI               | 13                     | d_ptr       | optional   | optional      | This type of entry gives the address of the termination function. |
| DT_SONAME             | 14                     | d_val       | ignored    | optional      | This type of entry gives the string table offset of a NULL-terminated string, and the corresponding string is the name of a shared object. The offset is actually an index into DT_STRTAB. |
| DT_RPATH              | 15                     | d_val       | optional   | ignored       | This type of entry contains the string table offset of a NULL-terminated string, and the corresponding string is the search path used when searching for libraries. The offset is actually an index into DT_STRTAB. |
| DT_SYMBOLIC           | 16                     | ignored     | ignored    | optional      | If this type of entry appears in a shared object library, it will alter the dynamic linker's symbol resolution algorithm. The dynamic linker will first search for symbols starting from the shared object file itself, and only when the search fails will it search the executable file for the corresponding symbol. |
| DT_REL                | 17                     | d_ptr       | mandatory  | optional      | This type of entry is similar to the DT_RELA type, except that its table contains implicit addends, which for 32-bit files means Elf32_Rel. If the ELF file contains this element, the dynamic structure must also contain DT_RELSZ and DT_RELENT type elements. |
| DT_RELSZ              | 18                     | d_val       | mandatory  | optional      | This type of entry contains the total byte size of the DT_REL relocation table. |
| DT_RELENT             | 19                     | d_val       | mandatory  | optional      | This type of entry contains the byte size of a DT_REL relocation entry. |
| DT_PLTREL             | 20                     | d_val       | optional   | optional      | This type of entry gives the address of the relocation entries referenced by the procedure linkage table. Depending on the situation, the address corresponding to d_val may contain DT_REL or DT_RELA. All relocations in the procedure linkage table must use the same relocation method. |
| DT_DEBUG              | 21                     | d_ptr       | optional   | ignored       | This type of entry is used for debugging. The ABI does not specify its content, and programs that access these entries may not be ABI-compatible. |
| DT_TEXTREL            | 22                     | ignored     | optional   | optional      | If the file does not contain an entry of this type, it means no relocation entry can cause modification to a non-writable segment. If present, there may be several relocation entries that request modifications to non-writable segments, so the dynamic linker can prepare accordingly. |
| DT_JMPREL             | 23                     | d_ptr       | optional   | optional      | The d_ptr member of this type of entry contains the address of the procedure linkage table, and when indexing, this address should be cast to a pointer of the corresponding relocation entry type. Separating relocation entries is beneficial for allowing the dynamic linker to ignore them during process initialization (when lazy binding is enabled). If this member exists, the related DT_PLTRELSZ and DT_PLTREL must also be present. |
| DT_BIND_NOW           | 24                     | ignored     | optional   | optional      | If an entry of this type exists in an executable file or shared object file, the dynamic linker should relocate all addresses that need relocation in the file before transferring control to the program. This entry takes priority over lazy binding, which can be set via environment variables or dlopen(BA_LIB). |
| DT_LOPROC  ~DT_HIPROC | 0x70000000 ~0x7fffffff | unspecified | unspecified | unspecified | This range of entries is reserved for processor-specific semantics. |

Tag values not appearing in this table are reserved. Furthermore, except for the DT_NULL element at the end of the array and the relative ordering constraint of DT_NEEDED elements, other entries can appear in any order.

## Global Offset Table

The GOT table is divided into two parts in ELF files:

-   .got, which stores the addresses of imported variables.
-   .got.plt, which stores the addresses of imported functions.

Generally speaking, position-independent code cannot contain absolute virtual addresses. The GOT table contains hidden absolute addresses, which allows obtaining the absolute addresses of related symbols without violating position independence and program code segment sharing. A program can use position-independent code to reference its GOT table, then extract the absolute values to redirect position-independent references to absolute addresses.

Initially, the GOT table contains the information needed for relocation. When a system creates memory segments for a loadable object file, the dynamic linker processes relocation entries, some of which may be of type R_386_GLOB_DAT, pointing to the GOT table. The dynamic linker determines the values of the related symbols, calculates their absolute addresses, and sets the appropriate memory table entries to the corresponding values. Although the absolute addresses are still unknown when the linker builds the object file, the dynamic linker knows the addresses of all memory segments and can therefore calculate the absolute addresses of the contained symbols.

If a program needs to directly access the absolute address of a symbol, that symbol will have a GOT table entry. Since executable files and shared object files each have separate entries, the address of a symbol may appear in multiple tables. The dynamic linker processes all GOT table relocation entries before granting permissions to the code segments in the process image, to ensure that all absolute addresses are accessible during execution.

The 0th entry in the GOT table contains the address of the dynamic structure (_DYNAMIC). This allows a program, such as the dynamic linker, to find the corresponding dynamic section before performing its relocations. This is very important for the dynamic linker, because it must be able to relocate its own memory image without depending on other programs.

In different programs, the system may choose different memory segment addresses for the same shared object file; even for the same program, library addresses may differ across different executions. However, once the process image is established, the memory segment addresses will not change again. As long as a process exists, its memory segment addresses will remain at fixed locations.

The format and interpretation of the GOT table depends on the specific processor. For the Intel architecture, the `_GLOBAL_OFFSET_TABLE_` symbol may be used to access this table.

```
extern Elf32_Addr _GLOBAL_OFFSET_TABLE[];
```

`_GLOBAL_OFFSET_TABLE_` may be located in the middle of the .got section, so that both positive and negative indices can be used to access the table.

In the Linux implementation, the specific meanings of the first three entries in .got.plt are as follows:

-   GOT[0], the address of .dynamic.
-   GOT[1], a pointer to link_map, used only in the dynamic loader, containing the information about the current ELF object needed for symbol resolution. Each link_map is a node in a doubly linked list, and this list stores the information of all loaded ELF objects.
-   GOT[2], a pointer to the `_dl_runtime_resolve` function in the dynamic loader.

The entries after .got.plt are the reference addresses of functions from different .so files in the program. The following diagram shows the corresponding relationship.

![img](./figure/got.png)

## Procedure Linkage Table

The PLT table redirects imported functions to absolute addresses. It mainly consists of two parts:

-   **.plt**, related to commonly imported functions, such as read and other functions.
-   **.plt.got**, related to dynamic linking.

Strictly speaking, the PLT table is not a lookup table, but rather a block of code. This content is code-related.

Under dynamic linking, there are a large number of function references between program modules. If all function addresses were resolved before the program starts executing, dynamic linking would spend considerable time on symbol lookup and relocation for inter-module function references. However, during program execution, many functions may never be used by the time the program finishes, so linking all functions at the beginning may waste a large amount of resources. Therefore, ELF adopts a lazy binding approach, whose basic idea is that binding (symbol lookup, relocation, etc.) is performed only when a function is first used; if a function is not used, no binding is performed. So before program execution begins, function calls between modules are not bound, and the dynamic linker is responsible for binding them only when needed.

In the case of lazy binding, the overall flow is shown in the diagram below. The blue lines represent the flow for the first execution, and the red lines represent the flow for subsequent calls:

![](./figure/lazy-plt.png)

The LD_BIND_NOW environment variable can change the behavior of the dynamic linker. If its value is non-empty, the dynamic linker will execute PLT table entries before handing control to the program. That is, the dynamic linker executes relocation entries of type R\_3862\_JMP_SLOT during the process initialization phase. Otherwise, the dynamic linker performs lazy binding on procedure linkage table entries, deferring symbol resolution and relocation until the first time the corresponding entry is executed.

Note

>   Lazy binding generally improves application performance because unused symbols do not increase the dynamic linking overhead. However, there are two situations where lazy binding can lead to unexpected behavior. First, the initial reference to a function in a shared object file generally takes longer than subsequent calls, because the dynamic linker needs to intercept the call to resolve the symbol. Some applications cannot tolerate this unpredictability. Second, if an error occurs and the dynamic linker cannot resolve the symbol, the dynamic linker will terminate the program. In the case of lazy binding, this situation can occur at any time. When lazy binding is disabled, the dynamic linker will not encounter such errors during process initialization, because these are all executed before the application gains control.

The link editor cannot resolve execution flow transfers (such as function calls) from one executable file or shared object file to another. The linker arranges for the program to transfer control to entries in the procedure linkage table. In the Intel architecture, the procedure linkage table resides in the shared code segment, but it uses data in the GOT table. The dynamic linker determines the absolute address of the target and modifies the corresponding memory image in the GOT table. Therefore, the dynamic linker can redirect PLT entries without violating position independence and program code segment compatibility. Executable files and shared object files each have independent PLT tables.

The absolute address procedure linkage table is as follows:

```assembly
.PLT0:  pushl	got_plus_4
        jmp	*got_plus_8
	      nop; nop
	      nop; nop
.PLT1:  jmp	*name1_in_GOT
        pushl	$offset@PC
	      jmp	.PLT0@PC
.PLT2:  jmp	*name2_in_GOT
        push	$offset
	     jmp	.PLT0@PC
	     ...
```

The position-independent procedure linkage table addresses are as follows:

```assembly
.PLT0:  pushl	4(%ebx)
        jmp	*8(%ebx)
	      nop; nop
	      nop; nop
.PLT1:  jmp	*name1_in_GOT(%ebx)
        pushl	$offset
	      jmp	.PLT0@PC
.PLT2:  jmp	*name2_in_GOT(%ebx)
        push	$offset
	      jmp	.PLT0@PC
	      ...
```

As can be seen, the procedure linkage table handles absolute addresses and position-independent code differently. However, when the dynamic linker processes them, the interface used is the same.









The dynamic linker and the program resolve symbol references in the procedure linkage table and the global offset table in the following manner.

1.  When the program's memory image is first created, the dynamic linker sets the second and third entries of the GOT table to special values, which are explained in detail in the following steps.
2.  If the procedure linkage table is position-independent, then the address of the GOT table must be in the ebx register. Each shared object file in the process image has an independent PLT table, and the program only transfers control flow to PLT entries within the same object file. Therefore, the calling function is responsible for setting the base address of the global offset table in the register before calling a PLT entry.
3.  Here is an example: suppose the program calls name1, which transfers control to label .PLT1.
4.  The first instruction jumps to the address of name1 in the global offset table. Initially, the global offset table contains the address of the next pushl instruction in the PLT, not the actual address of name1.
5.  Therefore, the program pushes the corresponding function's offset in `rel.plt` (relocation offset, reloc_index) onto the stack. The relocation offset is a 32-bit, non-negative value. Furthermore, the relocation entry is of type R\_386\_JMP_SLOT, and it specifies the offset of the global offset table entry used in the previous jmp instruction within the GOT table. The relocation entry also contains a symbol table index, thus telling the dynamic linker which symbol is currently being referenced. In this example, it is name1.
6.  After pushing the relocation offset, the program jumps to .PLT0, which is the first entry in the procedure linkage table. The pushl instruction pushes the second entry of the GOT table (got_plus_4 or 4(%ebx), **information about the current ELF object**) onto the stack, giving the dynamic linker an identification. Then the program jumps to the third global offset table entry (got_plus\_8 or 8(%ebx), **a pointer to the `_dl_runtime_resolve` function in the dynamic loader**), which transfers the program flow to the dynamic linker.
7.  When the dynamic linker receives control, it performs a pop operation, looks up the relocation entry, resolves the value of the corresponding symbol, writes the address of name1 into the global offset table entry, and finally transfers control to the destination address.
8.  After the procedure linkage table has been executed, program control will be directly transferred to the name1 function, and the dynamic linker will never be called again to resolve this function. That is, the jmp instruction at .PLT1 will jump directly to name1, rather than executing the pushl instruction again.

## .rel(a).dyn & .rel(a).plt

.rel.dyn contains information about variables that need to be relocated in dynamically linked binary files, while .rel.plt contains information about functions that need to be relocated. Both types of relocation sections use the following structure (using 32-bit as an example):

```
typedef struct {
    Elf32_Addr        r_offset;
    Elf32_Word       r_info;
} Elf32_Rel;

typedef struct {
    Elf32_Addr     r_offset;
    Elf32_Word    r_info;
    Elf32_Sword    r_addend;
} Elf32_Rela;
```

Entries of type Elf32_Rela contain explicit addend information. Entries of type Elf32_Rel store implicit addend information at the location to be modified. Due to processor architecture reasons, both forms exist and may even be necessary. Therefore, a specific machine implementation may use only one form, or may use both forms depending on context.

The description of each field is as follows:

| Member   | Description                                                  |
| -------- | ------------------------------------------------------------ |
| r_offset | **This member gives the location where relocation needs to be applied.** For a relocatable file, this value is the byte offset from the beginning of the section header where the symbol to be relocated resides to the location to be relocated. For executable files or shared object files, its value is the **virtual address** that needs to be relocated, which is generally the address of the GOT table. |
| r_info   | **This member gives the symbol table index of the symbol that needs to be relocated, as well as the corresponding relocation type.** For example, a relocation entry for a call instruction will contain the symbol table index of the called function. If the index is STN_UNDEF, then the relocation uses 0 as the "symbol value". Furthermore, the relocation type is processor-dependent. |
| r_addend | This member gives a constant addend used to compute the value to be filled into the relocatable field. |

More specific field information about r_info is shown in the code below:

- The upper three bytes of r_info represent the position of this dynamic symbol in the `.dynsym` symbol table.
- The lowest byte of r_info represents the relocation type.

```c
#define ELF32_R_SYM(i)    ((i)>>8)
#define ELF32_R_TYPE(i)   ((unsigned char)(i))
// Used to construct r_info
#define ELF32_R_INFO(s,t) (((s)<<8)+(unsigned char)(t))
```

A relocation section references two other sections: **the symbol table and the section to be modified**. The sh_info and sh_link members of the section header give the corresponding relationships.

Here, we specifically discuss the possible relocation types. In the calculations below, we assume that a relocatable file is being converted into an executable file or shared object file. Conceptually, the linker combines one or more relocatable files to produce the output file. It first decides how to combine and place these input files, then updates the symbol table values, and finally performs relocation. The relocation methods for executable files and shared object files are similar, and the results are nearly identical. In the following descriptions, we will use the following notation:

-   A (addend) is the addend used to compute the value of the relocatable field.
-   B (base) represents the base address at which the shared object file is loaded into memory during execution. Generally, the virtual base address of a shared object file is 0, but during execution, its address changes.
-   G (Global) represents the offset of the relocation entry's symbol in the global offset table at execution time.
-   GOT (global offset table) represents the address of the global offset table (GOT).
-   L (linkage) represents the section offset or address of a symbol in a procedure linkage table entry. The procedure linkage table entry redirects function calls to the correct target location. The link editor constructs the initial procedure linkage table, and the dynamic linker modifies these entries during execution.
-   P (place) represents the location (section offset or address) of the storage unit being relocated (calculated using r_offset).
-   S (symbol) represents the value of the symbol whose index is in the relocation entry.

The r_offset value of a relocation entry is the offset or virtual address of the first byte of the affected storage unit. The relocation type specifies which bits need to be modified and how to compute their values. The Intel architecture uses only ELF32_REL relocation entries, and the member to be relocated retains the corresponding addend value. In all cases, the addend value and the computed result use the same byte order.

The relocation types and some of their meanings are as follows:

| Name           | Value | Field  | Calculation | Meaning                                                      |
| -------------- | ----- | ------ | ----------- | ------------------------------------------------------------ |
| R_386_NONE     | 0     | none   | none        |                                                              |
| R_386_32       | 1     | word32 | S + A       |                                                              |
| R_386_PC32     | 1     | word32 | S + A - P   |                                                              |
| R_386_GOT32    | 1     | word32 | G + A - P   | This relocation type computes the distance from the base of the global offset table to the symbol's global offset table entry. It also instructs the linker to create a global offset table. |
| R_386_PLT32    | 1     | word32 | L + A - P   | This relocation type computes the address of the symbol's procedure linkage table entry. It also instructs the linker to create a procedure linkage table. |
| R_386_COPY     | 5     | none   | none        | This relocation type is created by the linker for the dynamic linking process. Its offset member points to a location in a writable segment. The symbol table specifies that such a symbol should exist in both the current object file and a shared object file. During execution, the dynamic linker copies the data associated with the shared object symbol to the location specified by the above offset. |
| R_386_GLOB_DAT | 6     | word32 | S           | This relocation type is used to set a symbol in the global offset table to the address of the specified symbol. This special relocation type allows determining the relationship between symbols and global offset table entries. |
| R_386_JMP_SLOT | 7     | word32 | S           | This relocation type is created by the linker for the dynamic linking process. Its offset member gives the location of the corresponding procedure linkage table entry. The dynamic linker modifies the procedure linkage table to transfer program control to the symbol address indicated above. |
| R_386_RELATIVE | 8     | word32 | B + A       | This relocation type is created by the linker for the dynamic linking process. Its offset member gives a location in the shared object that contains a value representing a relative address. The dynamic linker computes the corresponding virtual address by adding the virtual address at which the shared object file is loaded to the above relative address. This type of relocation entry sets the symbol table index to 0. |
| R_386_GOTOFF   | 9     | word32 | S + A - GOT | This relocation type computes the difference between the symbol value and the global offset table address. It also instructs the linker to create a global offset table. |
| R_386_GOTPC    | 10    | word32 | S + A - P   | This relocation type is similar to `R_386_PC32`, except that it uses the address of the global offset table in the computation. Normally, the symbol referenced in the relocation entry is `_GLOBAL_OFFSET_TABLE_`, and it instructs the linker to create a global offset table. |

## .dynsym

Dynamically linked ELF files have a dedicated dynamic symbol table, which also uses the Elf32_Sym structure, but is stored in the .dynsym section. Here is the Elf32_Sym structure again:

```c
typedef struct
{
  Elf32_Word    st_name;   /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;  /* Symbol value */
  Elf32_Word    st_size;   /* Symbol size */
  unsigned char st_info;   /* Symbol type and binding */
  unsigned char st_other;  /* Symbol visibility under glibc>=2.2 */
  Elf32_Section st_shndx;  /* Section index */
} Elf32_Sym;
```

Note that `.dynsym` is needed at runtime, and all export/import symbol information in the ELF file is stored here. However, the information stored in the `.symtab` section is compile-time symbol information, which will be removed after `strip`.

We mainly focus on two members of the dynamic symbol:

-   st_name, this member stores the offset of the dynamic symbol in the .dynstr table (dynamic string table).
-   st_value, if this symbol is exported, this member stores the corresponding virtual address.

The association between dynamic symbols and the Elf_Verdef that points to them is stored in the `.gnu.version` section. This section is an array composed of Elf_Verneed structures. Each entry corresponds to an entry in the dynamic symbol table. In fact, this structure has only one field: a 16-bit integer representing the index into the `.gnu.version_r` section.

In addition, the dynamic linker uses the index from the r_info member of the Elf_Rel structure as an index into both the .dynsym section and the .gnu.version section. This makes it possible to map each symbol to its exact version.

## .dynstr

This section contains strings needed for dynamic linking.

## Misc 

### version releated sections

ELF files can not only import external symbols, but can also import symbols of a specific version. For example, we can import some standard library functions from GLIBC_2.2.5, such as printf. Among them, .gnu.version_r stores the version definitions, and the corresponding structure is Elf_Verdef.

#### .gnu.version

This section has a one-to-one correspondence with the symbol information in .dynsym, meaning both have the same number of elements. Each element in .gnu.version is of type `Elfxx_Half`, specifying the version information of the corresponding symbol. There are two reserved values in `Elfxx_Half`:

- 0, indicates that this symbol is local and not publicly visible. The `__gmon_start__` below is a local symbol.
- 1, indicates that this symbol is defined in the current object file and is globally accessible. The `_IO_stdin_used` below is a global symbol.

```assembly
LOAD:080482D8 ; ELF GNU Symbol Version Table
LOAD:080482D8                 dw 0
LOAD:080482DA                 dw 2                    ; setbuf@@GLIBC_2.0
LOAD:080482DC                 dw 2                    ; read@@GLIBC_2.0
LOAD:080482DE                 dw 0                    ; local  symbol: __gmon_start__
LOAD:080482E0                 dw 2                    ; strlen@@GLIBC_2.0
LOAD:080482E2                 dw 2                    ; __libc_start_main@@GLIBC_2.0
LOAD:080482E4                 dw 2                    ; write@@GLIBC_2.0
LOAD:080482E6                 dw 2                    ; stdin@@GLIBC_2.0
LOAD:080482E8                 dw 2                    ; stdout@@GLIBC_2.0
LOAD:080482EA                 dw 1                    ; global symbol: _IO_stdin_used
...
.rodata:0804866C                 public _IO_stdin_used
.rodata:0804866C _IO_stdin_used  db    1                 ; DATA XREF: LOAD:0804825C↑o
.rodata:0804866D                 db    0
.rodata:0804866E                 db    2
.rodata:0804866F                 db    0
.rodata:0804866F _rodata         ends
```

#### .gnu.version_d

Version definitions of symbols.

#### .gnu.version_r

Version references (version needs) of symbols.

## References

- https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-PDA/LSB-PDA.junk/symversion.html
