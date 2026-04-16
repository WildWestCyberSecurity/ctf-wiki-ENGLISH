# Introduction

The main mechanisms here are:

- Default: User mode cannot directly access kernel-mode data or execute kernel-mode code
- SMEP: Kernel mode cannot execute user-mode code
- SMAP: Kernel mode cannot access user-mode data
- KPTI: User mode cannot see kernel-mode page tables; kernel mode cannot execute user-mode code (emulated)




