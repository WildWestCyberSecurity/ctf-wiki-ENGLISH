# Program Execution Flow

**Reference: Execution Angleboye@Bamboofox.**

## Static Execution

Here is the basic process of static program execution.

![](./figure/run_static_linking.png)

## Dynamic Execution

![](./figure/run_dynamic_linking.png)



Here is another more detailed diagram.

![image-20181201152204864](figure/program-running-overview.png)

## Basic Operation Description

### sys_execve

This function is mainly used to execute a new program, i.e., to execute the program we want to run. It checks the corresponding argv, envp, and other parameters.

### do_execve

This function opens the target image file and reads a specified length (currently 128) of bytes from the beginning of the target file to obtain the basic information of the target file.

### search_binary_handler

This function searches the binary file type queue that supports handling the current type, in order to allow the handlers of various executable programs to perform the appropriate processing.

### load_elf_binary

The main processing flow of this function is as follows:

- Check and obtain the header information of the ELF file.

- If the target file uses dynamic linking, use the .interp section to determine the loader path.

- Map the corresponding segments recorded in the program header into memory. The program header contains the following important information:

  - The address to which each segment needs to be mapped.
  - The corresponding permissions of each segment.
  - Records which sections belong to which segments.

  The specific mapping is as follows:

  ![](./figure/memory_mapping.png)

  Handle different cases:

  - In the case of dynamic linking, change the return address of sys_execve to the entry point of the loader (ld.so).
  - In the case of static linking, change the return address of sys_execve to the entry point of the program.

### ld.so

This file has the following functions:

- Mainly used to load the shared libraries recorded in DT_NEED of the ELF file.
- Initialization work:
  - Initialize the GOT table.
  - Merge the symbol table into the global symbol table.

### _start

The _start function passes the following items to libc_start_main:

- Starting address of environment variables
- .init
  - Initialization work before starting the main function.
- fini
  - Cleanup work before program termination.
