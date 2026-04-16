# Misc

## __ro_after_init

### Introduction

There is a lot of data in the Linux kernel that is only initialized during the `__init` phase and is never changed afterwards. Memory marked with `__ro_after_init` cannot be modified again after the init phase ends.

### Attack

We can use `set_memory_rw(unsigned long addr, int numpages)` to modify the permissions of the corresponding pages.

## mmap_min_addr

mmap_min_addr is used to counter NULL Pointer Dereference. It specifies the lowest virtual memory address that a user process can use through mmap.

## References

- https://lwn.net/Articles/676145/
- https://lwn.net/Articles/666550/
- https://lore.kernel.org/patchwork/patch/621386/
