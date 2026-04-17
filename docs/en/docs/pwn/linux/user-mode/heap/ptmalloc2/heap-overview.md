# Heap Overview

## What is a Heap

During program execution, the heap provides dynamically allocated memory, allowing the program to request memory of an unknown size. The heap is essentially a contiguous linear region in the program's virtual address space, and it grows from low addresses to high addresses. The part of the program that manages the heap is commonly referred to as the heap manager.

The heap manager sits between user programs and the kernel, and mainly performs the following tasks:

1. Responds to the user's memory allocation requests by requesting memory from the operating system and returning it to the user program. At the same time, to maintain efficient memory management, the kernel typically pre-allocates a large contiguous block of memory and then lets the heap manager manage this block through certain algorithms. The heap manager will only interact with the operating system again when heap space is insufficient.
2. Manages memory freed by the user. Generally speaking, freed memory is not returned directly to the operating system, but is managed by the heap manager. This freed memory can be used to serve new memory allocation requests from the user.

The early heap allocation and deallocation in Linux was implemented by Doug Lea, but when handling multiple threads in parallel, it shared the process's heap memory space. Therefore, for safety, when a thread uses the heap, a lock is acquired. However, at the same time, locking prevents other threads from using the heap, reducing the efficiency of memory allocation and deallocation. Additionally, if proper control is not maintained during multi-threaded usage, it may also affect the correctness of memory allocation and deallocation. Wolfram Gloger improved upon Doug Lea's implementation to support multi-threading, and this heap allocator is known as ptmalloc. After glibc-2.3.x, ptmalloc2 was integrated into glibc.

The heap allocator used in current standard Linux distributions is the heap allocator in glibc: ptmalloc2. ptmalloc2 primarily allocates and frees memory blocks through the malloc/free functions.

It is important to note that during memory allocation and usage, Linux has a fundamental memory management philosophy: **the system only establishes the mapping between virtual pages and physical pages when an address is actually accessed**. So although the operating system has allocated a large block of memory to the program, this block is actually only virtual memory. The system will only allocate physical pages to the user when the corresponding memory is actually used.

## Basic Heap Operations

Here we mainly introduce:

- Basic heap operations, including heap allocation, deallocation, and the system calls behind heap allocation.
- The current multi-threading support for the heap.

### malloc

In glibc's [malloc.c](https://github.com/iromise/glibc/blob/master/malloc/malloc.c#L448), the description of malloc is as follows:

```c++
/*
  malloc(size_t n)
  Returns a pointer to a newly allocated chunk of at least n bytes, or null
  if no space is available. Additionally, on failure, errno is
  set to ENOMEM on ANSI C systems.
  If n is zero, malloc returns a minumum-sized chunk. (The minimum
  size is 16 bytes on most 32bit systems, and 24 or 32 bytes on 64bit
  systems.)  On most systems, size_t is an unsigned type, so calls
  with negative arguments are interpreted as requests for huge amounts
  of space, which will often fail. The maximum supported value of n
  differs across systems, but is in all cases less than the maximum
  representable value of a size_t.
*/
```

As we can see, the malloc function returns a pointer to a memory block of the corresponding size in bytes. Additionally, the function handles some exceptional cases:

- When n=0, it returns the minimum memory block size allowed by the current system.
- When n is negative, since on most systems **size_t is an unsigned type (this is very important)**, the program will request a very large amount of memory space, but this will usually fail because the system does not have that much memory to allocate.

### free

In glibc's [malloc.c](https://github.com/iromise/glibc/blob/master/malloc/malloc.c#L465), the description of free is as follows:

```c++
/*
      free(void* p)
      Releases the chunk of memory pointed to by p, that had been previously
      allocated using malloc or a related routine such as realloc.
      It has no effect if p is null. It can have arbitrary (i.e., bad!)
      effects if p has already been freed.
      Unless disabled (using mallopt), freeing very large spaces will
      when possible, automatically trigger operations that give
      back unused memory to the system, thus reducing program footprint.
    */
```

As we can see, the free function releases the memory block pointed to by p. This memory block may have been obtained through the malloc function, or through a related function such as realloc.

Additionally, the function also handles exceptional cases:

- **When p is a null pointer, the function performs no operation.**
- When p has already been freed, freeing it again will cause unpredictable effects — this is essentially a `double free`.
- Except when disabled (via mallopt), when freeing very large memory spaces, the program will return this memory to the system in order to reduce the program's memory footprint.

### System Calls Behind Memory Allocation

Among the functions mentioned above, whether it is malloc or free, we frequently use them for dynamic memory allocation and deallocation, but they are not the functions that actually interact with the system. The system calls behind these functions are mainly the [(s)brk](http://man7.org/linux/man-pages/man2/sbrk.2.html) function and the [mmap, munmap](http://man7.org/linux/man-pages/man2/mmap.2.html) function.

As shown in the figure below, we mainly consider the operation of requesting memory blocks from the heap.

![](./figure/brk&mmap.png)

#### (s)brk

For heap operations, the operating system provides the brk function, and the glibc library provides the sbrk function. We can request memory from the operating system by increasing the size of [brk](https://en.wikipedia.org/wiki/Sbrk).

Initially, the heap's start address [start_brk](http://elixir.free-electrons.com/linux/v3.8/source/include/linux/mm_types.h#L365) and the heap's current end [brk](http://elixir.free-electrons.com/linux/v3.8/source/include/linux/mm_types.h#L365) point to the same address. Depending on whether ASLR is enabled, their specific positions will differ:

- When ASLR protection is not enabled, start_brk and brk will point to the end of the data/bss segment.
- When ASLR protection is enabled, start_brk and brk will also point to the same location, but this location is at a random offset after the end of the data/bss segment.

The specific effect is shown in the figure below (this figure is basically consistent with those widely circulated online; it is redrawn here as part of a larger diagram):

![](./figure/program_virtual_address_memory_space.png)

**Example**

```c
/* sbrk and brk example */
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main()
{
        void *curr_brk, *tmp_brk = NULL;

        printf("Welcome to sbrk example:%d\n", getpid());

        /* sbrk(0) gives current program break location */
        tmp_brk = curr_brk = sbrk(0);
        printf("Program Break Location1:%p\n", curr_brk);
        getchar();

        /* brk(addr) increments/decrements program break location */
        brk(curr_brk+4096);

        curr_brk = sbrk(0);
        printf("Program break Location2:%p\n", curr_brk);
        getchar();

        brk(tmp_brk);

        curr_brk = sbrk(0);
        printf("Program Break Location3:%p\n", curr_brk);
        getchar();

        return 0;
}
```

Note that after each operation, getchar() is called to allow us to conveniently inspect the program's actual memory mappings.

**Before the first brk call**

From the output below, we can see that no heap segment has appeared. Therefore:

- start_brk = brk = end_data = 0x804b000

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ ./sbrk
Welcome to sbrk example:6141
Program Break Location1:0x804b000
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ cat /proc/6141/maps
...
0804a000-0804b000 rw-p 00001000 08:01 539624     /home/sploitfun/ptmalloc.ppt/syscalls/sbrk
b7e21000-b7e22000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$
```

**After the first brk increase**

From the output below, we can see that the heap segment has appeared:

- start_brk = end_data = 0x804b000
- brk = 0x804c000

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ ./sbrk
Welcome to sbrk example:6141
Program Break Location1:0x804b000
Program Break Location2:0x804c000
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ cat /proc/6141/maps
...
0804a000-0804b000 rw-p 00001000 08:01 539624     /home/sploitfun/ptmalloc.ppt/syscalls/sbrk
0804b000-0804c000 rw-p 00000000 00:00 0          [heap]
b7e21000-b7e22000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$
```

Regarding the heap line:

- 0x0804b000 is the starting address of the corresponding heap.
- rw-p indicates the heap has read and write permissions, and belongs to private data.
- 00000000 is the file offset. Since this memory is not mapped from a file, it is 0.
- 00:00 is the major/minor device number. Since this memory is not mapped from a file, both are 0.
- 0 represents the inode number. Since this memory is not mapped from a file, it is 0.

#### mmap

malloc uses [mmap](http://lxr.free-electrons.com/source/mm/mmap.c?v=3.8#L1285) to create independent anonymous mapping segments. The purpose of anonymous mapping is mainly to request zero-filled memory, and this memory is used only by the calling process.

**Example**

```c++
/* Private anonymous mapping example using mmap syscall */
#include <stdio.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

void static inline errExit(const char* msg)
{
        printf("%s failed. Exiting the process\n", msg);
        exit(-1);
}

int main()
{
        int ret = -1;
        printf("Welcome to private anonymous mapping example::PID:%d\n", getpid());
        printf("Before mmap\n");
        getchar();
        char* addr = NULL;
        addr = mmap(NULL, (size_t)132*1024, PROT_READ|PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        if (addr == MAP_FAILED)
                errExit("mmap");
        printf("After mmap\n");
        getchar();

        /* Unmap mapped region. */
        ret = munmap(addr, (size_t)132*1024);
        if(ret == -1)
                errExit("munmap");
        printf("After munmap\n");
        getchar();
        return 0;
}
```

**Before executing mmap**

From the output below, we can see that currently there are only mmap segments for .so files.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ cat /proc/6067/maps
08048000-08049000 r-xp 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
08049000-0804a000 r--p 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
0804a000-0804b000 rw-p 00001000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
b7e21000-b7e22000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$
```

**After mmap**

From the output below, we can see that the requested memory has merged with the existing memory segment to form an mmap segment from b7e00000 to b7e21000.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ cat /proc/6067/maps
08048000-08049000 r-xp 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
08049000-0804a000 r--p 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
0804a000-0804b000 rw-p 00001000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
b7e00000-b7e22000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$
```

**munmap**

From the output below, we can see that the previously requested memory segment is now gone, and the memory segments have reverted to their original state.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$ cat /proc/6067/maps
08048000-08049000 r-xp 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
08049000-0804a000 r--p 00000000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
0804a000-0804b000 rw-p 00001000 08:01 539691     /home/sploitfun/ptmalloc.ppt/syscalls/mmap
b7e21000-b7e22000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/syscalls$
```

### Multi-threading Support

In the original dlmalloc implementation, when two threads simultaneously request memory, only one thread can enter the critical section to allocate memory, while the other thread must wait until no thread is in the critical section. This is because all threads share a single heap. In glibc's ptmalloc implementation, a notable improvement is the support for fast multi-threaded access. In the new implementation, all threads share multiple heaps.

Here is an example.

```c++
/* Per thread arena example. */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/types.h>

void* threadFunc(void* arg) {
        printf("Before malloc in thread 1\n");
        getchar();
        char* addr = (char*) malloc(1000);
        printf("After malloc and before free in thread 1\n");
        getchar();
        free(addr);
        printf("After free in thread 1\n");
        getchar();
}

int main() {
        pthread_t t1;
        void* s;
        int ret;
        char* addr;

        printf("Welcome to per thread arena example::%d\n",getpid());
        printf("Before malloc in main thread\n");
        getchar();
        addr = (char*) malloc(1000);
        printf("After malloc and before free in main thread\n");
        getchar();
        free(addr);
        printf("After free in main thread\n");
        getchar();
        ret = pthread_create(&t1, NULL, threadFunc, NULL);
        if(ret)
        {
                printf("Thread creation error\n");
                return -1;
        }
        ret = pthread_join(t1, &s);
        if(ret)
        {
                printf("Thread join error\n");
                return -1;
        }
        return 0;
}
```

**Before the first allocation**, there are no heap segments at all.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
b7e05000-b7e07000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

**After the first allocation**, from the output below we can see that the heap segment has been created, and it is right next to the data segment, which indicates that malloc uses the brk function behind the scenes. Also, note that although we only requested 1000 bytes, we received 0x0806c000-0x0804b000=0x21000 bytes of heap. **This shows that even though a program may only request a small amount of memory from the operating system, the OS will allocate a much larger block to the program for convenience. This avoids frequent switches between kernel mode and user mode, improving program efficiency.** We call this contiguous memory region an arena. Furthermore, we call memory requested by the main thread the main_arena. Subsequent memory requests will continue to be served from this arena until there is insufficient space. When the arena runs out of space, it can increase heap space by growing brk. Similarly, the arena can also shrink its space by reducing brk.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
After malloc and before free in main thread
...
sploitfun@sploitfun-VirtualBox:~/lsploits/hof/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804b000-0806c000 rw-p 00000000 00:00 0          [heap]
b7e05000-b7e07000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

**After the main thread frees memory**, from the output below we can see that the corresponding arena has not been reclaimed, but is instead managed by glibc. When the program requests memory again later, if glibc has sufficient managed memory, it will allocate the appropriate memory to the program according to the heap allocation algorithm.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
After malloc and before free in main thread
After free in main thread
...
sploitfun@sploitfun-VirtualBox:~/lsploits/hof/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804b000-0806c000 rw-p 00000000 00:00 0          [heap]
b7e05000-b7e07000 rw-p 00000000 00:00 0
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

**Before thread 1's malloc**, we can see that there is no heap associated with thread 1, but there is a stack associated with thread 1.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
After malloc and before free in main thread
After free in main thread
Before malloc in thread 1
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804b000-0806c000 rw-p 00000000 00:00 0          [heap]
b7604000-b7605000 ---p 00000000 00:00 0
b7605000-b7e07000 rw-p 00000000 00:00 0          [stack:6594]
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

**After thread 1's malloc**, from the output below we can see that thread 1's heap segment has been created. It is located in the memory mapping segment area and is also 132KB in size (b7500000-b7521000). This indicates that the function behind the thread's heap allocation is the mmap function. At the same time, we can see that the memory actually allocated to the program is 1MB (b7500000-b7600000). Moreover, only the 132KB portion has read and write permissions — this contiguous region is called a thread arena.

Note:

> When the user requests memory larger than 128KB, and no arena has enough space, the system will execute the mmap function to allocate the corresponding memory space. This is independent of whether the request comes from the main thread or a child thread.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
After malloc and before free in main thread
After free in main thread
Before malloc in thread 1
After malloc and before free in thread 1
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804b000-0806c000 rw-p 00000000 00:00 0          [heap]
b7500000-b7521000 rw-p 00000000 00:00 0
b7521000-b7600000 ---p 00000000 00:00 0
b7604000-b7605000 ---p 00000000 00:00 0
b7605000-b7e07000 rw-p 00000000 00:00 0          [stack:6594]
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

**After thread 1 frees memory**, from the output below we can see that freeing memory likewise does not return the memory back to the system.

```shell
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ ./mthread
Welcome to per thread arena example::6501
Before malloc in main thread
After malloc and before free in main thread
After free in main thread
Before malloc in thread 1
After malloc and before free in thread 1
After free in thread 1
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$ cat /proc/6501/maps
08048000-08049000 r-xp 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
08049000-0804a000 r--p 00000000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804a000-0804b000 rw-p 00001000 08:01 539625     /home/sploitfun/ptmalloc.ppt/mthread/mthread
0804b000-0806c000 rw-p 00000000 00:00 0          [heap]
b7500000-b7521000 rw-p 00000000 00:00 0
b7521000-b7600000 ---p 00000000 00:00 0
b7604000-b7605000 ---p 00000000 00:00 0
b7605000-b7e07000 rw-p 00000000 00:00 0          [stack:6594]
...
sploitfun@sploitfun-VirtualBox:~/ptmalloc.ppt/mthread$
```

## References

- [sploitfun](https://sploitfun.wordpress.com/archives/)
