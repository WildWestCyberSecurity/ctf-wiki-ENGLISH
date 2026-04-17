# Stack Introduction

## Basic Stack Introduction

The stack is a typical Last in First Out (LIFO) data structure. Its main operations are push and pop, as shown in the figure below (from Wikipedia). Both operations act on the top of the stack; of course, it also has a bottom.

![Basic Stack Operations](./figure/Data_stack.png)

High-level languages are converted into assembly programs at runtime. During the execution of assembly programs, this data structure is extensively utilized. Each program has a virtual address space at runtime, and a certain part of it is the program's stack, used for saving function call information and local variables. Additionally, the common operations are also push and pop. It is important to note that **the program's stack grows from high addresses to low addresses in the process address space**.

## Function Call Stack

Please be sure to carefully read the following articles to learn about the basic function call stack.

- [C Language Function Call Stack (Part 1)](http://www.cnblogs.com/clover-toeic/p/3755401.html)
- [C Language Function Call Stack (Part 2)](http://www.cnblogs.com/clover-toeic/p/3756668.html)

Here is another diagram of registers

![](./figure/register.png)

It should be noted that 32-bit and 64-bit programs have the following simple differences

- **x86**
    - **Function arguments** are located above the **function return address**
- **x64**
    - In the System V AMD64 ABI (used by Linux, FreeBSD, macOS, etc.), the first six integer or pointer arguments are stored in the **RDI, RSI, RDX, RCX, R8, and R9 registers** respectively. If there are more arguments, they are stored on the stack.
    - Memory addresses cannot be greater than 0x00007FFFFFFFFFFF, which is **6 bytes long**; otherwise, an exception will be thrown.

## Recommended Reading

- csapp
- Calling conventions for different C++ compilers and operating systems, Agner Fog
