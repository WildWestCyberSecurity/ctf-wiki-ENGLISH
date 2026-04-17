# FGKASLR

## Introduction

Given the shortcomings of KASLR, researchers implemented FGKASLR. FGKASLR builds upon KASLR's base address randomization by rearranging kernel code at function granularity during load time.

## Implementation

The implementation of FGKASLR is relatively simple, with modifications mainly in two parts. Currently, FGKASLR only supports the x86_64 architecture.

### Compilation Phase

FGKASLR utilizes GCC's compilation option `-ffunction-sections` to place different functions in the kernel into different sections. During compilation, any function written in C and not in a special input section will be a separate section; code written in assembly will be in a unified section.

The compiled vmlinux retains all section headers so that the address range of each function can be known. Additionally, FGKASLR has an extended table of relocation addresses. With these two sets of information, the kernel can rearrange functions out of order after decompression.

The first segment of the final binary contains a merged section (merged from several functions) and several other functions that each constitute their own section.

### Loading Phase

After decompressing the kernel, the preserved symbol information is first checked, then the `.text.*` sections that need to be randomized are located. The first merged section (`.text`) is skipped and not randomized. The addresses of subsequent sections are randomized but will still be adjacent to the `.text` section. At the same time, FGKASLR modified the existing code for updating relocation addresses, considering not only the offset relative to the load address, but also the position where the function section is to be moved.

To hide the new memory layout, symbols in /proc/kallsyms are arranged in random order. Before version v4, symbols in this file were arranged in alphabetical order.

By analyzing the code, we can see that in the `layout_randomized_image` function, the sections that will ultimately be randomized are calculated and stored in sections.

```c
	/*
	 * now we need to walk through the section headers and collect the
	 * sizes of the .text sections to be randomized.
	 */
	for (i = 0; i < shnum; i++) {
		s = &sechdrs[i];
		sname = secstrings + s->sh_name;

		if (s->sh_type == SHT_SYMTAB) {
			/* only one symtab per image */
			if (symtab)
				error("Unexpected duplicate symtab");

			symtab = malloc(s->sh_size);
			if (!symtab)
				error("Failed to allocate space for symtab");

			memcpy(symtab, output + s->sh_offset, s->sh_size);
			num_syms = s->sh_size / sizeof(*symtab);
			continue;
		}

		if (s->sh_type == SHT_STRTAB && i != ehdr->e_shstrndx) {
			if (strtab)
				error("Unexpected duplicate strtab");

			strtab = malloc(s->sh_size);
			if (!strtab)
				error("Failed to allocate space for strtab");

			memcpy(strtab, output + s->sh_offset, s->sh_size);
		}

		if (!strcmp(sname, ".text")) {
			if (text)
				error("Unexpected duplicate .text section");

			text = s;
			continue;
		}

		if (!strcmp(sname, ".data..percpu")) {
			/* get start addr for later */
			percpu = s;
			continue;
		}

		if (!(s->sh_flags & SHF_ALLOC) ||
		    !(s->sh_flags & SHF_EXECINSTR) ||
		    !(strstarts(sname, ".text")))
			continue;

		sections[num_sections] = s;

		num_sections++;
	}
	sections[num_sections] = NULL;
	sections_size = num_sections;
```

As we can see, only sections that simultaneously satisfy the following conditions will participate in randomization:

- The section name starts with .text
- Section flags include `SHF_ALLOC`
- Section flags include `SHF_EXECINSTR` 

Therefore, through the following command, we can determine that:

- __ksymtab does not participate in randomization
- .data does not participate in randomization

```
> readelf --section-headers -W vmlinux| grep -vE " .text|AX"
...
  [36106] .rodata           PROGBITS        ffffffff81c00000 e1e000 382241 00  WA  0   0 4096
  [36107] .pci_fixup        PROGBITS        ffffffff81f82250 11a0250 002ed0 00   A  0   0 16
  [36108] .tracedata        PROGBITS        ffffffff81f85120 11a3120 000078 00   A  0   0  1
  [36109] __ksymtab         PROGBITS        ffffffff81f85198 11a3198 00b424 00   A  0   0  4
  [36110] __ksymtab_gpl     PROGBITS        ffffffff81f905bc 11ae5bc 00dab8 00   A  0   0  4
  [36111] __ksymtab_strings PROGBITS        ffffffff81f9e074 11bc074 027a82 01 AMS  0   0  1
  [36112] __init_rodata     PROGBITS        ffffffff81fc5b00 11e3b00 000230 00   A  0   0 32
  [36113] __param           PROGBITS        ffffffff81fc5d30 11e3d30 002990 00   A  0   0  8
  [36114] __modver          PROGBITS        ffffffff81fc86c0 11e66c0 000078 00   A  0   0  8
  [36115] __ex_table        PROGBITS        ffffffff81fc8740 11e6738 001c50 00   A  0   0  4
  [36116] .notes            NOTE            ffffffff81fca390 11e8388 0001ec 00   A  0   0  4
  [36117] .data             PROGBITS        ffffffff82000000 11ea000 215d80 00  WA  0   0 8192
  [36118] __bug_table       PROGBITS        ffffffff82215d80 13ffd80 01134c 00  WA  0   0  1
  [36119] .vvar             PROGBITS        ffffffff82228000 14110d0 001000 00  WA  0   0 16
  [36120] .data..percpu     PROGBITS        0000000000000000 1413000 02e000 00  WA  0   0 4096
  [36122] .rela.init.text   RELA            0000000000000000 149eec0 000180 18   I 36137 36121  8
  [36124] .init.data        PROGBITS        ffffffff822b6000 14a0000 18d1a0 00  WA  0   0 8192
  [36125] .x86_cpu_dev.init PROGBITS        ffffffff824431a0 162d1a0 000028 00   A  0   0  8
  [36126] .parainstructions PROGBITS        ffffffff824431c8 162d1c8 01e04c 00   A  0   0  8
  [36127] .altinstructions  PROGBITS        ffffffff82461218 164b214 003a9a 00   A  0   0  1
  [36129] .iommu_table      PROGBITS        ffffffff82465bb0 164fbb0 0000a0 00   A  0   0  8
  [36130] .apicdrivers      PROGBITS        ffffffff82465c50 164fc50 000038 00  WA  0   0  8
  [36132] .smp_locks        PROGBITS        ffffffff82468000 1651610 007000 00   A  0   0  4
  [36133] .data_nosave      PROGBITS        ffffffff8246f000 1658610 001000 00  WA  0   0  4
  [36134] .bss              NOBITS          ffffffff82470000 165a000 590000 00  WA  0   0 4096
  [36135] .brk              NOBITS          ffffffff82a00000 1659610 02c000 00  WA  0   0  1
  [36136] .init.scratch     PROGBITS        ffffffff82c00000 1659620 400000 00  WA  0   0 32
  [36137] .symtab           SYMTAB          0000000000000000 1a59620 30abd8 18     36138 111196  8
  [36138] .strtab           STRTAB          0000000000000000 1d641f8 219a29 00      0   0  1
  [36139] .shstrtab         STRTAB          0000000000000000 1f7dc21 0ed17b 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

## Performance Overhead

The performance impact of FGKASLR mainly comes from two phases: boot and runtime.

### Boot Phase

During the boot phase, FGKASLR:

- Needs to parse the kernel ELF file to obtain the sections to be randomized.
- Calls a random number generator to determine the address where each section needs to be stored and performs layout.
- Copies the originally decompressed kernel to another location to avoid memory corruption.
- Increases the number of times the kernel needs to be relocated.
- Needs to check whether each address that needs relocation is in a randomized section, and if so, adjust to a new offset.
- Rearranges data tables that need to be sorted by address.

On a modern system, booting a test VM takes approximately 1 second.

### Runtime Phase

The overhead during the runtime phase mainly depends on the specific workload. However, since previously adjacent functions may be placed at different addresses due to randomization, overall performance is expected to decrease somewhat.

## Memory Overhead

During the boot phase, FGKASLR requires more heap memory. Therefore, FGKASLR may not be suitable for systems with smaller memory. This memory is freed after the kernel is decompressed.

## Binary Size Impact

FGKASLR introduces additional section header information, which increases the size of the vmlinux file. Under standard configuration, the vmlinux size increases by about 3%. The compressed image size increases by approximately 15%.

## Enabling and Disabling

### Enabling

To enable FGKASLR for the kernel, you need to enable the `CONFIG_FG_KASLR=y` option.

FGKASLR also supports module randomization. Although FGKASLR only supports the kernel on the x86_64 architecture, this feature can support modules on other architectures. We can use `CONFIG_MODULE_FG_KASLR=y` to enable this feature.

### Disabling

Using `nokaslr` on the command line to disable KASLR also disables FGKASLR. Of course, we can use `nofgkaslr` separately to disable only FGKASLR.

## Shortcomings

Based on the characteristics of FGKASLR, we can identify the following weaknesses:

- Function-granularity randomization means that if an address within a function is known, the relative addresses within that function are also known.
- The `.text` section does not participate in function randomization. Therefore, once an address within it is known, all addresses in that section can be obtained. Interestingly, the entry code for system calls is in this section, mainly because this code is written in assembly. Additionally, this section has some useful gadgets:
    - swapgs_restore_regs_and_return_to_usermode — this code can help us bypass KPTI protection
    - memcpy — memory copy
    - sync_regs — can move RAX into RDI
- The offset of `__ksymtab` relative to the kernel image is fixed. Therefore, if we can leak data, we can leak out other symbol addresses, such as prepare_kernel_cred and commit_creds. The specific method is as follows:
    - Obtain the __ksymtab address based on the kernel image address
    - Obtain the address of the corresponding symbol record entry based on __ksymtab
    - Obtain the address of the corresponding symbol based on the content of the symbol record entry
- The offset of the data section relative to the kernel image is also fixed. Therefore, after obtaining the kernel image base address, the addresses of data in the data section can be calculated. Some data in this section worth special attention:
    - modprobe_path

### __ksymtab Format

The format of each record entry's name in __ksymtab is `__ksymtab_func_name`. Taking `prepare_kernel_cred` as an example, the corresponding record entry name is `__ksymtab_prepare_kernel_cred`. Therefore, we can directly find the corresponding location in IDA using this name, as follows:

```assembly
__ksymtab:FFFFFFFF81F8D4FC __ksymtab_prepare_kernel_cred dd 0FF5392F4h
__ksymtab:FFFFFFFF81F8D500                 dd 134B2h
__ksymtab:FFFFFFFF81F8D504                 dd 1783Eh
```

The structure of each `__ksymtab` entry is:

```c
struct kernel_symbol {
	int value_offset;
	int name_offset;
	int namespace_offset;
};
```

The first field records the offset of the relocation entry relative to the current address. So the address of `prepare_kernel_cred` should be `0xFFFFFFFF81F8D4FC-(2**32-0xFF5392F4)=0xffffffff814c67f0`. This is indeed the case.

```assembly
.text.prepare_kernel_cred:FFFFFFFF814C67F0                 public prepare_kernel_cred
.text.prepare_kernel_cred:FFFFFFFF814C67F0 prepare_kernel_cred proc near           ; CODE XREF: sub_FFFFFFFF814A5ED5+52↑p
```

## References

- https://lwn.net/Articles/832434/
- https://github.com/kaccardi/linux/compare/fg-kaslr
- https://elixir.bootlin.com/linux/latest/source/include/linux/export.h#L60
- https://www.youtube.com/watch?v=VcqhJKfOcx4
- https://www.phoronix.com/scan.php?page=article&item=kaslr-fgkaslr-benchmark&num=1
