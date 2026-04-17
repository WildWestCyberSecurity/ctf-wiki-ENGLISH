# Introduction to Software Reverse Engineering

## Definition

> Reverse engineering, also called back engineering, is the process by which a man-made object is deconstructed to reveal its designs, architecture, or to extract knowledge from the object;       ------  from [wikipedia](https://en.wikipedia.org/wiki/Reverse_engineering)

Software code reverse engineering mainly refers to the reverse disassembly and analysis of software structure, flow, algorithms, and code.

## Application Areas

Mainly applied in software maintenance, software cracking, vulnerability discovery, and malicious code analysis.

## Reverse Engineering in CTF Competitions

> Involves multiple programming technologies on Windows, Linux, and Android platforms. Requires using common tools to perform reverse analysis on source code and binary files, mastering reverse analysis of Android mobile application APK files, and understanding encryption/decryption, kernel programming, algorithms, anti-debugging, and code obfuscation techniques.
> ------ "National College Student Information Security Competition Guide"

### Requirements

-   Familiarity with related knowledge such as operating systems, assembly language, encryption/decryption, etc.
-   Rich programming experience in multiple high-level languages
-   Familiarity with compilation principles of multiple compilers
-   Strong program comprehension and reverse analysis capabilities

## Standard Reverse Engineering Process

1.  Use static analysis tools like `strings/file/binwalk/IDA` to collect information, and search `google/github` based on this static information
2.  Study the program's protection methods, such as code obfuscation, packers, and anti-debugging techniques, and find ways to defeat or bypass the protections
3.  Disassemble the target software and quickly locate the key code for analysis
4.  Combine with dynamic debugging to verify initial hypotheses, and understand the program's functionality during the analysis process
5.  Based on the program's functionality, write corresponding scripts to solve for the flag

### Tips for Locating Key Code

1. Analyze the control flow

    The control flow can be viewed via IDA's generated Control Flow Graph (CFG). Follow branches, loops, and function calls to read and analyze the disassembly code block by block.

2. Use data and code cross-references

    For example, output prompt strings can be traced to their call locations through data cross-references, thereby finding key code. For code cross-references, for instance, GUI programs that obtain user input will use corresponding Windows API functions, and we can find key code through these API function call locations.

### Reverse Engineering Tips

1. Coding style

    Every programmer has a different coding style. Those familiar with development design patterns can more quickly analyze function module functionality.

2. Locality principle

    When developing programs, programmers tend to write functionally related code or data in the same location. This pattern is also visible in disassembly code, so during analysis you can examine functions and data near the key code.

3. Code reuse

    Code reuse is very common, and the largest source code repository GitHub is the primary source. During analysis, you can search for certain features (such as strings, coding style, etc.) on GitHub, which may reveal similar code and help recover missing symbol information.

4. Seven parts reverse, three parts guessing

    Reasonable guessing can often yield twice the result with half the effort. When encountering suspicious functions whose internal logic is unclear, try guessing their functionality based on available clues, and continue analyzing based on those guesses. Through continuous guessing and verification, you may get closer to the truth of the code.

5. Distinguishing code

    When you get disassembled code, you must be able to distinguish which code was written by humans and which was automatically added by the compiler. Among human-written code, you need to identify which are library function code and which was written by the challenge author, and how the compiler has optimized the author's code. It's very important not to spend time on code not written by the challenge author. If you spend a long time wandering through library functions, it's not only a terrible experience but also completely ineffective.

6. Patience

    In any case, given enough time, a program can always be thoroughly analyzed. But you shouldn't give up the analysis too early either. Believe that you can break through the problem through patient, methodical analysis.

### Dynamic Analysis

The purpose of dynamic analysis is to verify your deductions or understand program functionality by observing output information (registers, memory changes, program output) during program execution after locating key code.

Main methods include: debugging, symbolic execution, and taint analysis.

### Algorithm and Data Structure Identification

-   Common algorithm identification

Such as encryption algorithms like `Tea/XTea/XXTea/IDEA/RC4/RC5/RC6/AES/DES/IDEA/MD5/SHA256/SHA1`, and traditional algorithms like big number arithmetic, shortest path, etc.

-   Common data structure identification

Identification of advanced data structures such as graphs, trees, and hash tables in assembly code.


### Code Obfuscation

For example, using tools and techniques like `OLLVM`, `movfuscator`, `junk code`, `virtualization`, and `SMC` to obfuscate code, making program analysis very difficult.

Correspondingly, there are also de-obfuscation techniques, whose main purpose is to recover the control flow. For example, `emulation` and `symbolic execution`.

### Packers

There are many types of packers. Simple compression packers can be categorized as follows:

-   unpack -> execute

    Directly decompress all program code into memory and then continue executing the program code

-   unpack -> execute -> unpack -> execute ...

    Decompress part of the code, then decompress and execute progressively

-   unpack -> [decoder | encoded code] -> decode -> execute

    Program code is encoded. After decompression, a function runs to decode and execute the actual program code

There are also related methods for unpacking, such as `single-step debugging`, `ESP law`, and so on.

### Anti-Debugging

Anti-debugging aims to prevent programs from being debugged and analyzed by detecting debuggers and other methods. For example, using API functions like `IsDebuggerPresent` to detect debuggers, using `SEH exception handling`, timing checks, and other methods. Protection can also be achieved by overwriting debug ports, self-debugging, and other methods.

## Non-Standard Reverse Engineering Approaches

Non-standard reverse engineering challenges cover an extremely broad range and can involve files of any architecture and format.

-   lua/python/java/lua-jit/haskell/applescript/js/solidity/webassembly/etc..
-   firmware/raw bin/etc..
-   chip8/avr/clemency/risc-v/etc.

However, reverse engineering methodology is not afraid of these unknown platform formats. When encountering such non-standard challenges, we have some basic procedures that can be universally applied.

### Preparation

-   Read documentation. The fastest way to learn a platform/language is to read the official documentation.
-   Official tools. Tools provided or recommended officially are necessarily the most suitable tools.
-   Tutorials. In the field of reverse engineering, many predecessors may have written tutorials specifically targeting that platform/language, so you can also quickly absorb the knowledge within.

### Finding Tools

Mainly look for `file parsing tools`, `disassemblers`, `debuggers`, and `decompilers`. Among these, a `disassembler` is essential, a `debugger` also contains corresponding disassembly functionality, and for a `decompiler`, you'll have to rely on luck.

In summary, finding tools comes down to: Google is your best friend. Making good use of Google search syntax and keyword searches can help you find suitable tools faster and better.
