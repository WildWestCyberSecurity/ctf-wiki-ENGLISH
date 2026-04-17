# Hijacking Program Control Flow

In the process of hijacking program control flow, we can consider the following approaches.

## Directly Controlling EIP



## Return Address

This involves controlling the return address on the program's stack.

## Jump Pointers

Here we can consider the following approaches:

- call
- jmp

## Function Pointers

Common function pointers include:

- vtable, function table, such as the vtable of IO_FILE, printf function table.
- Hook pointers, such as `malloc_hook`, `free_hook`.
- handler

## Modifying Control Flow Related Variables
