# User Code Not Executable

Initially, when executing code in kernel mode, it was possible to directly execute user-mode code. If an attacker gained control of the execution flow in the kernel, they could execute code residing in user space. Since user-mode code is controllable by the attacker, it was easier to carry out attacks. To prevent such attacks, researchers proposed that when in kernel mode, user-mode code should not be executable. In the Linux kernel, the implementation of this defense measure is architecture-dependent.

## x86 - SMEP - Supervisor Mode Execution Protection

The corresponding protection mechanism in x86 is called SMEP. Bit 20 of the CR4 register is used to indicate whether SMEP protection is enabled.

![20180220141919-fc10512e-1605-1](figure/cr4.png)

### Development History

TODO.

### Implementation

TODO.

### Enabling and Disabling

#### Enabling

By default, SMEP protection is enabled.

If using a kernel started with qemu, we can add `+smep` to the `-append` option to enable SMEP.

#### Disabling

Add nosmep to the following two lines in `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"  
GRUB_CMDLINE_LINUX="initrd=/install/initrd.gz"
```

Then run `update-grub` and restart the system to disable smep.

If using a kernel started with qemu, we can add `nosmep` to the `-append` option to disable SMEP.

### Status Check

The following command can be used to check whether SMEP is enabled. If the smep string is found, it means smep protection is enabled; otherwise, it is not.

```bash
grep smep /proc/cpuinfo
```

### Attack SMEP

Setting bit 20 of the CR4 register to 0 allows us to execute user-mode code. Generally, we use 0x6f0 to set CR4, which disables both SMAP and SMEP.

The code in the kernel that modifies cr4 ultimately calls `native_write_cr4`. After we can hijack the control flow, we can execute gadgets in the kernel to modify CR4. From another perspective, there is fixed code in the kernel that modifies cr4, such as in the `refresh_pce` function and the `set_tsc_mode` function.

## ARM - PXN

TODO.

## References

- https://duasynt.com/slides/smep_bypass.pdf
- https://github.com/torvalds/linux/commit/15385dfe7e0fa6866b204dd0d14aec2cc48fc0a7
