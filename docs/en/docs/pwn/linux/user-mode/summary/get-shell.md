# Shell Acquisition Summary

## Overview

The shells we obtain generally come in two forms:

- A directly interactive shell
- A shell bound to a specified IP and port

Below is a summary of several common methods for obtaining a shell.

## shellcode

When using shellcode to obtain a shell, the basic requirement is that we can place the shellcode in a **writable and executable memory region**. Therefore, when there is no writable and executable memory region available, we need to use functions like `mprotect` to set the appropriate memory permissions.

Additionally, sometimes the characters in the shellcode must meet certain requirements, such as being printable characters, letters, digits, etc.

## system

Here we generally execute system("/bin/sh"), system('sh'), and similar functions.

The main thing we need here is to find certain addresses. Refer to the section on getting addresses.

- Address of system
- Address of "/bin/sh" or "sh"
    - Whether the string exists in the binary
    - Consider reading the corresponding string manually
    - libc actually contains /bin/sh

A great advantage of using system to obtain a shell is that we only need to set up one argument. The disadvantage is that when setting up the argument, we may corrupt environment variables, preventing execution.

## execve

Execute execve("/bin/sh", NULL, NULL).

When using `execve` to obtain a shell, the first few points are the same as with system. However, it has the advantage of being almost unaffected by environment variables. The disadvantage is that we need to set up three arguments.

Additionally, in glibc, we can also use one_gadget to obtain a shell.

## syscall

The system call number `__NR_execve` is 11 on IA-32 and 59 on x86-64.

Its advantage is that it is almost unaffected by environment variables. However, we need to find system call instructions like `syscall`.
