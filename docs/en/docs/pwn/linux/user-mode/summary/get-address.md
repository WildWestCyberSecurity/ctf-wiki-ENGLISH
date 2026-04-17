# Getting Addresses

During the exploitation process, we often need to obtain the addresses of certain variables and functions in order to further the exploit. Here, I categorize the methods for obtaining addresses into the following types:

- Directly finding addresses — we can directly see the address of the corresponding symbol through reverse engineering and other means.
- Leaking addresses — we need to control the program's execution flow to leak the contents of certain symbol pointers in the program and obtain the corresponding addresses.
- Inferring addresses — we generally exploit the fact that offsets between symbols within a given segment are fixed, thereby inferring the addresses of new symbols.
- Guessing addresses — this mainly refers to cases where we need to guess the address of the corresponding symbol ourselves, which is often accompanied by brute-force enumeration.

The above methods represent a progressive way of thinking. When obtaining the addresses of relevant symbols, we should maintain this thought process.

Among the above methods, I believe there are two core ideas:

- Fully exploit the properties of the code itself. For example, the positions of certain code sections are fixed, such as the code segment position when PIE is not enabled. Another example is that the last three digits of glibc addresses are fixed.
- Fully exploit the properties of relative offsets. This is because programs are typically loaded in memory as contiguous segments, so relative offsets are often fixed.

For more specifics, see the introduction below.

## Directly Finding Addresses

The program already provides the addresses of the relevant variables or functions. In this case, we can directly proceed with exploitation.

This scenario is often applicable when PIE is not enabled in the program.

## Leaking Addresses

In the process of leaking addresses, we often need to find some sensitive pointers that store either the address of the symbol we want, or an address related to the symbol we want.

Here are a few examples.

### Leaking Variable Pointers

For example:

1. Leaking the head pointers of various bins in the main arena may allow us to obtain the address of a variable in the heap or in glibc.

### Leaking the GOT Table

Sometimes we don't necessarily need to know the exact address of a function — we can use the GOT table to jump to the corresponding function's address. Of course, if we absolutely need to know the function's address, we can use output functions like write, puts, etc. to print out the contents at the address in the GOT table (**provided that the function has already been resolved once**).

### ret2dl-resolve

When an ELF file uses dynamic linking, the GOT table employs lazy binding. When a libc function is called for the first time, the program calls _dl_runtime_resolve to resolve its address. Therefore, we can use stack overflow to construct a ROP chain that fakes the resolution of other functions (e.g., system). This is the technique we introduce in the advanced ROP section.

### /proc/self/maps

We can consider reading the program's `/proc/self/maps` to obtain base addresses related to the program.

## Inferring Addresses

In most cases, we cannot directly obtain the address of the function we want and often need to perform some address inference. As mentioned above, this relies heavily on the idea that offsets between symbols are fixed.

### Stack Related

Regarding addresses on the stack, most of the time we don't actually need the specific stack address, but we can infer the position of a stack variable relative to EBP based on the stack's addressing mode.

### Glibc Related

Here the main consideration is how to find the relevant functions in Glibc.

#### With libc

In this case, we need to leverage the characteristic that functions in libc share the same base address. For example, we can leak the base address of libc in memory through the address of __libc_start_main.

**Note: Do not choose functions that have wrappers, as this will cause the base address calculation to be incorrect.**

Common functions with wrappers include? (To be supplemented).

#### Without libc

Actually, the solution strategies for this situation fall into two categories:

- Find a way to obtain the libc.
- Find a way to directly obtain the corresponding address.

For the addresses we want to leak, we simply need their corresponding contents, so puts, write, and printf can all be used.

- puts and printf have the issue of `\x00` truncation.
- write can specify the length of the output content.

Below are some corresponding methods.

##### `pwnlib.dynelf`

The prerequisite is that we can leak the contents at any address.

- **If using the write function to leak, it's best to output the contents of many addresses at once, because we generally keep reading towards higher addresses, which may overwrite environment variables at higher addresses, causing the shell to fail to start.**

##### libc Database

```shell
# Update the database
./get
# Add an existing libc to the database
./add libc.so 
# Find all the libc's in the database that have the given names at the given addresses. 
./find function1 addr function2 addr
# Dump some useful offsets, given a libc ID. You can also provide your own names to dump.
./dump __libc_start_main_ret system dup2
```

Search the libc database for a libc that matches the addresses that have already appeared — there is a good chance it will be the same one.

You can also use the following online websites:

- [libcdb.com](http://libcdb.com)
- [libc.blukat.me](https://libc.blukat.me)

**Of course, there is also the previously mentioned https://github.com/lieanu/LibcSearcher.**

### Heap related

Regarding inferring some heap addresses, we need to know in detail how much memory has been allocated on the heap, which memory block the currently leaked address belongs to, and from there obtain the heap's base address as well as related memory addresses within the heap.

## Guessing Addresses

In some unusual situations, we may be able to use the following approaches:

- Use brute-force methods to obtain addresses. For example, in 32-bit systems, the address randomization space is relatively small.
- When a program is deployed in a special way, its different libraries may be loaded at specific locations. We can test locally and then guess the remote situation.
