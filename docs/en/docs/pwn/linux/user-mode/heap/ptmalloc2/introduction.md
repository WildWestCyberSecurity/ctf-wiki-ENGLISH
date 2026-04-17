# Heap Exploitation

In this chapter, we will introduce the topic following these steps:

1. Introduce the macro-level operations of heap dynamic memory allocation that we are familiar with
2. Introduce the data structures used to achieve these operations
3. Introduce the specific operations that implement heap allocation and deallocation using these data structures
4. Introduce various heap exploitation techniques from basic to advanced

For different applications, since memory requirements and other characteristics vary, there are currently many heap implementations, as listed below:

```text
dlmalloc  – General purpose allocator
ptmalloc2 – glibc
jemalloc  – FreeBSD and Firefox
tcmalloc  – Google
libumem   – Solaris
```

Here we will mainly focus on the heap implementation in glibc. If time permits in the future, we will continue to introduce other heap implementations and their exploitation techniques.

The main references for this section are listed below. Much of the content will be consistent with the references, and we will not note this individually going forward.

- [black hat heap exploitation](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)
- [github heap exploition](https://heap-exploitation.dhavalkapil.com/)
- [sploitfun](https://sploitfun.wordpress.com/archives/)
- glibc source code
- For more references, please see the files in the ref directory
