# Program Linking

## Static Linking

## Dynamic Linking

Dynamic linking mainly resolves references to variables or functions during program initialization or during program execution. Certain sections and header elements in ELF files are related to dynamic linking. The dynamic linking model is defined and implemented by the operating system.

### Dynamic Linker

The dynamic linker can be used to help load the libraries required by the application and resolve the dynamic symbols (functions and global variables) exported by the libraries.

When using dynamic linking to build a program, the link editor adds a PT_INTERP type element to the executable file's program header, in order to tell the system to invoke the dynamic linker as the program interpreter.

> Note that the dynamic linker provided by the system varies across different systems.

The executable program and the dynamic linker cooperate to create the process image for the program. The specific details are as follows:

1. Add the memory segments of the executable file to the process image.
2. Add the memory segments of the shared object files to the process image.
3. Perform relocations for the executable file and the shared object files.
4. If a file descriptor was passed to the dynamic linker, close it.
5. Transfer control to the program. This makes it appear as though the program obtained execution rights directly from the executable file.

The link editor also creates various data to assist the dynamic linker in handling executable files and shared object files, for example:

- The section .dynamic of type SHT_DYNAMIC contains various data; at the beginning of this section is other dynamic linking information.
- The section .hash of type SHT_HASH contains the symbol hash table.
- The sections .got and .plt of type SHT_PROGBITS contain two separate tables: the Global Offset Table and the Procedure Linkage Table. Programs use the Procedure Linkage Table to handle position-independent code.

Because all UNIX System V implementations import basic system services from a shared object file, the dynamic linker participates in the execution of every TIS ELF-conforming program.

As mentioned in program loading, shared object files may occupy virtual addresses different from those recorded in the program header. The dynamic linker will relocate the memory image and update absolute addresses before the program gets control. If a shared object file is indeed loaded at the address specified in its program header, then the absolute address values will be correct. However, typically this is not the case.

If the process environment contains a non-empty value named LD_BIND_NOW, then the dynamic linker will perform all relocations before transferring control to the program. For example, all of the following environment variable values would specify this behavior:

- LD_BIND_NOW = 1
- LD_BIND_NOW = on
- LD_BIND_NOW = off

Otherwise, LD_BIND_NOW either does not exist in the current process environment or has a non-empty value. The dynamic linker can defer resolving Procedure Linkage Table entries — this is actually lazy binding of the PLT table, meaning address resolution is performed only when the program needs to use a particular symbol. This can reduce the overhead of symbol resolution and relocation.


### Function Address

The address references of functions in executable files and the related references in shared objects may not be resolved to the same value. The corresponding references in a shared object file will be resolved by the dynamic linker to the virtual address of the function itself. The corresponding references in the executable file (originating from shared object files) will be resolved by the link editor to the address of the corresponding function's entry in the Procedure Linkage Table.

To allow different function addresses to work as expected, if an executable file references a function defined in a shared object file, the link editor will place the corresponding function's Procedure Linkage Table entry into the associated symbol table entry. The dynamic linker performs special handling for such symbol table entries. If the dynamic linker is looking for a symbol and encounters a symbol table entry that is in the executable file, it follows these rules:

1. If the symbol table entry's `st_shndx` is not `SHN_UNDEF`, the dynamic linker finds the definition of this symbol and uses its st_value as the symbol's address.
2. If `st_shndx` is `SHN_UNDEF` and the symbol type is `STT_FUNC`, and the `st_value` member is not 0, the dynamic linker treats this entry as special and uses the value of `st_value` as the symbol's address.
3. Otherwise, the dynamic linker considers the symbol in the executable file as undefined and continues processing.

Some relocations are associated with Procedure Linkage Table entries. These entries are used for direct function calls rather than referencing function addresses. These relocations are not handled in the manner described above, because the dynamic linker must not redirect Procedure Linkage Table entries to point to themselves.

### Shared Object Dependencies

When the link editor is processing an archive library, it extracts library members and copies them into the output object file. Such static linking operations do not require the dynamic linker's participation during execution. Shared object files also provide services, and the dynamic linker must attach the appropriate shared object files to the process image for execution. Therefore, executable files and shared object files specifically describe their dependencies.

When a dynamic linker creates memory segments for an object file, the dependencies described in DT_NEEDED entries indicate what dependency files are needed to support the program's services. By continuously linking the referenced shared object files (even if a shared object file is referenced multiple times, it will only be linked once by the dynamic linker) and their dependencies, the dynamic linker builds a complete process image. When resolving symbol references, the dynamic linker uses BFS (Breadth-First Search) to check symbol tables. That is, it first checks the symbol table of the executable file itself, then checks the symbol tables of the DT_NEEDED entries in order, and then continues to look at the next level of dependencies, and so on. Shared object files must be readable by the program; other permissions are not necessarily required.

The names in the dependency list are either the string from DT_SONAME or the pathname of the shared object file used to build the corresponding object file. For example, if a linker uses a shared object file with a DT_SONAME entry named lib1 and another shared object file with a pathname of /usr/lib/lib2, then the executable file will contain lib1 and /usr/lib/lib2 in its dependency list.

If a shared object file has one or more / characters, such as /usr/lib/lib2 or directory/file, the dynamic linker will directly use that string as the pathname. If the name does not contain /, such as lib1, then the following three mechanisms specify the search order for shared object files:

- First, the dynamic array tag DT_RPATH may provide a string containing a series of directories separated by colons. For example, /home/dir/lib:/home/dir2/lib: tells us to first search in the `/home/dir/lib` directory, then in `/home/dir2/lib`, and finally in the current directory.

- Second, the process environment variable named LD_LIBRARY_PATH contains a series of directories in the format described above, optionally followed by a semicolon and then another directory list. Here is an example with the same effect as described in the first mechanism:

  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib;/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:;

  All directories in LD_LIBRARY_PATH will only be searched after DT_RPATH has been searched. Although some programs (such as the link editor) handle the lists before and after the semicolon differently, the dynamic linker handles them in exactly the same way. Additionally, the dynamic linker accepts the semicolon notation syntax as described above.

- Finally, if neither of the above two sets of directories can locate the desired library, the dynamic linker searches for libraries under the `/usr/lib` path.

Note

> **For security purposes, for `set-user` and `set-group` identified programs, the dynamic linker ignores search environment variables (such as `LD_LIBRARY_PATH`) and only searches directories specified by `DT_RPATH` and `/usr/lib`.**
