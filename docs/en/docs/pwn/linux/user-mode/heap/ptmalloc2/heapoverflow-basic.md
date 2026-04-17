# Heap Overflow

## Introduction

A heap overflow refers to the situation where the number of bytes written into a heap chunk by a program exceeds the usable number of bytes of the heap chunk itself (**the reason it is "usable" rather than "user-requested" bytes is that the heap manager adjusts the number of bytes requested by the user, which also means that the number of usable bytes is always no less than the number of bytes requested by the user**), resulting in data overflow that overwrites the next heap chunk at a **physically adjacent higher address**.

It is easy to see that the basic prerequisites for a heap overflow vulnerability are:

- The program writes data to the heap.
- The size of the written data is not properly controlled.

For an attacker, a heap overflow vulnerability can at minimum cause a program to crash, and at worst allow the attacker to control the program's execution flow.

A heap overflow is a specific type of buffer overflow (others include stack overflow, BSS segment overflow, etc.). However, unlike a stack overflow, there is no return address or similar data on the heap that would allow an attacker to directly control the execution flow. Therefore, we generally cannot directly control EIP through a heap overflow. In general, the strategies for exploiting a heap overflow are:

1.  Overwrite the content of the next **physically adjacent chunk**.
    -   prev_size
    -   size, which mainly has three flag bits, as well as the true size of the chunk.
        -   NON_MAIN_ARENA 
        -   IS_MAPPED  
        -   PREV_INUSE 
        -   the True chunk size
    -   chunk content, thereby altering the program's inherent execution flow.
2.  Leverage heap mechanisms (such as unlink, etc.) to achieve arbitrary address write (Write-Anything-Anywhere) or control the content of heap chunks, thereby controlling the program's execution flow.

## Basic Example

Below we provide a simple example:

```
#include <stdio.h>

int main(void) 
{
  char *chunk;
  chunk=malloc(24);
  puts("Get input:");
  gets(chunk);
  return 0;
}
```

The main purpose of this program is to call malloc to allocate a block of heap memory, and then write a string into this heap chunk. If the input string is too long, it will overflow the chunk's region and overwrite the top chunk that follows it (in practice, puts internally calls malloc to allocate heap memory, so what gets overwritten may not actually be the top chunk).
```
0x602000:	0x0000000000000000	0x0000000000000021 <===chunk
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1 <===top chunk
0x602030:	0x0000000000000000	0x0000000000000000
0x602040:	0x0000000000000000	0x0000000000000000
```
print 'A'*100
After writing:
```
0x602000:	0x0000000000000000	0x0000000000000021 <===chunk
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x4141414141414141	0x4141414141414141 <===top chunk(has been overflowed)
0x602030:	0x4141414141414141	0x4141414141414141
0x602040:	0x4141414141414141	0x4141414141414141
```


## Brief Summary

Several important steps in heap overflow:

### Finding Heap Allocation Functions
Typically, heap memory is allocated by calling the glibc function malloc. In some cases, calloc is used for allocation. The difference between calloc and malloc is that **calloc automatically zeroes out the memory after allocation, which can be fatal for the exploitation of certain information leakage vulnerabilities**.

```
calloc(0x20);
//equivalent to
ptr=malloc(0x20);
memset(ptr,0,0x20);
```
In addition, there is another type of allocation performed via realloc. The realloc function can serve the roles of both malloc and free.
```
#include <stdio.h>

int main(void) 
{
  char *chunk,*chunk1;
  chunk=malloc(16);
  chunk1=realloc(chunk,32);
  return 0;
}
```
The operation of realloc is not as simple as its name suggests; internally it performs different operations depending on the situation:

-   When the size in realloc(ptr, size) is not equal to the size of ptr:
    -   If the requested size > original size:
        -   If the chunk is adjacent to the top chunk, directly extend this chunk to the new size.
        -   If the chunk is not adjacent to the top chunk, it is equivalent to free(ptr), malloc(new_size).
    -   If the requested size < original size:
        -   If the difference is not enough to hold a minimum chunk (32 bytes on 64-bit systems, 16 bytes on 32-bit systems), the chunk remains unchanged.
        -   If the difference can hold a minimum chunk, the original chunk is split into two parts, and the latter part is freed.
-   When the size in realloc(ptr, size) equals 0, it is equivalent to free(ptr).
-   When the size in realloc(ptr, size) equals the size of ptr, no operation is performed.

### Finding Dangerous Functions
By looking for dangerous functions, we can quickly determine whether a program may have a heap overflow, and if so, where the heap overflow occurs.

Common dangerous functions are as follows:

-   Input
    -   gets, reads an entire line directly, ignores `'\x00'`
    -   scanf
    -   vscanf
-   Output
    -   sprintf
-   String
    -   strcpy, string copy, stops at `'\x00'`
    -   strcat, string concatenation, stops at `'\x00'`
    -   bcopy

### Determining the Fill Length
This part mainly involves calculating **the distance between the address where we start writing and the address we want to overwrite**.
A common misconception is that the parameter to malloc equals the actual size of the allocated heap chunk, but in fact the size allocated by ptmalloc is aligned. This alignment is generally twice the word size — for example, 8 bytes on 32-bit systems and 16 bytes on 64-bit systems. However, for requests no larger than twice the word size, malloc will directly return a block of twice the word size, i.e., the minimum chunk. For example, on a 64-bit system, executing `malloc(0)` will return a block with a 16-byte user region.

```
#include <stdio.h>

int main(void) 
{
  char *chunk;
  chunk=malloc(0);
  puts("Get input:");
  gets(chunk);
  return 0;
}
```

```
//Depending on the system's word size, malloc will allocate 8 or 16 bytes of user space
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1
0x602030:	0x0000000000000000	0x0000000000000000
```
Note that the size of the user region is not equal to chunk_head.size; chunk_head.size = user region size + 2 * word size.

Another point, as mentioned earlier, is that the memory size requested by the user may be modified, and it is possible that the prev_size field of the next physically adjacent chunk is used to store content. Let's revisit the previous example code:
```
#include <stdio.h>

int main(void) 
{
  char *chunk;
  chunk=malloc(24);
  puts("Get input:");
  gets(chunk);
  return 0;
}
```
Looking at the code above, the chunk size we requested is 24 bytes. However, when we compile it as a 64-bit executable, the actually allocated memory will be 16 bytes rather than 24.
```
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000020fe1
```
How can 16 bytes of space hold 24 bytes of content? The answer is that it borrows the prev_size field of the next chunk. Let's take a look at the conversion between the memory size requested by the user and the memory size actually allocated by glibc.

```c
/* pad request bytes into a usable size -- internal version */
//MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1
#define request2size(req)                                                      \
    (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)                           \
         ? MINSIZE                                                             \
         : ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
```

When req=24, request2size(24)=32. After subtracting the 16-byte chunk header, the actual usable bytes for the user chunk is 16. Based on what we learned earlier, the prev_size of a chunk is only meaningful when its previous chunk is in a freed state. So the user can actually also use the prev_size field of the next chunk, which gives exactly 24 bytes. **In practice, ptmalloc allocates memory in units of double words. Taking a 64-bit system as an example, the allocated space is a multiple of 16, meaning that user-requested chunks are all 16-byte aligned.**
