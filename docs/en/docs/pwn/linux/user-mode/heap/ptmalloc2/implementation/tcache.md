# tcache

tcache is a technique introduced after glibc 2.26 (ubuntu 17.10) (see [commit](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc)), with the purpose of improving heap management performance. However, while improving performance, many security checks were sacrificed, which also led to many new exploitation methods.

> Main references include glibc source code, angelboy's slides, and tukan.farm. Links are all provided at the end.

## Related Structures

tcache introduced two new structures, `tcache_entry` and `tcache_perthread_struct`.

This is actually very similar to fastbin, but also different.

### tcache_entry

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#tcache_entry)

```C
/* We overlay this structure on the user-data portion of a chunk when
   the chunk is stored in the per-thread cache.  */
typedef struct tcache_entry
{
  struct tcache_entry *next;
} tcache_entry;
```

`tcache_entry` is used to link free chunk structures, where the `next` pointer points to the next chunk of the same size.

It should be noted that the next pointer here points to the chunk's user data, while fastbin's fd points to the address at the beginning of the chunk.

Moreover, tcache_entry reuses the user data portion of free chunks.

### tcache_perthread_struct

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#tcache_perthread_struct)
```C
/* There is one of these for each thread, which contains the
   per-thread cache (hence "tcache_perthread_struct").  Keeping
   overall size low is mildly important.  Note that COUNTS and ENTRIES
   are redundant (we could have just counted the linked list each
   time), this is for performance reasons.  */
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;

# define TCACHE_MAX_BINS                64

static __thread tcache_perthread_struct *tcache = NULL;
```

Each thread maintains a `tcache_perthread_struct`, which is the management structure for the entire tcache. It has `TCACHE_MAX_BINS` counters and `TCACHE_MAX_BINS` tcache_entry items, where

- `tcache_entry` links free chunks (after free) of the same size using a singly linked list, which is similar to fastbin in this regard.
- `counts` records the number of free chunks on the `tcache_entry` chain, with a maximum of 7 chunks per chain.

A diagram roughly looks like:

![](https://i0.wp.com/tvax1.sinaimg.cn/large/006AWYXBly1fw87zlnrhtj30nh0ciglz.jpg)


## Basic Working Mechanism
- On the first malloc, memory is first allocated to store `tcache_perthread_struct`.
- When freeing memory with size smaller than small bin size:
  - Before tcache, it would be placed into fastbin or unsorted bin
  - After tcache:
    - First placed into the corresponding tcache until the tcache is full (default is 7)
    - After the tcache is full, subsequently freed memory is placed into fastbin or unsorted bin as before
    - Chunks in tcache are not consolidated (inuse bit is not cleared)
- When allocating memory with size within tcache range:
  - First take chunks from tcache until tcache is empty
  - After tcache is empty, search from bins
  - When tcache is empty, if there are chunks of matching size in `fastbin/smallbin/unsorted bin`, chunks from `fastbin/smallbin/unsorted bin` will first be moved to tcache until it is full. Then chunks are taken from tcache; therefore the order of chunks in bins and tcache will be reversed

## Source Code Analysis

Next, let's analyze tcache from the source code perspective.

### __libc_malloc
On the first malloc, it will enter `MAYBE_INIT_TCACHE ()`

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#3010)
```C
void *
__libc_malloc (size_t bytes)
{
    ......
    ......
#if USE_TCACHE
  /* int_free also calls request2size, be careful to not pad twice.  */
  size_t tbytes;
  // Calculate the actual chunk size based on the malloc parameter and compute the corresponding tcache index
  checked_request2size (bytes, tbytes);
  size_t tc_idx = csize2tidx (tbytes);

  // Initialize tcache
  MAYBE_INIT_TCACHE ();
  DIAG_PUSH_NEEDS_COMMENT;
  if (tc_idx < mp_.tcache_bins  // The idx calculated from size is within the valid range
      /*&& tc_idx < TCACHE_MAX_BINS*/ /* to appease gcc */
      && tcache
      && tcache->entries[tc_idx] != NULL) // tcache->entries[tc_idx] has chunks
    {
      return tcache_get (tc_idx);
    }
  DIAG_POP_NEEDS_COMMENT;
#endif
    ......
    ......
}
```

### __tcache_init()
Among these, `MAYBE_INIT_TCACHE ()` calls `tcache_init()` when tcache is empty (i.e., the first malloc). Let's look directly at `tcache_init()`

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#tcache_init)

```C
tcache_init(void)
{
  mstate ar_ptr;
  void *victim = 0;
  const size_t bytes = sizeof (tcache_perthread_struct);
  if (tcache_shutting_down)
    return;
  arena_get (ar_ptr, bytes); // Find an available arena
  victim = _int_malloc (ar_ptr, bytes); // Allocate a chunk of sizeof(tcache_perthread_struct) size
  if (!victim && ar_ptr != NULL)
    {
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }
  if (ar_ptr != NULL)
    __libc_lock_unlock (ar_ptr->mutex);
  /* In a low memory situation, we may not be able to allocate memory
     - in which case, we just keep trying later.  However, we
     typically do this very early, so either there is sufficient
     memory, or there isn't enough memory to do non-trivial
     allocations anyway.  */
  if (victim) // Initialize tcache
    {
      tcache = (tcache_perthread_struct *) victim;
      memset (tcache, 0, sizeof (tcache_perthread_struct));
    }
}
```

After `tcache_init()` returns successfully, `tcache_perthread_struct` has been successfully established.

### Memory Allocation
Next we enter the memory allocation step
```C
  // Get memory from the tcache list
  if (tc_idx < mp_.tcache_bins // The idx calculated from size is within the valid range
      /*&& tc_idx < TCACHE_MAX_BINS*/ /* to appease gcc */
      && tcache
      && tcache->entries[tc_idx] != NULL) // The tcache chain is not empty
    {
      return tcache_get (tc_idx);
    }
  DIAG_POP_NEEDS_COMMENT;
#endif
  // Enter a flow similar to when there is no tcache
  if (SINGLE_THREAD_P)
    {
      victim = _int_malloc (&main_arena, bytes);
      assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
              &main_arena == arena_for_chunk (mem2chunk (victim)));
      return victim;
    }

```
When `tcache->entries` is not empty, it enters the `tcache_get()` flow to obtain a chunk, otherwise the flow is similar to before the tcache mechanism. Here we mainly analyze the first case `tcache_get()`. This also shows that tcache has very high priority, even higher than fastbin (fastbin allocation is in the flow that doesn't enter tcache).

### tcache_get()
Let's look at `tcache_get()`

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#tcache_get)
```C
/* Caller must ensure that we know tc_idx is valid and there's
   available chunks to remove.  */
static __always_inline void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  assert (tc_idx < TCACHE_MAX_BINS);
  assert (tcache->entries[tc_idx] > 0);
  tcache->entries[tc_idx] = e->next;
  --(tcache->counts[tc_idx]); // Got a chunk, decrement counts
  return (void *) e;
}
```
`tcache_get()` is the process of obtaining a chunk. As we can see, this process is quite simple - it gets the first chunk from `tcache->entries[tc_idx]`, decrements `tcache->counts`, and has almost no protection.

### __libc_free()
After looking at allocation, let's look at freeing with tcache

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#3068)
```C
void
__libc_free (void *mem)
{
  ......
  ......
  MAYBE_INIT_TCACHE ();
  ar_ptr = arena_for_chunk (p);
  _int_free (ar_ptr, p, 0);
}
```
`__libc_free()` doesn't have many changes. `MAYBE_INIT_TCACHE ()` loses its effect when tcache is not empty.

### _int_free()
Following into `_int_free()`

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#4123)
```C
static void
_int_free (mstate av, mchunkptr p, int have_lock)
{
  ......
  ......
#if USE_TCACHE
  {
    size_t tc_idx = csize2tidx (size);
    if (tcache
        && tc_idx < mp_.tcache_bins // 64
        && tcache->counts[tc_idx] < mp_.tcache_count) // 7
      {
        tcache_put (p, tc_idx);
        return;
      }
  }
#endif
  ......
  ......
```
When `tc_idx` is valid and `tcache->counts[tc_idx]` is within 7, it enters `tcache_put()`. The two parameters passed are the chunk to be freed and the index in tcache corresponding to the chunk's size.


### tcache_put()

[source code](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#2907)

```C
/* Caller must ensure that we know tc_idx is valid and there's room
   for more chunks.  */
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
`tcache_puts()` completes the operation of inserting the freed chunk at the head of the `tcache->entries[tc_idx]` linked list, also with almost no protection. Furthermore, **the p bit is not set to zero**.



## References

- http://tukan.farm/2017/07/08/tcache/
- https://github.com/bash-c/slides/blob/master/pwn_heap/tcache_exploitation.pdf
- https://www.secpulse.com/archives/71958.html
