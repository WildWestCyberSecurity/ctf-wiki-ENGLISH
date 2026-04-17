# User Data Not Accessible

If kernel mode can access user-mode data, problems can also arise. For example, after hijacking the control flow, an attacker can use stack pivoting to migrate the stack to user space, then perform ROP to further achieve privilege escalation. In the Linux kernel, the implementation of this defense measure is architecture-dependent.

## x86 - SMAP - Supervisor Mode Access Protection

### Introduction

The corresponding protection mechanism in x86 is called SMAP. Bit 21 of the CR4 register is used to indicate whether SMEP protection is enabled.

![20180220141919-fc10512e-1605-1](figure/cr4.png)

### Development History

TODO.

### Implementation

TODO.

### Enabling and Disabling

#### Enabling

By default, SMAP protection is enabled.

If using a kernel started with qemu, we can add `+smap` to the `-append` option to enable SMAP.

#### Disabling

Add nosmap to the following two lines in `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"  
GRUB_CMDLINE_LINUX="initrd=/install/initrd.gz"
```

Then run `update-grub` and restart the system to disable smap.

If using a kernel started with qemu, we can add `nosmap` to the `-append` option to disable SMAP.

### Status Check

The following command can be used to check whether SMAP is enabled. If the smap string is found, it means smap protection is enabled; otherwise, it is not.

```bash
grep smap /proc/cpuinfo
```

### Attack SMEP

Here are several approaches.

#### Setting the CR4 Register

Setting bit 21 of the CR4 register to 0 allows us to access user-mode data. Generally, we use 0x6f0 to set CR4, which disables both SMAP and SMEP.

The code in the kernel that modifies cr4 ultimately calls `native_write_cr4`. After we can hijack the control flow, we can execute the corresponding gadgets in the kernel to modify CR4. From another perspective, there is fixed code in the kernel that modifies cr4, such as in the `refresh_pce` function and the `set_tsc_mode` function.

#### copy_from/to_user

After hijacking the control flow, an attacker can call `copy_from_user` and `copy_to_user` to access user-mode memory. These two functions temporarily clear the flag that prohibits access to user-mode memory.

## ARM - PAN

TODO.
