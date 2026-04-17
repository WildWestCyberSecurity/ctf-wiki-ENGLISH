# Symbol Table

## .symtab

The .symtab section stores the symbol table, which holds the information needed to locate and relocate symbolic definitions and symbolic references within a program. The symbol table is an array, and each element is defined as follows:

```c
typedef struct
{
  Elf32_Word    st_name;   /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;  /* Symbol value */
  Elf32_Word    st_size;   /* Symbol size */
  unsigned char st_info;   /* Symbol type and binding */
  unsigned char st_other;  /* Symbol visibility; 0 in current version */
  Elf32_Section st_shndx;  /* Section index */
} Elf32_Sym;
```

The meaning of each field is as follows:

| Field    | Description |
| -------- | ----------- |
| st_name  | Symbol name. It is an index into the string table. If the value is non-zero, it represents the string table index that gives the symbol name; otherwise, the symbol has no name. |
| st_value | Symbol value. This is a related address. |
| st_size  | Size of the associated object. |
| st_info  | Symbol type and binding attributes. |
| st_other | This member currently has no defined meaning. Its value should be 0. |
| st_shndx | The section header table index associated with each symbol table entry. |

### Symbol Type - st_info

The lower 4 bits of the st_info field specify the symbol type:

| Macro Name    | Value | Description |
| ------------- | ----- | ----------- |
| STT_NOTYPE    | 0     | Symbol type not defined |
| STT_OBJECT    | 1     | Symbol associated with a data object |
| STT_FUNC      | 2     | Symbol associated with a function or other executable code |
| STT_SECTION   | 3     | Symbol associated with a section, mainly for relocation |
| STT_FILE      | 4     | The symbol's name gives the name of the source file associated with the object file |
| STT_LOPROC~STT_HIPROC | 13~15 | Reserved for processor-specific semantics |

### Symbol Binding - st_info

The upper 4 bits of the st_info field specify the symbol's binding type, determining the scope of visibility and behavior:

| Macro Name    | Value | Description |
| ------------- | ----- | ----------- |
| STB_LOCAL     | 0     | Local symbol, not visible outside the object file |
| STB_GLOBAL    | 1     | Global symbol, visible to all object files being combined |
| STB_WEAK      | 2     | Similar to global symbols but with lower precedence |
| STB_LOPROC~STB_HIPROC | 13~15 | Reserved for processor-specific semantics |

The following code shows how to manipulate st_info:

```c
#define ELF32_ST_BIND(i)    ((i)>>4)
#define ELF32_ST_TYPE(i)    ((i)&0xf)
#define ELF32_ST_INFO(b, t) (((b)<<4)+((t)&0xf))
```

### Symbol Value - st_value

The symbol table entries for different object files have slightly different interpretations of the st_value member:

1. In relocatable files, if the section index corresponding to the symbol is SHN_COMMON, st_value gives the alignment constraint.
2. In relocatable files, if the symbol is defined (i.e., not SHN_UNDEF), st_value gives the offset from the beginning of the section indicated by st_shndx.
3. In executable and shared object files, st_value gives a virtual address.
