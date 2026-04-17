# malloc_state Related Functions

## malloc_init_state

```c
/*
   Initialize a malloc_state struct.
   This is called only from within malloc_consolidate, which needs
   be called in the same contexts anyway.  It is never called directly
   outside of malloc_consolidate because some optimizing compilers try
   to inline it at all call points, which turns out not to be an
   optimization at all. (Inlining it in malloc_consolidate is fine though.)
 */

static void malloc_init_state(mstate av) {
    int     i;
    mbinptr bin;

    /* Establish circular links for normal bins */
    for (i = 1; i < NBINS; ++i) {
        bin     = bin_at(av, i);
        bin->fd = bin->bk = bin;
    }

#if MORECORE_CONTIGUOUS
    if (av != &main_arena)
#endif
        set_noncontiguous(av);
    if (av == &main_arena) set_max_fast(DEFAULT_MXFAST);
    // Set the flags to indicate there are currently no fast chunks
    av->flags |= FASTCHUNKS_BIT;
    // This is the unsorted bin
    av->top = initial_top(av);
}
```



## malloc_consolidate

This function mainly has two purposes:

1. If fastbin has not been initialized, i.e., global_max_fast is 0, then initialize malloc_state.
2. If it has already been initialized, consolidate (merge) the chunks in fastbin.

The basic flow is as follows:

### Initial

```c
static void malloc_consolidate(mstate av) {
    mfastbinptr *fb;             /* current fastbin being consolidated */
    mfastbinptr *maxfb;          /* last fastbin (for loop control) */
    mchunkptr    p;              /* current chunk being consolidated */
    mchunkptr    nextp;          /* next chunk to consolidate */
    mchunkptr    unsorted_bin;   /* bin header */
    mchunkptr    first_unsorted; /* chunk to link to */

    /* These have same use as in free() */
    mchunkptr       nextchunk;
    INTERNAL_SIZE_T size;
    INTERNAL_SIZE_T nextsize;
    INTERNAL_SIZE_T prevsize;
    int             nextinuse;
    mchunkptr       bck;
    mchunkptr       fwd;
```

### Consolidating Chunks

```c
    /*
      If max_fast is 0, we know that av hasn't
      yet been initialized, in which case do so below
    */
	// This means fastbin has already been initialized
    if (get_max_fast() != 0) {
        // Clear the fastbin flag
        // because we are about to consolidate the chunks in fastbin.
        clear_fastchunks(av);
        //
        unsorted_bin = unsorted_chunks(av);

        /*
          Remove each chunk from fast bin and consolidate it, placing it
          then in unsorted bin. Among other reasons for doing this,
          placing in unsorted bin avoids needing to calculate actual bins
          until malloc is sure that chunks aren't immediately going to be
          reused anyway.
        */
        // Traverse each bin in fastbin in fd order, and consolidate every chunk in each bin.
        maxfb = &fastbin(av, NFASTBINS - 1);
        fb    = &fastbin(av, 0);
        do {
            p = atomic_exchange_acq(fb, NULL);
            if (p != 0) {
                do {
                    check_inuse_chunk(av, p);
                    nextp = p->fd;

                    /* Slightly streamlined version of consolidation code in
                     * free() */
                    size      = chunksize(p);
                    nextchunk = chunk_at_offset(p, size);
                    nextsize  = chunksize(nextchunk);

                    if (!prev_inuse(p)) {
                        prevsize = prev_size(p);
                        size += prevsize;
                        p = chunk_at_offset(p, -((long) prevsize));
                        unlink(av, p, bck, fwd);
                    }

                    if (nextchunk != av->top) {
                        // Check whether nextchunk is free.
                        nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

                        if (!nextinuse) {
                            size += nextsize;
                            unlink(av, nextchunk, bck, fwd);
                        } else
                         // Set the prev inuse bit of nextchunk to 0, indicating that the current fast chunk can be consolidated.
                            clear_inuse_bit_at_offset(nextchunk, 0);

                        first_unsorted     = unsorted_bin->fd;
                        unsorted_bin->fd   = p;
                        first_unsorted->bk = p;

                        if (!in_smallbin_range(size)) {
                            p->fd_nextsize = NULL;
                            p->bk_nextsize = NULL;
                        }

                        set_head(p, size | PREV_INUSE);
                        p->bk = unsorted_bin;
                        p->fd = first_unsorted;
                        set_foot(p, size);
                    }

                    else {
                        size += nextsize;
                        set_head(p, size | PREV_INUSE);
                        av->top = p;
                    }

                } while ((p = nextp) != 0);
            }
        } while (fb++ != maxfb);
```

### Initialization

This means fastbin has not been initialized yet.

```c
    } else {
        malloc_init_state(av);
        // Not useful in non-debug mode; in debug mode, it performs some checks.
        check_malloc_state(av);
    }
```
