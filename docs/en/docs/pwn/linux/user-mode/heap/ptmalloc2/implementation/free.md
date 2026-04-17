# Freeing Memory Chunks

## __libc_free

Similar to malloc, the free function also has a wrapper layer, with a naming format largely similar to malloc. The code is as follows:

```c++
void __libc_free(void *mem) {
    mstate    ar_ptr;
    mchunkptr p; /* chunk corresponding to mem */
    // Check if there is a hook function __free_hook
    void (*hook)(void *, const void *) = atomic_forced_read(__free_hook);
    if (__builtin_expect(hook != NULL, 0)) {
        (*hook)(mem, RETURN_ADDRESS(0));
        return;
    }
    // free(NULL) has no effect
    if (mem == 0) /* free(0) has no effect */
        return;
    // Convert mem to chunk state
    p = mem2chunk(mem);
    // If this memory block was obtained via mmap
    if (chunk_is_mmapped(p)) /* release mmapped memory. */
    {
        /* See if the dynamic brk/mmap threshold needs adjusting.
       Dumped fake mmapped chunks do not affect the threshold.  */
        if (!mp_.no_dyn_threshold && chunksize_nomask(p) > mp_.mmap_threshold &&
            chunksize_nomask(p) <= DEFAULT_MMAP_THRESHOLD_MAX &&
            !DUMPED_MAIN_ARENA_CHUNK(p)) {
            mp_.mmap_threshold = chunksize(p);
            mp_.trim_threshold = 2 * mp_.mmap_threshold;
            LIBC_PROBE(memory_mallopt_free_dyn_thresholds, 2,
                       mp_.mmap_threshold, mp_.trim_threshold);
        }
        munmap_chunk(p);
        return;
    }
    // Get the arena pointer based on the chunk
    ar_ptr = arena_for_chunk(p);
    // Perform the free operation
    _int_free(ar_ptr, p, 0);
}
```

## _int_free

At the beginning of the function, a series of variables are defined, and the size of the chunk that the user wants to free is obtained.

```c++
static void _int_free(mstate av, mchunkptr p, int have_lock) {
    INTERNAL_SIZE_T size;      /* its size */
    mfastbinptr *   fb;        /* associated fastbin */
    mchunkptr       nextchunk; /* next contiguous chunk */
    INTERNAL_SIZE_T nextsize;  /* its size */
    int             nextinuse; /* true if nextchunk is used */
    INTERNAL_SIZE_T prevsize;  /* size of previous contiguous chunk */
    mchunkptr       bck;       /* misc temp for linking */
    mchunkptr       fwd;       /* misc temp for linking */

    const char *errstr = NULL;
    int         locked = 0;

    size = chunksize(p);
```

### Simple Checks

```c++
    /* Little security check which won't hurt performance: the
       allocator never wrapps around at the end of the address space.
       Therefore we can exclude some size values which might appear
       here by accident or by "design" from some intruder.  */
    // The pointer must not point to an illegal address; it must be less than or equal to -size. Why?
    // The pointer must be aligned; the 2*SIZE_SZ alignment requires careful consideration.
    if (__builtin_expect((uintptr_t) p > (uintptr_t) -size, 0) ||
        __builtin_expect(misaligned_chunk(p), 0)) {
        errstr = "free(): invalid pointer";
    errout:
        if (!have_lock && locked) __libc_lock_unlock(av->mutex);
        malloc_printerr(check_action, errstr, chunk2mem(p), av);
        return;
    }
    /* We know that each chunk is at least MINSIZE bytes in size or a
       multiple of MALLOC_ALIGNMENT.  */
    // The size is smaller than the minimum chunk size, or the size is not a multiple of MALLOC_ALIGNMENT
    if (__glibc_unlikely(size < MINSIZE || !aligned_OK(size))) {
        errstr = "free(): invalid size";
        goto errout;
    }
    // Check whether this chunk is in use; has no effect in non-debug mode
    check_inuse_chunk(av, p);
```

Where:

```c
/* Check if m has acceptable alignment */

#define aligned_OK(m) (((unsigned long) (m) &MALLOC_ALIGN_MASK) == 0)

#define misaligned_chunk(p)                                                    \
    ((uintptr_t)(MALLOC_ALIGNMENT == 2 * SIZE_SZ ? (p) : chunk2mem(p)) &       \
     MALLOC_ALIGN_MASK)
```



### fast bin

If all the above checks pass, it checks whether the current bin falls within the fast bin range. If so, it is inserted at the **head of the fastbin**, becoming the **first free chunk** of the corresponding fastbin linked list.

```c++
    /*
      If eligible, place chunk on a fastbin so it can be found
      and used quickly in malloc.
    */

    if ((unsigned long) (size) <= (unsigned long) (get_max_fast())

#if TRIM_FASTBINS
        /*
      If TRIM_FASTBINS set, don't place chunks
      bordering top into fastbins
        */
       // By default #define TRIM_FASTBINS 0, so the following statement is not executed by default
       // If the current chunk is a fast chunk and the next chunk is the top chunk, it cannot be inserted
        && (chunk_at_offset(p, size) != av->top)
#endif
            ) {
        // The next chunk's size must not be less than 2*SIZE_SZ, and
        // the next chunk's size must not be greater than system_mem, which is typically 132k.
        // If this situation occurs, an error is reported.
        if (__builtin_expect(
                chunksize_nomask(chunk_at_offset(p, size)) <= 2 * SIZE_SZ, 0) ||
            __builtin_expect(
                chunksize(chunk_at_offset(p, size)) >= av->system_mem, 0)) {
            /* We might not have a lock at this point and concurrent
               modifications
               of system_mem might have let to a false positive.  Redo the test
               after getting the lock.  */
            if (have_lock || ({
                    assert(locked == 0);
                    __libc_lock_lock(av->mutex);
                    locked = 1;
                    chunksize_nomask(chunk_at_offset(p, size)) <= 2 * SIZE_SZ ||
                        chunksize(chunk_at_offset(p, size)) >= av->system_mem;
                })) {
                errstr = "free(): invalid next size (fast)";
                goto errout;
            }
            if (!have_lock) {
                __libc_lock_unlock(av->mutex);
                locked = 0;
            }
        }
        // Set the entire mem portion of the chunk to perturb_byte
        free_perturb(chunk2mem(p), size - 2 * SIZE_SZ);
        // Set the fast chunk flag
        set_fastchunks(av);
        // Get the fast bin index based on size
        unsigned int idx = fastbin_index(size);
        // Get the head pointer of the corresponding fastbin, initialized to NULL.
        fb               = &fastbin(av, idx);

        /* Atomically link P to its fastbin: P->FD = *FB; *FB = P;  */
        // Use atomic operations to insert P into the linked list
        mchunkptr    old     = *fb, old2;
        unsigned int old_idx = ~0u;
        do {
            /* Check that the top of the bin is not the record we are going to
               add
               (i.e., double free).  */
            // so we can not double free one fastbin chunk
            // Prevent fast bin double free
            if (__builtin_expect(old == p, 0)) {
                errstr = "double free or corruption (fasttop)";
                goto errout;
            }
            /* Check that size of fastbin chunk at the top is the same as
               size of the chunk that we are adding.  We can dereference OLD
               only if we have the lock, otherwise it might have already been
               deallocated.  See use of OLD_IDX below for the actual check.  */
            if (have_lock && old != NULL)
                old_idx = fastbin_index(chunksize(old));
            p->fd = old2 = old;
        } while ((old = catomic_compare_and_exchange_val_rel(fb, p, old2)) !=
                 old2);
        // Ensure the fast bin is the same before and after insertion
        if (have_lock && old != NULL && __builtin_expect(old_idx != idx, 0)) {
            errstr = "invalid fastbin entry (free)";
            goto errout;
        }
    }
```

### Consolidating Non-mmap Free Chunks

**Unlink is only triggered when the chunk is not in a fast bin.**

First, let's explain why chunks are consolidated. This is to avoid having too many fragmented memory blocks in the heap. After consolidation, they can be used to serve larger memory allocation requests. The main consolidation order is:

- First consider the physically lower-addressed free chunk
- Then consider the physically higher-addressed free chunk

**After consolidation, the chunk pointer points to the lower address of the merged chunk.**

If the lock is not held, acquire the lock first.

```c++
    /*
      Consolidate other non-mmapped chunks as they arrive.
    */

    else if (!chunk_is_mmapped(p)) {
        if (!have_lock) {
            __libc_lock_lock(av->mutex);
            locked = 1;
        }
        nextchunk = chunk_at_offset(p, size);
```

#### Lightweight Checks

```c++
        /* Lightweight tests: check whether the block is already the
           top block.  */
        // The chunk currently being freed must not be the top chunk
        if (__glibc_unlikely(p == av->top)) {
            errstr = "double free or corruption (top)";
            goto errout;
        }
        // The next chunk after the chunk being freed must not exceed the arena boundary
        /* Or whether the next chunk is beyond the boundaries of the arena.  */
        if (__builtin_expect(contiguous(av) &&
                                 (char *) nextchunk >=
                                     ((char *) av->top + chunksize(av->top)),
                             0)) {
            errstr = "double free or corruption (out)";
            goto errout;
        }
        // The in-use flag of the chunk being freed is not set — double free
        /* Or whether the block is actually not marked used.  */
        if (__glibc_unlikely(!prev_inuse(nextchunk))) {
            errstr = "double free or corruption (!prev)";
            goto errout;
        }
        // Size of the next chunk
        nextsize = chunksize(nextchunk);
        // next chunk size valid check
        // Check whether the next chunk's size is not greater than 2*SIZE_SZ, or
        // whether nextsize is greater than the memory available from the system
        if (__builtin_expect(chunksize_nomask(nextchunk) <= 2 * SIZE_SZ, 0) ||
            __builtin_expect(nextsize >= av->system_mem, 0)) {
            errstr = "free(): invalid next size (normal)";
            goto errout;
        }
```

#### Free Fill

```c++
        // Set the entire mem portion of the pointer to perturb_byte
		free_perturb(chunk2mem(p), size - 2 * SIZE_SZ);
```

#### Backward Consolidation — Merge with Lower-Address Chunk

```c++
        /* consolidate backward */
        if (!prev_inuse(p)) {
            prevsize = prev_size(p);
            size += prevsize;
            p = chunk_at_offset(p, -((long) prevsize));
            unlink(av, p, bck, fwd);
        }
```

#### Next Chunk Is Not the Top Chunk — Forward Consolidation — Merge with Higher-Address Chunk

Note that if the next chunk is not the top chunk, the higher-addressed chunk is consolidated, and the merged chunk is placed into the unsorted bin.

```c++
		// If the next chunk is not the top chunk
		if (nextchunk != av->top) {
            /* get and clear inuse bit */
            // Get the in-use status of the next chunk
            nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
            // If not in use, consolidate; otherwise, clear the in-use flag of the current chunk.
            /* consolidate forward */
            if (!nextinuse) {
                unlink(av, nextchunk, bck, fwd);
                size += nextsize;
            } else
                clear_inuse_bit_at_offset(nextchunk, 0);

            /*
          Place the chunk in unsorted chunk list. Chunks are
          not placed into regular bins until after they have
          been given one chance to be used in malloc.
            */
            // Place the chunk at the head of the unsorted chunk list
            bck = unsorted_chunks(av);
            fwd = bck->fd;
            // Simple check
            if (__glibc_unlikely(fwd->bk != bck)) {
                errstr = "free(): corrupted unsorted chunks";
                goto errout;
            }
            p->fd = fwd;
            p->bk = bck;
            // If it is a large chunk, set the nextsize pointer fields to NULL.
            if (!in_smallbin_range(size)) {
                p->fd_nextsize = NULL;
                p->bk_nextsize = NULL;
            }
            bck->fd = p;
            fwd->bk = p;

            set_head(p, size | PREV_INUSE);
            set_foot(p, size);

            check_free_chunk(av, p);
        }
```

#### Next Chunk Is the Top Chunk — Merge into the Top Chunk

```c++
        /*
          If the chunk borders the current high end of memory,
          consolidate into top
        */
        // If the next chunk of the chunk being freed is the top chunk, merge into the top chunk
        else {
            size += nextsize;
            set_head(p, size | PREV_INUSE);
            av->top = p;
            check_chunk(av, p);
        }
```

#### Returning Memory to the System

```c++
        /*
          If freeing a large space, consolidate possibly-surrounding
          chunks. Then, if the total unused topmost memory exceeds trim
          threshold, ask malloc_trim to reduce top.

          Unless max_fast is 0, we don't know if there are fastbins
          bordering top, so we cannot tell for sure whether threshold
          has been reached unless fastbins are consolidated.  But we
          don't want to consolidate on each free.  As a compromise,
          consolidation is performed if FASTBIN_CONSOLIDATION_THRESHOLD
          is reached.
        */
         // If the size of the consolidated chunk is greater than FASTBIN_CONSOLIDATION_THRESHOLD,
         // this code is generally executed when merging into the top chunk.
         // Then return memory to the system.
        if ((unsigned long) (size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
            // If there are fast chunks, consolidate them
            if (have_fastchunks(av)) malloc_consolidate(av);
            // Main arena
            if (av == &main_arena) {
#ifndef MORECORE_CANNOT_TRIM
                // The top chunk is larger than the current trim threshold
                if ((unsigned long) (chunksize(av->top)) >=
                    (unsigned long) (mp_.trim_threshold))
                    systrim(mp_.top_pad, av);
#endif      // Non-main arena, directly trim the heap
            } else {
                /* Always try heap_trim(), even if the top chunk is not
                   large, because the corresponding heap might go away.  */
                heap_info *heap = heap_for_ptr(top(av));

                assert(heap->ar_ptr == av);
                heap_trim(heap, mp_.top_pad);
            }
        }

        if (!have_lock) {
            assert(locked);
            __libc_lock_unlock(av->mutex);
        }
```

### Freeing mmap'd Chunks

```c++
    } else {
        //  If the chunk was allocated via mmap, release via munmap().
        munmap_chunk(p);
    }
```

## systrim

## heap_trim

## munmap_chunk
