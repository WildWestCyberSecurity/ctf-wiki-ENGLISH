# System.map

The `System.map` file is an important file generated during the Linux kernel compilation process. It contains kernel symbols and their corresponding memory addresses. These symbols include kernel functions, variables, and other important symbols defined in the kernel. When the vmlinux we obtain is stripped, we need System.map to help us debug.
The `System.map` file is a plain text file, with each line containing three fields:

Symbol address (memory address)
Symbol type (e.g., function, variable). If lowercase, the symbol is local; if uppercase, the symbol is global (external).
Symbol name (identifier)
```
└─$ head System.map                 
0000000000000000 A VDSO32_PRELINK
0000000000000000 D __per_cpu_start
0000000000000000 D per_cpu__irq_stack_union
0000000000000000 A xen_irq_disable_direct_reloc
0000000000000000 A xen_save_fl_direct_reloc
0000000000000040 A VDSO32_vsyscall_eh_frame_size
00000000000001e7 A kexec_control_code_size
00000000000001f0 A VDSO32_NOTE_MASK
0000000000000400 A VDSO32_sigreturn
0000000000000410 A VDSO32_rt_sigreturn
```

## Adding System.map to a Stripped vmlinux in IDA

Load the following script, then select the corresponding System.map file:
```python
import idaapi

def load_system_map(file_path):
    with open(file_path, "r") as f:
        for line in f:
            parts = line.split()
            if len(parts) < 3:
                continue
            
            addr = int(parts[0], 16)
            symbol_type = parts[1]
            symbol_name = parts[2]
            
            # if symbol_type in ['T', 't', 'D', 'd', 'B', 'b']:     # If the symbol table is too large, you can selectively add symbols
            if not idaapi.add_entry(addr, addr, symbol_name, 0):
                print(f"Failed to add symbol: {symbol_name} at {hex(addr)}")
            else:
                print(f"Added symbol: {symbol_name} at {hex(addr)}")

system_map_path = idaapi.ask_file(0, "*.map", "Select System.map file")
if system_map_path:
    load_system_map(system_map_path)
else:
    print("No file selected")

```

Comparison before and after loading:
![System.map-load-diff](figure/System.map-load-diff.png)


## Directly Repairing vmlinux

If you want to directly repair vmlinux, you can refer to https://github.com/marin-m/vmlinux-to-elf

## References
https://linux.die.net/man/1/nm
