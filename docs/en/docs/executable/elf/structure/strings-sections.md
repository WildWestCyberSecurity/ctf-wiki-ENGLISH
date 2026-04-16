# String Sections

## .strtab: String Table

This section describes the default string table, containing a series of NULL-terminated strings. ELF files use these strings to store symbol names in the program, including:

-   Variable names
-   Function names

This section does not need to be loaded during runtime; only the corresponding subset .dynstr section needs to be loaded.

Strings are generally indexed by the offset of their first character in the string table.

The first and last bytes of the string table are both NULL. Additionally, the string at index 0 either has no name or has an empty name; its interpretation depends on the context. The string table can also be empty, in which case the sh_size member of its section header will be 0. Indexing with an offset greater than 0 in an empty string table is obviously illegal.

The sh_name member of a section header is an index into the corresponding section header string table section, which is specified by the e_shstrndx member of the ELF header. The figure below shows a string table containing 25 bytes, and the strings associated with different indices.

| Index | +0   | +1   | +2   | +3   | +4   | +5   | +6   | +7   | +8   | +9   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | \0   | n    | a    | m    | e    | .    | \0   | V    | a    | r    |
| 10   | i    | a    | b    | l    | e    | \0   | a    | b    | l    | e    |
| 20   | \0   | \0   | x    | x    | \0   |      |      |      |      |      |

The strings contained are:

| Index | String       |
| ---- | ------------ |
| 0    | none         |
| 1    | name.        |
| 7    | Variable     |
| 11   | able         |
| 16   | able         |
| 24   | empty string |

We can observe that:

-   String table indices can reference any byte in the section.
-   Strings can appear multiple times.
-   References to substrings can exist.
-   The same string can be referenced multiple times.
-   Unreferenced strings can also exist in the string table.

This information will disappear after performing a `strip` operation.

## .shstrtab: Section Header String Table

This section has a storage structure similar to `.strtab`, but it stores section name strings.
