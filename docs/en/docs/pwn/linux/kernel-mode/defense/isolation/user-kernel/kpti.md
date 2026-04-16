# KPTI - Kernel Page Table Isolation

## Introduction

The initial main purpose of the KPTI mechanism was to mitigate KASLR bypasses and CPU side-channel attacks.

In the KPTI mechanism, the isolation between kernel-space memory and user-space memory is further enhanced.

- The page table in kernel mode includes page tables for both user-space memory and kernel-space memory.
- The page table in user mode only includes page tables for user-space memory and the necessary kernel-space memory page tables, such as memory used for handling system calls, interrupts, and other information.

![File:Kernel page-table isolation.svg](figure/476px-Kernel_page-table_isolation.svg.png)

In the x86_64 PTI mechanism, the user-space memory mapping portion in kernel mode is entirely marked as non-executable. This means that hardware that previously did not have the SMEP feature will have a feature similar to SMEP if KPTI protection is enabled. Additionally, SMAP emulation can also be introduced in a similar way, but it has not been introduced yet. Therefore, in kernels with KPTI protection enabled, if SMAP protection is not enabled, the kernel can still access user-space memory, but it cannot jump to user-space to execute shellcode.

KPTI was introduced in Linux 4.15 and was backported to Linux 4.14.11, 4.9.75, and 4.4.110.

## Development History

TODO.

## Implementation

TODO.

## Enabling and Disabling

If using a kernel started with qemu, we can add `kpti=1` to the `-append` option to enable KPTI.

If using a kernel started with qemu, we can add `nopti` to the `-append` option to disable KPTI.

## Status Check

We can check whether the KPTI mechanism is enabled through the following two methods.

```shell
/home/pwn # dmesg | grep 'page table'
[    0.000000] Kernel/User page tables isolation: enabled
/home/pwn # cat /proc/cpuinfo | grep pti
fpu_exception   : yes
flags           : ... pti smep smap
```

## Attack KPTI

The KPTI mechanism is different from SMAP and SMEP. Since it is tightly integrated with the source code, it seems impossible to disable at runtime.

### Modify Page Tables

After KPTI is enabled, all data in user-space is marked with NX permissions. However, we can consider modifying the corresponding page table permissions to grant executable permissions. When the kernel does not have smep enabled, after modifying the page table permissions, we can return to user mode and execute user-space code.

### SWITCH_TO_USER_CR3_STACK

After the KPTI mechanism is enabled, a page table switch occurs when transitioning from user mode to kernel mode; a page table switch also occurs when returning from kernel mode to user mode. Therefore, if we can control the kernel to execute the page table switching code snippet that runs when returning to user mode, we can normally return to user mode.

By analyzing the code for switching from kernel mode to user mode, we can learn that the page table switch mainly relies on the `SWITCH_TO_USER_CR3_STACK` assembly macro. Therefore, we just need to be able to call this code.

```assembly
.macro SWITCH_TO_USER_CR3_STACK	scratch_reg:req
	pushq	%rax
	SWITCH_TO_USER_CR3_NOSTACK scratch_reg=\scratch_reg scratch_reg2=%rax
	popq	%rax
.endm
.macro SWITCH_TO_USER_CR3_NOSTACK scratch_reg:req scratch_reg2:req
	ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
	mov	%cr3, \scratch_reg

	ALTERNATIVE "jmp .Lwrcr3_\@", "", X86_FEATURE_PCID

	/*
	 * Test if the ASID needs a flush.
	 */
	movq	\scratch_reg, \scratch_reg2
	andq	$(0x7FF), \scratch_reg		/* mask ASID */
	bt	\scratch_reg, THIS_CPU_user_pcid_flush_mask
	jnc	.Lnoflush_\@

	/* Flush needed, clear the bit */
	btr	\scratch_reg, THIS_CPU_user_pcid_flush_mask
	movq	\scratch_reg2, \scratch_reg
	jmp	.Lwrcr3_pcid_\@

.Lnoflush_\@:
	movq	\scratch_reg2, \scratch_reg
	SET_NOFLUSH_BIT \scratch_reg

.Lwrcr3_pcid_\@:
	/* Flip the ASID to the user version */
	orq	$(PTI_USER_PCID_MASK), \scratch_reg

.Lwrcr3_\@:
	/* Flip the PGD to the user version */
	orq     $(PTI_USER_PGTABLE_MASK), \scratch_reg
	mov	\scratch_reg, %cr3
.Lend_\@:
.endm
```

In fact, we not only want to switch page tables, but also want to return to user mode. Therefore, we also need to reuse the code in the kernel that returns to user mode. There are two main ways the kernel returns to user mode: iret and sysret. Details are described below.

#### iret

```assembly
SYM_INNER_LABEL(swapgs_restore_regs_and_return_to_usermode, SYM_L_GLOBAL)
#ifdef CONFIG_DEBUG_ENTRY
	/* Assert that pt_regs indicates user mode. */
	testb	$3, CS(%rsp)
	jnz	1f
	ud2
1:
#endif
	POP_REGS pop_rdi=0

	/*
	 * The stack is now user RDI, orig_ax, RIP, CS, EFLAGS, RSP, SS.
	 * Save old stack pointer and switch to trampoline stack.
	 */
	movq	%rsp, %rdi
	movq	PER_CPU_VAR(cpu_tss_rw + TSS_sp0), %rsp
	UNWIND_HINT_EMPTY

	/* Copy the IRET frame to the trampoline stack. */
	pushq	6*8(%rdi)	/* SS */
	pushq	5*8(%rdi)	/* RSP */
	pushq	4*8(%rdi)	/* EFLAGS */
	pushq	3*8(%rdi)	/* CS */
	pushq	2*8(%rdi)	/* RIP */

	/* Push user RDI on the trampoline stack. */
	pushq	(%rdi)

	/*
	 * We are on the trampoline stack.  All regs except RDI are live.
	 * We can do future final exit work right here.
	 */
	STACKLEAK_ERASE_NOCLOBBER

	SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi

	/* Restore RDI. */
	popq	%rdi
	SWAPGS
	INTERRUPT_RETURN

```

As we can see, by forging the following stack and jumping to `movq	%rsp, %rdi`, we can both switch page tables and return to user mode.

```
fake rax
fake rdi
RIP
CS
EFLAGS
RSP
SS
```

#### sysret

When using sysret, we first need to ensure that rcx and r11 have the following values:

```
rcx, save the rip of the code to be executed when returning to userspace
r11, save eflags
```

Then construct the following stack:

```
fake rdi
rsp, the stack of the userspace
```

Finally, jump to the following code in entry_SYSCALL_64 to return to user mode.

```assembly
	SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi

	popq	%rdi
	popq	%rsp
	swapgs
	sysretq
```

### signal handler

We can also consider registering a signal handler in user mode to execute code located in user space. With this approach, we do not need to switch page tables.

## References

- https://github.com/pr0cf5/kernel-exploit-practice/tree/master/bypass-smep#bypassing-smepkpti-via-rop
- https://outflux.net/blog/archives/2018/02/05/security-things-in-linux-v4-15/
