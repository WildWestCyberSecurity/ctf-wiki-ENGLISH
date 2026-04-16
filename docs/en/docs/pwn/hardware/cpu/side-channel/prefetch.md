# prefetch side-channel attack

The Prefetch Side-Channel Attack is an auxiliary attack technique proposed by [Daniel Gruss](https://gruss.cc/) in the paper _[Prefetch Side-Channel Attacks: Bypassing SMAP and Kernel ASLR](https://gruss.cc/files/prefetch.pdf)_. This attack method exploits hardware design weaknesses in the `prefetch` family of instructions in Intel CPUs, leaking memory-related information by comparing the timing differences of executing `prefetch` instructions on different virtual addresses, and bypassing protections such as KASLR.

The high-speed operation of CPUs is highly dependent on _speculative execution_ (i.e., predicting subsequent branches and executing corresponding instructions before conditional branch resolution). Data prefetching, based on this idea, speculatively loads data into the cache. This can be accomplished through hardware (transient execution) or software (instructions, though they may be ignored by the CPU). Intel CPUs have five prefetch instructions: `prefetch0`, `prefetch1`, `prefetch2`, `prefetchchnta`, and `prefetchw`, used to proactively tell the CPU that certain memory may be accessed soon. ARMv8-A CPUs also support a similar prefetch instruction `PRFM`.

## Building Attack Primitives

The Prefetch side-channel attack exploits the following two properties of the `prefetch` instruction:

- **Property 1**: The execution time of the prefetch instruction depends on the state of various internal CPU caches.
- **Property 2**: The prefetch instruction does not require any privilege checks.

### Translation-level oracle

> Under construction.

###  Address-translation oracle

Since the prefetch instruction does not require any privilege checks, an attacker can execute the `prefetch` instruction on any virtual address, **including unmapped addresses and kernel addresses**. From this, we can verify whether two virtual addresses $p$ and $\overline{p}$ map to the same physical address through the following steps:

1. Flush address $p$.

2. Prefetch the (inaccessible) address $\overline{p}$.

3. Reload address $p$.

If the two virtual addresses map to the same physical address, the prefetch instruction executed on address $\overline{p}$ in step 2 will cause a high probability of a cache hit in step 3. In this case, the execution time of step 3 will be much less than in the case of a cache miss.

Similarly, based on the execution time of the prefetch instruction (property 1), we can determine **whether the target address p exists in the cache**:

1. Flush address $p$.

2. Execute a function or system call.

3. Prefetch address $p$.

If address $\overline{p}$ accessed in step 2 maps to the same physical page as address $p$, the execution time of step 3 will be much less than in the case of a cache miss. From this, we can determine whether two virtual addresses $p$ and $\overline{p}$ map to the same physical address. However, in this case, the attacker cannot know $\overline{p}$, but can determine that $p$ is used by the function or system call.

## Translation-level Recovery Attack

> Under construction.

## Address-Translation Attack

Modern operating system kernels typically have a complete linear mapping of the physical memory space. Therefore, an attacker can use the `Address-translation oracle` to brute-force the kernel address space address $\overline{p}$ corresponding to the user address space address $p$, and use property 1 to brute-force forward to obtain the base virtual address of the linear mapping of the physical address space in the kernel address space (for Linux, this region starts at `page_offset_base`). Since virtual addresses before the start of the physical address linear mapping region do not have mappings to corresponding physical pages, the timing differences of the prefetch instruction can be observed.

## KASLR bypass

We use a variant of the `Address-translation oracle` to bypass KASLR. Instead of searching for virtual addresses that map to the same physical page, we determine whether a virtual address $p$ is used by a system call through the following method:

1. Flush all caches (accomplished by accessing a sufficiently large buffer).

2. Execute the system call; at this point, the corresponding pages will be loaded into the cache.

3. Measure the execution time of a set of prefetch instructions to determine whether virtual address $p$ is used by the system call.

Through this method, we can determine the virtual address of the corresponding system call, thereby bypassing KASLR.

### KASLR bypass with KPTI enabled

When KPTI is enabled, the page tables used by user-mode programs have almost no mappings for kernel memory. However, **mappings for the system call entry function still exist, leaving an opening for the prefetch side-channel attack**. Since the system call entry function also resides in the kernel code segment, we can use the prefetch side-channel attack to obtain the virtual address where the system call entry function is mapped, thereby bypassing KASLR.

> See [EntryBleed: Breaking KASLR under KPTI with Prefetch (CVE-2022-4543)](https://www.willsroot.io/2022/12/entrybleed.html).

## Example: TCTF2021 Final - kbrop

> Under construction.

## REFERENCE

[Prefetch Side-Channel Attacks: Bypassing SMAP and Kernel ASLR](https://gruss.cc/files/prefetch.pdf)

[EntryBleed: Breaking KASLR under KPTI with Prefetch (CVE-2022-4543)](https://www.willsroot.io/2022/12/entrybleed.html)
