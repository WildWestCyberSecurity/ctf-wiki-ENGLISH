# Change Others

If we can change the execution trajectory of a privileged process, we can also achieve privilege escalation. Here we consider how to change the execution trajectory of a privileged process from the following perspectives:

- Modify data
- Modify code

## Modify Data

Here are several methods to achieve privilege escalation by modifying the data used by a privileged process.

### Symbolic Links

If a root-privileged process executes a program through a symbolic link, and the symbolic link or the program it points to can be controlled by an attacker, the attacker can achieve privilege escalation.

### call_usermodehelper

`call_usermodehelper` is a mechanism for kernel threads to execute user-space applications, and the launched process has root privileges. Therefore, if we can **control which application to execute**, we can achieve privilege escalation. In the kernel, the application to be executed by `call_usermodehelper` is usually specified by a variable, so we just need to find a way to modify this variable.

It is worth mentioning that in some recent CTF challenges and a few real-world applications (such as the AOSP kernel), the CONFIG_STATIC_USERMODEHELPER option is enabled. This causes `call_usermodehelper` to only execute a specific application (controlled by a read-only string determined at compile time), thus defending against this exploitation technique.

It is easy to see that this is a typical data-flow attack method. The following are some commonly used approaches.

#### Modify modprobe_path

The basic process of achieving privilege escalation by modifying modprobe_path is as follows:

1. Obtain the address of modprobe_path.
2. Modify modprobe_path to point to a specified program.
3. Trigger the execution of `call_modprobe` to achieve privilege escalation. We can use the following methods to trigger it:
    1. (Obsolete) Execute an illegal executable file. The illegal executable file must meet certain requirements (refer to the call_usermodehelper section for details). However, a [new patch](https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa1bdca98d74472dcdb79cb948b54f63b5886c04) introduced in November 2024 removed the automatic loading of binfmt when loading illegal executables, making this method no longer applicable to Linux 6.12 and later versions.
    2. Entice the kernel to load a new kernel module (e.g., by using an unknown protocol) to trigger it.

Here we also provide a template for using modprobe_path.

```c
// step 1. modify modprobe_path to the target value

// step 2. create related file
system("echo -ne '#!/bin/sh\n/bin/cp /flag /home/pwn/flag\n/bin/chmod 777 /home/pwn/flag\ncat flag' > /home/pwn/catflag.sh");
system("chmod +x /home/pwn/catflag.sh");

// step 3. trigger modprobe using unknown executable (may be obsolete)
system("echo -ne '\\xff\\xff\\xff\\xff' > /home/pwn/dummy");
system("chmod +x /home/pwn/dummy");
system("/home/pwn/dummy");

// step 3. trigger modprobe using unknown protocol
socket(AF_INET,SOCK_STREAM,132);
```

In this process, let us focus on how to locate modprobe_path.

##### Direct Locating

Since the value of modprobe_path is deterministic, we can directly scan memory to find the corresponding string. This requires us to have the ability to scan memory.

##### Indirect Locating

Considering that the offset of modprobe_path relative to the kernel base address is fixed, we can first obtain the kernel base address, then calculate the address of modprobe_path based on the relative offset.

#### Modify poweroff_cmd

1. Modify poweroff_cmd to point to a specified program.
2. Hijack the control flow to execute `__orderly_poweroff`.

For locating poweroff_cmd, we can use methods similar to locating `modprobe_path`.

#### Modify core_pattern

1. Modify core_pattern to `|/path/to/your/program`.
2. Enable crash dumping, triggering the kernel to execute the [program after the pipe character](https://elixir.bootlin.com/linux/v6.18.1/source/fs/coredump.c#L971) in core_pattern.

Since the uppercase %P in core_pattern represents the PID of the crashing process in the host, it is also commonly used as a substitute for modprobe_path for privilege escalation in container environments.

For locating core_pattern, we can use methods similar to locating `modprobe_path`.  
For more details on privilege escalation using core_pattern, refer to [a recent kctf exp](https://github.com/google/security-research/blob/master/pocs/linux/kernelctf/CVE-2025-21836_lts/exploit/lts-6.6.75/exploit.c#L398).  

## Modify Code

At runtime, if we can modify the code executed by a root-privileged process, we can also achieve privilege escalation.

### Modify vDSO Code

The vDSO code in the kernel is mapped into all user-space processes. If a highly privileged process periodically calls functions in vDSO, we can consider modifying the corresponding function in vDSO to specific shellcode. When the highly privileged process executes the corresponding code, we can achieve privilege escalation.

In the early days, vDSO in Linux was writable. Considering this risk, Kees Cook proposed introducing `post-init read-only` data, which marks data that is no longer written after initialization as read-only, to defend against such exploitation.

Before the introduction, the raw_data corresponding to vDSO was only marked with an alignment attribute.

```c
	fprintf(outfile, "/* AUTOMATICALLY GENERATED -- DO NOT EDIT */\n\n");
	fprintf(outfile, "#include <linux/linkage.h>\n");
	fprintf(outfile, "#include <asm/page_types.h>\n");
	fprintf(outfile, "#include <asm/vdso.h>\n");
	fprintf(outfile, "\n");
	fprintf(outfile,
		"static unsigned char raw_data[%lu] __page_aligned_data = {",
		mapping_size);
```

After the introduction, the raw_data corresponding to vDSO was marked as read-only after initialization.

```c
	fprintf(outfile, "/* AUTOMATICALLY GENERATED -- DO NOT EDIT */\n\n");
	fprintf(outfile, "#include <linux/linkage.h>\n");
	fprintf(outfile, "#include <asm/page_types.h>\n");
	fprintf(outfile, "#include <asm/vdso.h>\n");
	fprintf(outfile, "\n");
	fprintf(outfile,
		"static unsigned char raw_data[%lu] __ro_after_init __aligned(PAGE_SIZE) = {",
		mapping_size);
```

The basic approach to privilege escalation by modifying vDSO is as follows:

- Locate vDSO
- Modify a specific function in vDSO to the specified shellcode
- Wait for the shellcode to be triggered

Here we focus on how to locate vDSO.

#### Locating in IDA

Here we introduce how to find the location of vDSO in vmlinux.

1. Locate the address of the init_vdso function in IDA

```c
__int64 init_vdso()
{
  init_vdso_image(&vdso_image_64 + 0x20000000);
  init_vdso_image(&vdso_image_x32 + 0x20000000);
  cpu_maps_update_begin();
  on_each_cpu((char *)startup_64 + 0x100003EA0LL, 0LL, 1LL);
  _register_cpu_notifier(&sdata + 536882764);
  cpu_maps_update_done();
  return 0LL;
}
```

2. We can see `vdso_image_64` and `vdso_image_x32`. Taking `vdso_image_64` as an example, click on the address of this variable

```
.rodata:FFFFFFFF81A01300                 public vdso_image_64
.rodata:FFFFFFFF81A01300 vdso_image_64   dq offset raw_data      ; DATA XREF: arch_setup_additional_pages+18↑o
.rodata:FFFFFFFF81A01300                                         ; init_vdso+1↓o
```

3. Click `raw_data` to find out the address of the 64-bit vDSO in the kernel image. As we can see, vDSO is indeed page-aligned.

```
.data:FFFFFFFF81E04000 raw_data        db  7Fh ;              ; DATA XREF: .rodata:vdso_image_64↑o
.data:FFFFFFFF81E04001                 db  45h ; E
.data:FFFFFFFF81E04002                 db  4Ch ; L
.data:FFFFFFFF81E04003                 db  46h ; F
```

From the last symbol, we can also directly use `raw_data` to find vDSO.

#### Locating in Memory

##### Direct Locating

vDSO is actually an ELF file with an ELF header. At the same time, exported function strings are stored at specific locations in vDSO. Therefore, we can scan memory based on these two characteristics to locate the position of vDSO.

##### Indirect Locating

Considering that the offset of vDSO relative to the kernel base address is fixed, we can first obtain the kernel base address, then calculate the address of vDSO based on the relative offset.

#### References

- https://lwn.net/Articles/676145/
- https://lwn.net/Articles/666550/
- https://sam4k.com/like-techniques-modprobe_path/
- https://theori.io/blog/reviving-the-modprobe-path-technique-overcoming-search-binary-handler-patch
- https://github.com/google/security-research/blob/master/pocs/linux/kernelctf/CVE-2025-21836_lts/exploit/lts-6.6.75/exploit.c
