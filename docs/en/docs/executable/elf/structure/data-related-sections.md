# Data Related Sections

## .BSS Section

This section corresponds to uninitialized global variables. This section does not occupy space in the ELF file, but it does occupy space in the program's memory image. When the program begins execution, the system initializes this data to 0. BSS is actually an abbreviation for "block started by symbol".

## .data Section

These sections contain initialized data that will appear in the program's memory image.

## .rodata Section

These sections contain read-only data, which typically participates in the non-writable segments of the process image.
