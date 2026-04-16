# Introduction to Packers

## What is a Packer?

A **packer** is a program in computer software that is specifically responsible for protecting software from being illegally modified or decompiled.

Packers generally run before the main program, take control, and then complete their task of protecting the software.

![](./figure/what_is_pack.png)

Since this type of program shares many functional similarities with shells in nature, following naming conventions, such programs are called **packers** (or "shells" in Chinese).

## Types of Packers

Packers are typically divided into two categories: compression packers and encryption packers.

### Compression Packers

Compression packers appeared as early as the DOS era, but at that time, due to limited computing power and excessive decompression overhead, they were not widely used.

Using compression packers can help reduce the size of PE files, hide the internal code and resources of PE files, and facilitate network transmission and storage.

Compression packers generally have two types of uses: one is purely for compressing ordinary PE files, while the other significantly transforms the source file, severely damaging the PE file header, and is often used for compressing malicious programs.

Common compression packers include: UPX, ASPack, PECompact

### Encryption Packers

Encryption packers, also known as protection packers, employ various techniques to prevent code reverse analysis. Their primary function is to protect PE files from code reverse analysis.

Since the main purpose of encryption packers is no longer to compress file resources, PE programs protected by encryption packers are usually much larger than the original files.

Currently, encryption packers are widely used in applications with high security requirements and sensitivity to cracking. At the same time, malicious programs also use them to avoid (reduce) detection and removal by antivirus software.

Common encryption packers include: ASProtector, Armadillo, EXECryptor, Themida, VMProtect

## Packer Loading Process

### Saving Entry Parameters

1.  The packed program saves the values of all registers during initialization
2.  After the packer finishes execution, it restores all register values
3.  Finally, it jumps to the original program for execution

Typically, `pushad` / `popad` and `pushfd` / `popfd` instruction pairs are used to save and restore the execution environment.

### Obtaining Required API Functions

1.  Generally, a packer's import table contains only a few API functions such as `GetProcAddress`, `GetModuleHandle`, and `LoadLibrary`
2.  If other API functions are needed, `LoadLibraryA(W)` or `LoadLibraryExA(W)` is used to map DLL files into the calling process's address space
3.  If the DLL file has already been mapped into the calling process's address space, the `GetModuleHandleA(W)` function can be called to obtain the DLL module handle
4.  Once a DLL module is loaded, `GetProcAddress` can be called to obtain the address of the imported function

### Decrypting Section Data

1.  For the purpose of protecting the source program's code and data, the various sections of the source program file are generally encrypted. During program execution, the packer decrypts these section data to allow the program to run normally
2.  The packer generally encrypts by section and decrypts by section, placing the decrypted data back at the appropriate memory locations

### Jumping Back to the Original Entry Point

1.  Before jumping back to the entry point, the original PE file's import table (IAT) is generally restored and filled in, and relocation entries are handled (mainly for DLL files)
2.  Because the packer constructs its own import table during packing, it needs to re-obtain the addresses of all functions imported from each DLL and fill them into the IAT table
3.  After completing the above work, control is transferred back to the original program, and execution continues
