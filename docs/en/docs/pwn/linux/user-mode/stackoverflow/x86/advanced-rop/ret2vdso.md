# ret2VDSO

## Introduction to VDSO

What is VDSO (Virtual Dynamically-linked Shared Object)? As the name suggests, it is roughly a virtual dynamically-linked shared object, meaning it should be virtual, consistent with virtual memory — it does not physically exist on the computer. Specifically, it is a library that maps kernel-mode calls into the user address space. Why does it exist? Because some system calls are frequently used by user programs, which would result in a large amount of overhead from switching between user mode and kernel mode. Through vdso, we can significantly reduce this overhead and also achieve a better execution path. A better path here means that we don't need to use the traditional `int 0x80` for system calls — different processors have implemented different fast system call instructions:

- Intel implemented `sysenter` and `sysexit`
- AMD implemented `syscall` and `sysret`

When different processor architectures implement different instructions, compatibility issues naturally arise. Therefore, Linux implemented the vsyscall interface, which performs the specific operation based on the underlying architecture. And vsyscall is implemented within vdso.

Here, let's take a look at vdso. In Linux (kernel 2.6 or upper), running `ldd /bin/sh` will reveal a dynamic file named `linux-vdso.so.1` (older versions call it `linux-gate.so.1`), but it cannot be found on the system — that is VDSO. For example:

```shell
➜  ~ ldd /bin/sh
	linux-vdso.so.1 =>  (0x00007ffd8ebf2000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f84ff2f9000)
	/lib64/ld-linux-x86-64.so.2 (0x0000560cae6eb000)
```

In addition to fast system calls, glibc also provides VDSO support. `open()`, `read()`, `write()`, `gettimeofday()` can all directly use the implementations in VDSO, making these calls faster. New kernel features can also be deployed more quickly without affecting glibc.

Here we use Intel processors as an example for a brief explanation.

The parameter passing method for `sysenter` is the same as `int 0x80`, but we may need to set up the function prolog ourselves (using 32-bit as an example):

```asm
push ebp
mov ebp,esp
```

Additionally, if we don't provide a function prolog, we also need a gadget that can perform stack pivoting, so that we can change the stack's location.

## Principle

To be supplemented.

## Challenges

- **Defcon 2015 Qualifier fuckup**

## References

- http://man7.org/linux/man-pages/man7/vdso.7.html
- http://adam8157.info/blog/2011/10/linux-vdso/
