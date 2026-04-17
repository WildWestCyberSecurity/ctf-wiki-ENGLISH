# KASLR

## Introduction

In kernels with KASLR enabled, addresses such as the kernel code segment base address are shifted as a whole.

## Development History

TODO.

## Implementation

TODO.

## Enabling and Disabling

If using a kernel started with qemu, we can add `kaslr` to the `-append` option to enable KASLR.

If using a kernel started with qemu, we can add `nokaslr` to the `-append` option to disable KASLR.

## Attack

By leaking the address of a certain kernel segment, we can obtain all addresses within that segment. For example, when we leak the kernel code segment address, we know all addresses in the kernel code segment.

## References

- https://outflux.net/slides/2013/lss/kaslr.pdf
- https://bneuburg.github.io/volatility/kaslr/2017/04/26/KASLR1.html

