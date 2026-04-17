# Sections

## Overview

The section header table in ELF files describes all sections within the ELF file. The **Section Header Table** must be able to locate all sections in the ELF file.

## Common Sections

We know that sections contain most of the information in an ELF file. Moreover, we can think of the section header table as the index for finding these sections.

The sections in an ELF file satisfy several conditions:

1. Every section in an ELF file has a corresponding section header to describe it. A section header may exist but the corresponding section does not have to exist.
2. Each section occupies a contiguous (possibly empty) sequence of bytes in the file.
3. Sections in a file do not overlap. No byte in a file can exist in two sections simultaneously.
4. There may be inactive space (dead bytes/padding) in an object file. The contents of these areas are unspecified.

The list of commonly used sections is as follows:

| Name | Description |
| ---- | ----------- |
| .bss | Contains uninitialized global variables. Takes no space in the file. |
| .comment | Contains version control information |
| .data | Contains initialized data |
| .debug | Contains debug information |
| .dynamic | Contains dynamic linking information |
| .dynstr | Contains strings needed for dynamic linking |
| .dynsym | Contains the dynamic symbol table |
| .fini | Contains process termination code |
| .got | Contains the Global Offset Table |
| .hash | Contains the symbol hash table |
| .init | Contains process initialization code |
| .interp | Contains the path of the program interpreter |
| .line | Contains line number information for debugging |
| .note | Contains note information |
| .plt | Contains the Procedure Linkage Table |
| .rel.dyn | Contains dynamic relocation information |
| .rel.plt | Contains PLT relocation information |
| .rodata | Contains read-only data |
| .shstrtab | Contains section name strings |
| .strtab | Contains strings, usually symbol name strings |
| .symtab | Contains the symbol table |
| .text | Contains executable code |
