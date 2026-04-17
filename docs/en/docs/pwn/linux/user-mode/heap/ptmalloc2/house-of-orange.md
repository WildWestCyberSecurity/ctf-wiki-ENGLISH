# House of Orange


## Introduction
House of Orange differs from other House of XX exploitation methods. This technique originates from a challenge of the same name in Hitcon CTF 2016. Since this exploitation method had not appeared in CTF challenges before, the series of derivative challenges that followed are collectively referred to as House of Orange.

## Overview
The exploitation of House of Orange is quite special. First, the target vulnerability must be a heap vulnerability, but what makes it unique is that there is no `free` function or any other function that releases heap chunks in the program. We know that to exploit heap vulnerabilities, we generally need to perform malloc and free operations on heap chunks. However, in a House of Orange exploitation, the `free` function is unavailable, so the core of House of Orange is to achieve the effect of `free` through vulnerability exploitation.


## Principle
As mentioned above, the core of House of Orange is to obtain a freed heap chunk (unsorted bin) without the `free` function.
The principle of this operation is simply that when the current heap's top chunk size is not sufficient to satisfy the requested allocation size, the original top chunk will be freed and placed into the unsorted bin. Through this, we can obtain unsorted bins without a `free` function.

Let's look at this process in detail. We assume the current top chunk can no longer satisfy the malloc allocation request.
First, the `malloc` call in the program will execute into libc.so's `_int_malloc` function. In `_int_malloc`, it sequentially checks whether fastbin, small bins, unsorted bin, and large bins can satisfy the allocation requirement. Due to size constraints, none of these are suitable. Next, `_int_malloc` will try to use the top chunk, but here the top chunk also cannot satisfy the allocation requirement, so the following branch will be executed:

```
/*
Otherwise, relay to handle system-dependent cases
*/
else {
      void *p = sysmalloc(nb, av);
      if (p != NULL && __builtin_expect (perturb_byte, 0))
	    alloc_perturb (p, bytes);
      return p;
}
```

At this point, ptmalloc can no longer satisfy the user's heap memory allocation request and needs to execute sysmalloc to request more space from the system.
However, for the heap there are two allocation methods: mmap and brk. We need the heap to expand via brk, after which the original top chunk will be placed in the unsorted bin.

In summary, we want to achieve brk expansion of the top chunk, but to accomplish this we need to bypass some checks in libc.
First, the malloc size must not be greater than `mmp_.mmap_threshold`:
```
if ((unsigned long)(nb) >= (unsigned long)(mp_.mmap_threshold) && (mp_.n_mmaps < mp_.n_mmaps_max))
```
If the chunk size to be allocated is greater than the mmap allocation threshold (default 128K), and the number of memory blocks currently allocated via mmap() by the process is less than the configured maximum, mmap() system call will be used to directly request memory from the operating system.

In the sysmalloc function, there is a check on the top chunk size, as follows:

```
assert((old_top == initial_top(av) && old_size == 0) ||
	 ((unsigned long) (old_size) >= MINSIZE &&
	  prev_inuse(old_top) &&
	  ((unsigned long)old_end & pagemask) == 0));
```
This checks the legitimacy of the top chunk. If this function is called for the first time, the top chunk may not have been initialized, so old_size might be 0.
If the top chunk has already been initialized, the top chunk size must be greater than or equal to MINSIZE, because the top chunk contains a fencepost, so its size must be greater than MINSIZE. Additionally, the top chunk must indicate that the previous chunk is in the inuse state, and the end address of the top chunk must be page-aligned. Furthermore, the top chunk size minus the fencepost must be smaller than the requested chunk size; otherwise, `_int_malloc()` would use the top chunk to split out the chunk.

Let's summarize the requirements for the forged top chunk size:

1. The forged size must be page-aligned
2. The size must be greater than MINSIZE (0x10)
3. The size must be less than the subsequently requested chunk size + MINSIZE (0x10)
4. The prev inuse bit of the size must be 1

After that, the original top chunk will execute `_int_free` and be smoothly placed into the unsorted bin.


## Example

Here is an example program that simulates an overflow that overwrites the top chunk's size field. We attempt to reduce the size to achieve brk expansion and place the original top chunk into the unsorted bin.

```
#include <stdlib.h>
#define fake_size 0x41

int main(void)
{
    void *ptr;
    
    ptr=malloc(0x10);
    ptr=(void *)((long long)ptr+24);
    
    *((long long*)ptr)=fake_size; // overwrite top chunk size
    
    malloc(0x60);
    
    malloc(0x60);
}
```
Here we overwrite the top chunk's size to 0x41. Then we request a heap chunk larger than this size, i.e., 0x60.
However, when we execute this example, we find that the program does not exploit successfully because the assert was not satisfied and an exception was thrown.

```
[#0] 0x7ffff7a42428 → Name: __GI_raise(sig=0x6)
[#1] 0x7ffff7a4402a → Name: __GI_abort()
[#2] 0x7ffff7a8a2e8 → Name: __malloc_assert(assertion=0x7ffff7b9e150 "(old_top == initial_top (av) && old_size == 0) || ((unsigned long) (old_size) >= MINSIZE && prev_inuse (old_top) && ((unsigned long) old_end & (pagesize - 1)) == 0)", file=0x7ffff7b9ab85 "malloc.c", line=0x95a, function=0x7ffff7b9e998 <__func__.11509> "sysmalloc")
[#3] 0x7ffff7a8e426 → Name: sysmalloc(nb=0x70, av=0x7ffff7dd1b20 <main_arena>)
```


## Correct Example

Let's go back and look at the assert conditions. We can see that all the previously listed requirements were satisfied except for the first one.

```
1. The forged size must be page-aligned
```

What does page-aligned mean? We know that modern operating systems manage memory in units of memory pages, and the typical memory page size is 4KB. So our forged size must be aligned to this size. Before overwriting, the top chunk size was 20fe1. By calculation, 0x602020 + 0x20fe0 = 0x623000, which is aligned to 0x1000 (4KB).

```
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1 <== top chunk
0x602030:	0x0000000000000000	0x0000000000000000
```
Therefore, our forged fake_size can be 0x0fe1, 0x1fe1, 0x2fe1, 0x3fe1, or other sizes aligned to 4KB. Since 0x40 does not satisfy alignment, it cannot be used for exploitation.

```
#include <stdlib.h>
#define fake_size 0x1fe1

int main(void)
{
    void *ptr;
    
    ptr=malloc(0x10);
    ptr=(void *)((long long)ptr+24);
    
    *((long long*)ptr)=fake_size;
    
    malloc(0x2000);
    
    malloc(0x60);
}
```

After allocation, we can observe that the original heap has been expanded via brk:

```
// Original heap
0x0000000000602000 0x0000000000623000 0x0000000000000000 rw- [heap]

// Expanded heap
0x0000000000602000 0x0000000000646000 0x0000000000000000 rw- [heap]
```

Our request was allocated at position 0x623010, and at the same time the original heap was placed into the unsorted bin:

```
[+] unsorted_bins[0]: fw=0x602020, bk=0x602020
 →   Chunk(addr=0x602030, size=0x1fc0, flags=PREV_INUSE)
```

Because there is a chunk in the unsorted bin, our next allocation will split this chunk:

```
 malloc(0x60);
 0x602030

[+] unsorted_bins[0]: fw=0x602090, bk=0x602090
 →   Chunk(addr=0x6020a0, size=0x1f50, flags=PREV_INUSE)
```

We can see that the allocated memory was split from the unsorted bin. The memory layout is as follows:

```
0x602030:	0x00007ffff7dd2208	0x00007ffff7dd2208 <== uncleared unsorted bin linked list
0x602040:	0x0000000000602020	0x0000000000602020
0x602050:	0x0000000000000000	0x0000000000000000
0x602060:	0x0000000000000000	0x0000000000000000
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x0000000000000000	0x0000000000000000
0x602090:	0x0000000000000000	0x0000000000001f51 <== remaining new unsorted bin after splitting
0x6020a0:	0x00007ffff7dd1b78	0x00007ffff7dd1b78
0x6020b0:	0x0000000000000000	0x0000000000000000

```


The key point of House of Orange lies precisely here. The subsequent exploitation involves knowledge of _IO_FILE, which will be covered in a separate IO_FILE chapter.
