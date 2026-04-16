# Unicorn Engine

## What is the Unicorn Engine

Unicorn is a lightweight, multi-platform, multi-architecture CPU emulator framework. It allows us to focus on CPU operations while ignoring differences between machine devices. Imagine applying it in these scenarios: for instance, when we simply need to emulate code execution without requiring a real CPU to perform those operations, or when we want to more safely analyze malicious code, detect virus signatures, or verify the meaning of certain code during reverse engineering. Using a CPU emulator can greatly help us with convenience.

Its highlights (which are also thanks to Unicorn being developed based on [qemu](http://www.qemu.org)) include:

* Support for multiple architectures: Arm, Arm64 (Armv8), M68K, Mips, Sparc, & X86 (including X86_64).
* Native support for Windows and *nix systems (confirmed to include Mac OSX, Linux, *BSD & Solaris)
* A platform-independent, clean, and easy-to-use API
* Excellent performance using JIT compilation technology

You can learn more technical details about the Unicorn Engine at [Black Hat USA 2015](http://www.unicorn-engine.org/BHUSA2015-unicorn.pdf). GitHub project page: [unicorn](https://github.com/unicorn-engine/unicorn)

Despite being extraordinary, it cannot emulate an entire program or system, nor does it support system calls. You need to manually map memory and write data into it, and only then can you start emulation from a specified address.

## Application Scenarios

When can the Unicorn Engine be useful?

* You can call interesting functions in malware without creating a harmful process.
* For CTF competitions
* For fuzz testing
* For GDB plugins, plugins based on code emulation execution
* Emulating execution of obfuscated code

## How to Install

The simplest way to install Unicorn is via pip. Just run the following command in the terminal (this is the installation method for users who prefer Python; for those who want to use C, you'll need to check the official documentation to compile the source package):

``` shell
pip install unicorn
```

However, if you want to compile from source locally, you need to download the source package from the [download](http://www.unicorn-engine.org/download/) page, then execute the following commands:

* *nix platform users

``` shell
$ cd bindings/python
$ sudo make install
```

* Windows platform users

``` shell
cd bindings/python
python setup.py install
```

For Windows, after executing the above commands, you also need to copy all DLL files from the `Windows core engine` on the [download](http://www.unicorn-engine.org/download/) page to the `C:\locationtopython\Lib\site-packages\unicorn` location.

## Quick Guide to Using Unicorn

We will demonstrate how to use Python to call Unicorn's API and how easily it can emulate binary code. Of course, only a small portion of the API is used here, but it is sufficient for getting started.

``` python
 1 from __future__ import print_function
 2 from unicorn import *
 3 from unicorn.x86_const import *
 4 
 5 # code to be emulated
 6 X86_CODE32 = b"\x41\x4a" # INC ecx; DEC edx
 7 
 8 # memory address where emulation starts
 9 ADDRESS = 0x1000000
10 
11 print("Emulate i386 code")
12 try:
13     # Initialize emulator in X86-32bit mode
14     mu = Uc(UC_ARCH_X86, UC_MODE_32)
15 
16     # map 2MB memory for this emulation
17     mu.mem_map(ADDRESS, 2 * 1024 * 1024)
18 
19     # write machine code to be emulated to memory
20     mu.mem_write(ADDRESS, X86_CODE32)
21 
22     # initialize machine registers
23     mu.reg_write(UC_X86_REG_ECX, 0x1234)
24     mu.reg_write(UC_X86_REG_EDX, 0x7890)
25 
26     # emulate code in infinite time & unlimited instructions
27     mu.emu_start(ADDRESS, ADDRESS + len(X86_CODE32))
28 
29     # now print out some registers
30     print("Emulation done. Below is the CPU context")
31 
32     r_ecx = mu.reg_read(UC_X86_REG_ECX)
33     r_edx = mu.reg_read(UC_X86_REG_EDX)
34     print(">>> ECX = 0x%x" %r_ecx)
35     print(">>> EDX = 0x%x" %r_edx)
36 
37 except UcError as e:
38     print("ERROR: %s" % e)
```

The output is as follows:

``` shell
$ python test1.py 
Emulate i386 code
Emulation done. Below is the CPU context
>>> ECX = 0x1235
>>> EDX = 0x788f
```

The comments in the example are already very intuitive, but let's explain each line of code:

* Lines 2–3: Before using Unicorn, import the `unicorn` module. The example uses some x86 register constants, so the `unicorn.x86_const` module also needs to be imported.

* Line 6: This is the binary machine code we need to emulate, represented in hexadecimal. The corresponding assembly instructions are: "INC ecx" and "DEC edx".

* Line 9: The virtual address at which we will emulate execution of the above instructions.

* Line 14: Initialize Unicorn using the `Uc` class, which accepts 2 parameters: the hardware architecture and the hardware mode (bit width). In this example, we need to emulate 32-bit code on the x86 architecture. We use the variable `mu` to store the return value.

* Line 17: Use the `mem_map` method to map 2MB of memory space for emulation at the address declared on line 9. All CPU operations in the process should only access this memory region. The mapped memory has default read, write, and execute permissions.

* Line 20: Write the code to be emulated into the memory we just mapped. The `mem_write` method accepts 2 parameters: the memory address to write to and the code to write into memory.

* Lines 23–24: Use the `reg_write` method to set the values of the `ECX` and `EDX` registers.

* Line 27: Use the `emu_start` method to begin emulation. This API accepts 4 parameters: the address of the code to emulate, the memory address where emulation stops (here it's the last byte of `X86_CODE32`), the emulation time, and the number of instructions to execute. If we omit the last two parameters as in the example, Unicorn will default to emulating with infinite time and unlimited instruction count.

* Lines 32–35: Print the values of the `ECX` and `EDX` registers. We use the `reg_read` function to read register values.


To see more Python examples, check the code in the [bindings/python](https://github.com/unicorn-engine/unicorn/tree/master/bindings/python) folder. For C examples, check the code in the [sample](https://github.com/unicorn-engine/unicorn/tree/master/samples) folder.


## References

* [Unicorn Official Site](http://www.unicorn-engine.org/)
* [Quick tutorial on programming with Unicorn - with C & Python.](http://www.unicorn-engine.org/docs/)
