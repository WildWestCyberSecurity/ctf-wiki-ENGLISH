# Kernel Stack Canary

Canary is a typical detection mechanism. In the Linux kernel, the implementation of Canary is architecture-dependent, so here we introduce it from different architectures.

## x86

### Introduction

In the x86 architecture, the same Canary is used within the same task.

### Development History

TODO.

### Implementation

TODO.

### Usage

#### Enabling

When compiling the kernel, we can set the CONFIG_CC_STACKPROTECTOR option to enable this protection.

#### Disabling

We need to recompile the kernel and disable the compilation option to turn off Canary protection.

### Status Check

We can check whether Canary protection is enabled using the following methods:

1. `checksec` 
2. Manually analyze the binary file to see if there is code for saving and checking the Canary in functions

### Characteristics

As we can see, the characteristic of the Canary implementation under the x86 architecture is that the same task shares the same Canary.

### Attack

Based on the characteristics of the Canary implementation under the x86 architecture, once we leak the Canary in one system call, the Canary in all other system calls of the same task is also leaked.

## References

- https://www.workofard.com/2018/01/per-task-stack-canaries-for-arm64/
- [PESC: A Per System-Call Stack Canary Design for Linux Kernel](https://yajin.org/papers/pesc.pdf)
