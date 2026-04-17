# Heap-Related Data Structures

Heap operations are quite complex, so glibc must have carefully designed data structures internally to manage them. The data structures related to the heap can be mainly divided into:

- Macroscopic structures, which contain high-level information about the heap and can be used to index basic heap information.
- Microscopic structures, which are used to specifically handle memory blocks during heap allocation and deallocation.

## Overview

> To be supplemented.

## Microscopic Structures

Here we first introduce the main internal structures of the heap. **Heap vulnerability exploitation is closely related to these structures.**

### malloc_chunk

#### Overview

During program execution, we refer to the memory allocated by malloc as a `chunk`. This memory is internally represented by the malloc_chunk structure in ptmalloc. When an allocated `chunk` is freed, it is added to the corresponding free management list.

Interestingly, **regardless of a `chunk`'s size or whether it is in an allocated or freed state, they all use a unified structure**. Although they use the same data structure, their representation differs depending on whether they have been freed.

The structure of malloc_chunk is as follows:

```c
/*
  This struct declaration is misleading (but accurate and necessary).
  It declares a "view" into memory allowing access to necessary
  fields at known offsets from a given base. See explanation below.
*/
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```

First, let's provide the definitions of some necessary macros:

```c
/* INTERNAL_SIZE_T is the word-size used for internal bookkeeping of
   chunk sizes.
   The default version is the same as size_t.
   While not strictly necessary, it is best to define this as an
   unsigned type, even if size_t is a signed type. This may avoid some
   artificial size limitations on some systems.
   On a 64-bit machine, you may be able to reduce malloc overhead by
   defining INTERNAL_SIZE_T to be a 32 bit `unsigned int' at the
   expense of not being able to handle more than 2^32 of malloced
   space. If this limitation is acceptable, you are encouraged to set
   this unless you are on a platform requiring 16byte alignments. In
   this case the alignment requirements turn out to negate any
   potential advantages of decreasing size_t word size.
   Implementors: Beware of the possible combinations of:
     - INTERNAL_SIZE_T might be signed or unsigned, might be 32 or 64 bits,
       and might be the same width as int or as long
     - size_t might have different width and signedness as INTERNAL_SIZE_T
     - int and long might be 32 or 64 bits, and might be the same width
   To deal with this, most comparisons and difference computations
   among INTERNAL_SIZE_Ts should cast them to unsigned long, being
   aware of the fact that casting an unsigned int to a wider long does
   not sign-extend. (This also makes checking for negative numbers
   awkward.) Some of these casts result in harmless compiler warnings
   on some systems.  */
#ifndef INTERNAL_SIZE_T
# define INTERNAL_SIZE_T size_t
#endif

/* The corresponding word size.  */
#define SIZE_SZ (sizeof (INTERNAL_SIZE_T))

/* The corresponding bit mask value.  */
#define MALLOC_ALIGN_MASK (MALLOC_ALIGNMENT - 1)

/* MALLOC_ALIGNMENT is the minimum alignment for malloc'ed chunks.  It
   must be a power of two at least 2 * SIZE_SZ, even on machines for
   which smaller alignments would suffice. It may be defined as larger
   than this though. Note however that code and data structures are
   optimized for the case of 8-byte alignment.  */
#define MALLOC_ALIGNMENT (2 * SIZE_SZ < __alignof__ (long double) \
			  ? __alignof__ (long double) : 2 * SIZE_SZ)
```

> Generally speaking, `size_t` is defined as `unsigned long`, which is a 64-bit unsigned integer on 64-bit systems and a 32-bit unsigned integer on 32-bit systems.

Now let's look at the `chunk` structure. The detailed explanation of each field is as follows:

-   **prev_size**: If the **physically adjacent previous chunk (the address difference between the two pointers equals the size of the previous chunk)** of this `chunk` is free, then this field records the size of the previous `chunk` (including the `chunk` header). Otherwise, this field can be used to store data for the physically adjacent previous chunk. **The "previous chunk" here refers to the chunk at a lower address.**
-   **size**: The size of this `chunk`. The size must be a multiple of `MALLOC_ALIGNMENT`. If the requested memory size is not a multiple of `MALLOC_ALIGNMENT`, it will be converted to the smallest multiple of `MALLOC_ALIGNMENT` that satisfies the size requirement, done through the `request2size()` macro. On 32-bit systems, `MALLOC_ALIGNMENT` may be `4` or `8`; on 64-bit systems, `MALLOC_ALIGNMENT` is `8`. The lowest three bits of this field do not affect the `chunk` size. From high to low, they represent:
    -   NON_MAIN_ARENA: Records whether the current `chunk` does not belong to the main thread. 1 means it does not belong, 0 means it belongs.
    -   IS_MAPPED: Records whether the current `chunk` was allocated by mmap. 
    -   PREV_INUSE: Records whether the previous `chunk` is allocated. Generally, the P bit in the size field of the first allocated memory block in the heap is always set to 1 to prevent accessing illegal memory before it. When a `chunk`'s size P bit is 0, we can obtain the size and address of the previous `chunk` through the `prev_size` field. This also facilitates merging of free `chunk`s.
-   **fd, bk**: When a `chunk` is in the allocated state, the user data starts from the fd field. When a `chunk` is free, it is added to the corresponding free management linked list, and the fields have the following meanings:
    -   fd points to the next (non-physically adjacent) free `chunk`.
    -   bk points to the previous (non-physically adjacent) free `chunk`.
    -   Through fd and bk, free `chunk`s can be added to a free `chunk` linked list for unified management.
-   **fd_nextsize, bk_nextsize**: These are also only used when the `chunk` is free, and they are used for larger chunks (large chunks).
    -   fd_nextsize points to the first free block preceding the current `chunk` that has a different size, excluding the bin's head pointer.
    -   bk_nextsize points to the first free block following the current `chunk` that has a different size, excluding the bin's head pointer.
    -   Generally, free large `chunk`s are sorted in descending order of size when traversed via fd pointers. **This is done to avoid having to traverse each chunk one by one when searching for a suitable chunk.**

> Starting from glibc version 2.26, in 32-bit glibc, the `MALLOC_ALIGNMENT` macro definition is preferentially selected at compile time from the definition in `sysdeps/i386/malloc-alignment.h`, which defines the value as a constant:
> 
> ```c
> #define MALLOC_ALIGNMENT 16
> ```
> 
> Therefore, for 32-bit glibc starting from version 2.26, `MALLOC_ALIGNMENT` is not calculated based on `SIZE_SZ` as `8`, but is the same value `16` as used by 64-bit glibc.

An allocated `chunk` looks like the following. **We call the first two fields the `chunk` header, and the remaining part the user data. The memory pointer returned by each malloc call actually points to the beginning of the user data.**

When a `chunk` is in use, the prev_size field of its next `chunk` is invalid, so that portion of the next `chunk` can also be used by the current chunk. **This is the space reuse within chunks.**

```c++
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if unallocated (P clear)  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     |A|M|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             User data starts here...                          .
        .                                                               .
        .             (malloc_usable_size() bytes)                      .
next    .                                                               |
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             (size of chunk, but used for application data)    |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of next chunk, in bytes                |A|0|1|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

A freed `chunk` is recorded in a linked list (which may be a circular doubly-linked list or a singly-linked list). The specific structure is as follows:

```c++
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if unallocated (P clear)  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`head:' |             Size of chunk, in bytes                     |A|0|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Forward pointer to next `chunk` in list             |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Back pointer to previous `chunk` in list            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Unused space (may be 0 bytes long)                .
        .                                                               .
 next   .                                                               |
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`foot:' |             Size of chunk, in bytes                           |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of next chunk, in bytes                |A|0|0|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

We can observe that if a `chunk` is in the free state, its corresponding size is recorded in two places:

1. Its own size field records it.

2. The `chunk` after it records it.

**In general**, two physically adjacent free `chunk`s will be merged into one `chunk`. The heap manager uses the prev_size field and the size field to merge two physically adjacent free `chunk` blocks.

**!!! Some constraints about the heap — to be considered in detail later !!!**

```c++
/*
    The three exceptions to all this are:
     1. The special chunk `top' doesn't bother using the
    trailing size field since there is no next contiguous chunk
    that would have to index off it. After initialization, `top'
    is forced to always exist.  If it would become less than
    MINSIZE bytes long, it is replenished.
     2. Chunks allocated via mmap, which have the second-lowest-order
    bit M (IS_MMAPPED) set in their size fields.  Because they are
    allocated one-by-one, each must contain its own trailing size
    field.  If the M bit is set, the other bits are ignored
    (because mmapped chunks are neither in an arena, nor adjacent
    to a freed chunk).  The M bit is also used for chunks which
    originally came from a dumped heap via malloc_set_state in
    hooks.c.
     3. Chunks in fastbins are treated as allocated chunks from the
    point of view of the chunk allocator.  They are consolidated
    with their neighbors only in bulk, in malloc_consolidate.
*/
```

#### Chunk-Related Macros

Here we mainly introduce macros related to `chunk` size, alignment checking, and some conversions.

**Conversion Between chunk and mem Pointers**

The mem pointer points to the start of the memory that the user receives.

```c++
/* conversion from malloc headers to user pointers, and back */
#define chunk2mem(p) ((void *) ((char *) (p) + 2 * SIZE_SZ))
#define mem2chunk(mem) ((mchunkptr)((char *) (mem) -2 * SIZE_SZ))
```

**Minimum `chunk` Size**

```c++
/* The smallest possible chunk */
#define MIN_CHUNK_SIZE (offsetof(struct malloc_chunk, fd_nextsize))
```

Here, the offsetof function calculates the offset of fd_nextsize within malloc_chunk, indicating that the smallest `chunk` must at least contain the bk pointer.

**Minimum Requested Heap Memory Size**

The minimum memory size requested by the user must be the smallest integer multiple of 2 * SIZE_SZ.

**Note: Currently, MIN_CHUNK_SIZE and MINSIZE are the same size. The reason for having two macros is presumably to make future modifications to malloc_chunk more convenient.**

```c++
/* The smallest size we can malloc is an aligned minimal chunk */
//MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1
#define MINSIZE                                                                \
    (unsigned long) (((MIN_CHUNK_SIZE + MALLOC_ALIGN_MASK) &                   \
                      ~MALLOC_ALIGN_MASK))
```

**Check Whether Memory Allocated to the User is Aligned**

Aligned to 2 * SIZE_SZ.

```c++
/* Check if m has acceptable alignment */
// MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1
#define aligned_OK(m) (((unsigned long) (m) & MALLOC_ALIGN_MASK) == 0)

#define misaligned_chunk(p)                                                    \
    ((uintptr_t)(MALLOC_ALIGNMENT == 2 * SIZE_SZ ? (p) : chunk2mem(p)) &       \
     MALLOC_ALIGN_MASK)
```

**Request Size Validation**

```c++
/*
   Check if a request is so large that it would wrap around zero when
   padded and aligned. To simplify some other code, the bound is made
   low enough so that adding MINSIZE will also not wrap around zero.
 */

#define REQUEST_OUT_OF_RANGE(req)                                              \
    ((unsigned long) (req) >= (unsigned long) (INTERNAL_SIZE_T)(-2 * MINSIZE))
```

**Convert User-Requested Memory Size to Actual Allocation Size**

```c++
/* pad request bytes into a usable size -- internal version */
//MALLOC_ALIGN_MASK = 2 * SIZE_SZ -1
#define request2size(req)                                                      \
    (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)                           \
         ? MINSIZE                                                             \
         : ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

/*  Same, except also perform argument check */

#define checked_request2size(req, sz)                                          \
    if (REQUEST_OUT_OF_RANGE(req)) {                                           \
        __set_errno(ENOMEM);                                                   \
        return 0;                                                              \
    }                                                                          \
    (sz) = request2size(req);
```

When a `chunk` is in the allocated state, the prev_size field of its physically adjacent next `chunk` is necessarily invalid, so this field can be used by the current `chunk`. This is the reuse between `chunk`s in ptmalloc. The specific process is as follows:

1. First, REQUEST_OUT_OF_RANGE is used to determine whether a chunk of the user-requested byte size can be allocated.
2. Second, it is important to note that the bytes requested by the user are for storing data, i.e., the part after the `chunk` header. At the same time, due to chunk reuse, the prev_size field of the next `chunk` can be used. Therefore, only an additional SIZE_SZ is needed to fully store the content.
3. Since the minimum `chunk` allowed by the system is MINSIZE, a comparison is made with it. If the minimum requirement is not met, MINSIZE bytes are directly allocated.
4. If it is larger, since `chunk`s allocated by the system need to be aligned to 2 * SIZE_SZ, MALLOC_ALIGN_MASK is added here for alignment purposes.

**Flag Bit Related**

```c++
/* size field is or'ed with PREV_INUSE when previous adjacent chunk in use */
#define PREV_INUSE 0x1

/* extract inuse bit of previous chunk */
#define prev_inuse(p) ((p)->mchunk_size & PREV_INUSE)

/* size field is or'ed with IS_MMAPPED if the chunk was obtained with mmap() */
#define IS_MMAPPED 0x2

/* check for mmap()'ed chunk */
#define chunk_is_mmapped(p) ((p)->mchunk_size & IS_MMAPPED)

/* size field is or'ed with NON_MAIN_ARENA if the chunk was obtained
   from a non-main arena.  This is only set immediately before handing
   the chunk to the user, if necessary.  */
#define NON_MAIN_ARENA 0x4

/* Check for chunk from main arena.  */
#define chunk_main_arena(p) (((p)->mchunk_size & NON_MAIN_ARENA) == 0)

/* Mark a chunk as not being on the main arena.  */
#define set_non_main_arena(p) ((p)->mchunk_size |= NON_MAIN_ARENA)

/*
   Bits to mask off when extracting size
   Note: IS_MMAPPED is intentionally not masked off from size field in
   macros for which mmapped chunks should never be seen. This should
   cause helpful core dumps to occur if it is tried by accident by
   people extending or adapting this malloc.
 */
#define SIZE_BITS (PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
```

**Get chunk size**

```c++
/* Get size, ignoring use bits */
#define chunksize(p) (chunksize_nomask(p) & ~(SIZE_BITS))

/* Like chunksize, but do not mask SIZE_BITS.  */
#define chunksize_nomask(p) ((p)->mchunk_size)
```

**Get the Next Physically Adjacent chunk**

```c++
/* Ptr to next physical malloc_chunk. */
#define next_chunk(p) ((mchunkptr)(((char *) (p)) + chunksize(p)))
```

**Get Information About the Previous chunk**

```c++
/* Size of the chunk below P.  Only valid if !prev_inuse (P).  */
#define prev_size(p) ((p)->mchunk_prev_size)

/* Set the size of the chunk below P.  Only valid if !prev_inuse (P).  */
#define set_prev_size(p, sz) ((p)->mchunk_prev_size = (sz))

/* Ptr to previous physical malloc_chunk.  Only valid if !prev_inuse (P).  */
#define prev_chunk(p) ((mchunkptr)(((char *) (p)) - prev_size(p)))
```

**Operations Related to the Current chunk's Usage State**

```c++
/* extract p's inuse bit */
#define inuse(p)                                                               \
    ((((mchunkptr)(((char *) (p)) + chunksize(p)))->mchunk_size) & PREV_INUSE)

/* set/clear chunk as being inuse without otherwise disturbing */
#define set_inuse(p)                                                           \
    ((mchunkptr)(((char *) (p)) + chunksize(p)))->mchunk_size |= PREV_INUSE

#define clear_inuse(p)                                                         \
    ((mchunkptr)(((char *) (p)) + chunksize(p)))->mchunk_size &= ~(PREV_INUSE)
```

**Set the chunk's size Field**

```c++
/* Set size at head, without disturbing its use bit */
// SIZE_BITS = 7
#define set_head_size(p, s)                                                    \
    ((p)->mchunk_size = (((p)->mchunk_size & SIZE_BITS) | (s)))

/* Set size/use field */
#define set_head(p, s) ((p)->mchunk_size = (s))

/* Set size at footer (only when chunk is not in use) */
#define set_foot(p, s)                                                         \
    (((mchunkptr)((char *) (p) + (s)))->mchunk_prev_size = (s))
```

**Get the chunk at a Specified Offset**

```c++
/* Treat space at ptr + offset as a chunk */
#define chunk_at_offset(p, s) ((mchunkptr)(((char *) (p)) + (s)))
```

**Operations Related to the Usage State of a chunk at a Specified Offset**

```c++
/* check/set/clear inuse bits in known places */
#define inuse_bit_at_offset(p, s)                                              \
    (((mchunkptr)(((char *) (p)) + (s)))->mchunk_size & PREV_INUSE)

#define set_inuse_bit_at_offset(p, s)                                          \
    (((mchunkptr)(((char *) (p)) + (s)))->mchunk_size |= PREV_INUSE)

#define clear_inuse_bit_at_offset(p, s)                                        \
    (((mchunkptr)(((char *) (p)) + (s)))->mchunk_size &= ~(PREV_INUSE))
```

### bin

#### Overview

As we mentioned before, `chunk`s freed by the user are not immediately returned to the system. ptmalloc uniformly manages free chunks in the heap and mmap mapped regions. When the user requests memory allocation again, the ptmalloc allocator will try to pick a suitable chunk from the free chunks for the user. This avoids frequent system calls and reduces the overhead of memory allocation.

In the specific implementation, ptmalloc uses a binning method to manage free `chunk`s. First, it initially classifies free `chunk`s into 4 categories based on their size and usage state: fast bins, small bins, large bins, and unsorted bin. Within each category, there are further subdivisions where `chunk`s of similar sizes are linked together using doubly-linked lists. In other words, within each bin category, there are still multiple independent linked lists to store chunks of different sizes.

For small bins, large bins, and unsorted bin, ptmalloc maintains them in the same array. The data structures corresponding to these bins are in malloc_state, as follows:

```c++
#define NBINS 128
/* Normal bins packed as described above */
mchunkptr bins[ NBINS * 2 - 2 ];
```

`bins` is mainly used to index the fd and bk of different bins.

To simplify usage in doubly-linked lists, each bin's header is set to the malloc_chunk type. This avoids special handling for header types. However, to save space and improve locality, only the fd/bk pointers of the bin are allocated, and then repositioning tricks are used to treat these pointers as fields of a `malloc_chunk*`.

Taking a 32-bit system as an example, the meanings of the first 4 items of bins are as follows:

| Meaning    | bin1's fd / bin2's prev_size | bin1's bk / bin2's size | bin2's fd / bin3's prev_size | bin2's bk / bin3's size |
| ----- | ---------------------- | ----------------- | ---------------------- | ----------------- |
| bin index | 0                      | 1                 | 2                      | 3                 |

We can see that bin2's prev_size and size overlap with bin1's fd and bk. Since we only use fd and bk to index the linked list, the data in the overlapping portion actually records bin1's fd and bk. In other words, although the latter bin shares part of its data with the former bin, what is actually recorded is still the linked list data of the former bin. This reuse saves space.

The bins in the array are as follows:

1. The first one is the unsorted bin — as the name implies, the `chunk`s in it are unsorted, and the stored `chunk`s are quite varied.
2. Bins with indices from 2 to 63 are called small bins. `Chunk`s in the same small bin linked list have the same size. The size difference between `chunk`s in two adjacent small bin linked lists is **2 machine words**, i.e., 8 bytes on 32-bit systems and 16 bytes on 64-bit systems.
3. The bins after small bins are called large bins. Each bin in large bins contains chunks within a certain size range. The chunks within are sorted in descending order by fd pointer. Chunks of the same size are also sorted by most-recently-used order.

Additionally, the arrangement of all these bins follows one principle: **no two physically adjacent free chunks can exist next to each other**.

It should be noted that not all freed `chunk`s are immediately placed into bins. To improve allocation speed, ptmalloc **first** places some small `chunk`s into fast bin containers. **Moreover, the usage flag of `chunk`s in fastbin containers is always set, so they do not satisfy the above principle.**

The common macros for bins are as follows:

```c++
typedef struct malloc_chunk *mbinptr;

/* addressing -- note that bin_at(0) does not exist */
#define bin_at(m, i)                                                           \
    (mbinptr)(((char *) &((m)->bins[ ((i) -1) * 2 ])) -                        \
              offsetof(struct malloc_chunk, fd))

/* analog of ++bin */
// Get the address of the next bin
#define next_bin(b) ((mbinptr)((char *) (b) + (sizeof(mchunkptr) << 1)))

/* Reminders about list directionality within bins */
// These two macros can be used to traverse bins
// Get the chunk at the head of the bin's linked list
#define first(b) ((b)->fd)
// Get the chunk at the tail of the bin's linked list
#define last(b) ((b)->bk)
```

#### Fast Bin

Most programs frequently allocate and free relatively small memory blocks. If small `chunk`s are freed and then merged with adjacent free `chunk`s, the next time a chunk of the corresponding size is requested, the chunk would need to be split again. This would greatly reduce the efficiency of heap utilization, **because we would spend most of our time on merging, splitting, and intermediate checks.** Therefore, ptmalloc specifically designed fast bins, whose corresponding variable is fastbinsY in malloc_state.

```c++
/*
   Fastbins

    An array of lists holding recently freed small chunks.  Fastbins
    are not doubly linked.  It is faster to single-link them, and
    since chunks are never removed from the middles of these lists,
    double linking is not necessary. Also, unlike regular bins, they
    are not even processed in FIFO order (they use faster LIFO) since
    ordering doesn't much matter in the transient contexts in which
    fastbins are normally used.

    Chunks in fastbins keep their inuse bit set, so they cannot
    be consolidated with other free chunks. malloc_consolidate
    releases all chunks in fastbins and consolidates them with
    other free chunks.
 */
typedef struct malloc_chunk *mfastbinptr;

/*
    This is in malloc_state.
    /* Fastbins */
    mfastbinptr fastbinsY[ NFASTBINS ];
*/
```

To utilize fast bins more efficiently, glibc uses singly-linked lists to organize each bin within them, and **each bin adopts a LIFO (Last In, First Out) strategy** — the most recently freed `chunk` will be allocated sooner, making it more suitable for locality. In other words, when the size of the `chunk` needed by the user is less than the maximum size of fastbin, ptmalloc first checks whether there is a free block of the corresponding size in the appropriate fastbin bin. If so, the chunk is directly obtained from this bin. If not, ptmalloc then proceeds with subsequent operations.

By default (**using a 32-bit system as an example**), fastbin supports a maximum `chunk` data space size of 64 bytes. However, the maximum chunk data space it can support is 80 bytes. Additionally, fastbin can support a maximum of 10 bins, ranging from data space sizes of 8 bytes up to 80 bytes (note that this refers to the data space size, i.e., the size excluding the prev_size and size fields). The definitions are as follows:

```c++
#define NFASTBINS (fastbin_index(request2size(MAX_FAST_SIZE)) + 1)

#ifndef DEFAULT_MXFAST
#define DEFAULT_MXFAST (64 * SIZE_SZ / 4)
#endif
  
/* The maximum fastbin request size we support */
#define MAX_FAST_SIZE (80 * SIZE_SZ / 4)

/*
   Since the lowest 2 bits in max_fast don't matter in size comparisons,
   they are used as flags.
 */

/*
   FASTCHUNKS_BIT held in max_fast indicates that there are probably
   some fastbin chunks. It is set true on entering a chunk into any
   fastbin, and cleared only in malloc_consolidate.

   The truth value is inverted so that have_fastchunks will be true
   upon startup (since statics are zero-filled), simplifying
   initialization checks.
 */
// Determines whether the arena has fast bin chunks; 1 means it does not
#define FASTCHUNKS_BIT (1U)

#define have_fastchunks(M) (((M)->flags & FASTCHUNKS_BIT) == 0)
#define clear_fastchunks(M) catomic_or(&(M)->flags, FASTCHUNKS_BIT)
#define set_fastchunks(M) catomic_and(&(M)->flags, ~FASTCHUNKS_BIT)

/*
   NONCONTIGUOUS_BIT indicates that MORECORE does not return contiguous
   regions.  Otherwise, contiguity is exploited in merging together,
   when possible, results from consecutive MORECORE calls.

   The initial value comes from MORECORE_CONTIGUOUS, but is
   changed dynamically if mmap is ever used as an sbrk substitute.
 */
// Whether MORECORE returns contiguous memory regions.
// MORECORE in the main arena is actually sbrk(), which returns contiguous virtual address space by default.
// Non-main arenas use mmap() to allocate large blocks of virtual memory, then split them to simulate main arena behavior.
// By default, mmap mapped regions are not guaranteed to have contiguous virtual address space, so non-main arenas allocate non-contiguous virtual address space by default.
#define NONCONTIGUOUS_BIT (2U)

#define contiguous(M) (((M)->flags & NONCONTIGUOUS_BIT) == 0)
#define noncontiguous(M) (((M)->flags & NONCONTIGUOUS_BIT) != 0)
#define set_noncontiguous(M) ((M)->flags |= NONCONTIGUOUS_BIT)
#define set_contiguous(M) ((M)->flags &= ~NONCONTIGUOUS_BIT)

/* ARENA_CORRUPTION_BIT is set if a memory corruption was detected on the
   arena.  Such an arena is no longer used to allocate chunks.  Chunks
   allocated in that arena before detecting corruption are not freed.  */

#define ARENA_CORRUPTION_BIT (4U)

#define arena_is_corrupt(A) (((A)->flags & ARENA_CORRUPTION_BIT))
#define set_arena_corrupt(A) ((A)->flags |= ARENA_CORRUPTION_BIT)

/*
   Set value of max_fast.
   Use impossibly small value if 0.
   Precondition: there are no existing fastbin chunks.
   Setting the value clears fastchunk bit but preserves noncontiguous bit.
 */

#define set_max_fast(s)                                                        \
    global_max_fast =                                                          \
        (((s) == 0) ? SMALLBIN_WIDTH : ((s + SIZE_SZ) & ~MALLOC_ALIGN_MASK))
#define get_max_fast() global_max_fast
```

By default, ptmalloc calls `set_max_fast(s)` to set the global variable `global_max_fast` to `DEFAULT_MXFAST`, which sets the maximum `chunk` size in fast bins. When `MAX_FAST_SIZE` is set to 0, the system will not support fastbin.

**Fastbin Indexing**

```c++

#define fastbin(ar_ptr, idx) ((ar_ptr)->fastbinsY[ idx ])

/* offset 2 to use otherwise unindexable first 2 bins */
// chunk size=2*size_sz*(2+idx)
// We subtract 2 here; otherwise the first two bins cannot be indexed.
#define fastbin_index(sz)                                                      \
    ((((unsigned int) (sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)
```

**It is particularly important to note that the inuse bit of `chunk`s in the fastbin range is always set to 1. Therefore, they will not be merged with other freed `chunk`s.**

However, when a freed `chunk` merges with its adjacent free `chunk` and the resulting size exceeds FASTBIN_CONSOLIDATION_THRESHOLD, memory fragmentation may be significant. In this case, we need to merge all chunks in fast bins to reduce the impact of memory fragmentation on the system.

```c++
/*
   FASTBIN_CONSOLIDATION_THRESHOLD is the size of a chunk in free()
   that triggers automatic consolidation of possibly-surrounding
   fastbin chunks. This is a heuristic, so the exact value should not
   matter too much. It is defined at half the default trim threshold as a
   compromise heuristic to only attempt consolidation if it is likely
   to lead to trimming. However, it is not dynamically tunable, since
   consolidation reduces fragmentation surrounding large chunks even
   if trimming is not used.
 */

#define FASTBIN_CONSOLIDATION_THRESHOLD (65536UL)
```

**The malloc_consolidate function can merge all `chunk`s in fastbin that can be merged with other `chunk`s. See the detailed function analysis later for specifics.**

```
/*
	Chunks in fastbins keep their inuse bit set, so they cannot
    be consolidated with other free chunks. malloc_consolidate
    releases all chunks in fastbins and consolidates them with
    other free chunks.
 */
```

#### Small Bin

The relationship between the size of each `chunk` in small bins and the index of its bin is: chunk_size = 2 * SIZE_SZ * index, specifically:

| Index   | SIZE_SZ=4 (32-bit) | SIZE_SZ=8 (64-bit) |
| ---- | -------------- | -------------- |
| 2    | 16             | 32             |
| 3    | 24             | 48             |
| 4    | 32             | 64             |
| 5    | 40             | 80             |
| x    | 2\*4\*x        | 2\*8\*x        |
| 63   | 504            | 1008           |

There are a total of 62 circular doubly-linked lists in small bins, and each linked list stores `chunk`s of the same size. For example, on a 32-bit system, the doubly-linked list at index 2 stores `chunk`s that are all 16 bytes in size. Each linked list has a head node, which facilitates management of internal nodes. Additionally, **each bin's linked list in small bins follows the FIFO (First In, First Out) rule**, so `chunk`s that were freed first in the same linked list will be allocated first.

The macros related to small bins are as follows:

```c++
#define NSMALLBINS 64
#define SMALLBIN_WIDTH MALLOC_ALIGNMENT
// Whether correction of small bin indices is needed
#define SMALLBIN_CORRECTION (MALLOC_ALIGNMENT > 2 * SIZE_SZ)

#define MIN_LARGE_SIZE ((NSMALLBINS - SMALLBIN_CORRECTION) * SMALLBIN_WIDTH)
// Determine whether the chunk size is within the small bin range
#define in_smallbin_range(sz)                                                  \
    ((unsigned long) (sz) < (unsigned long) MIN_LARGE_SIZE)
// Get the small bin index based on chunk size
#define smallbin_index(sz)                                                     \
    ((SMALLBIN_WIDTH == 16 ? (((unsigned) (sz)) >> 4)                          \
                           : (((unsigned) (sz)) >> 3)) +                       \
     SMALLBIN_CORRECTION)
```

**You might be confused — the sizes of chunks in fastbin and small bin overlap significantly, so wouldn't the small bin at the corresponding size be useless?** That's actually not the case. Chunks in fast bins can indeed be placed into small bins. We will have a deeper understanding of this when analyzing the specific source code later.

#### Large Bin

Large bins contain a total of 63 bins. The `chunk` sizes within each bin are not uniform but fall within a certain range. Furthermore, these 63 bins are divided into 6 groups, where the size difference between chunks within each group is consistent, as follows:

| Group    | Count   | Common Difference      |
| ---- | ---- | ------- |
| 1    | 32   | 64B     |
| 2    | 16   | 512B    |
| 3    | 8    | 4096B   |
| 4    | 4    | 32768B  |
| 5    | 2    | 262144B |
| 6    | 1    | No limit     |

Here, taking the 32-bit platform's large bin as an example, the starting `chunk` size of the first large bin is 512 bytes. It is in the first group, so this bin can store `chunk`s with sizes in the range [512, 512+64).

The macros related to large bins are as follows. Taking the 32-bit platform as an example, the starting `chunk` size of the first large bin is 512 bytes, so 512>>6 = 8, and thus its index is 56+8=64.

```c++
#define largebin_index_32(sz)                                                  \
    (((((unsigned long) (sz)) >> 6) <= 38)                                     \
         ? 56 + (((unsigned long) (sz)) >> 6)                                  \
         : ((((unsigned long) (sz)) >> 9) <= 20)                               \
               ? 91 + (((unsigned long) (sz)) >> 9)                            \
               : ((((unsigned long) (sz)) >> 12) <= 10)                        \
                     ? 110 + (((unsigned long) (sz)) >> 12)                    \
                     : ((((unsigned long) (sz)) >> 15) <= 4)                   \
                           ? 119 + (((unsigned long) (sz)) >> 15)              \
                           : ((((unsigned long) (sz)) >> 18) <= 2)             \
                                 ? 124 + (((unsigned long) (sz)) >> 18)        \
                                 : 126)

#define largebin_index_32_big(sz)                                              \
    (((((unsigned long) (sz)) >> 6) <= 45)                                     \
         ? 49 + (((unsigned long) (sz)) >> 6)                                  \
         : ((((unsigned long) (sz)) >> 9) <= 20)                               \
               ? 91 + (((unsigned long) (sz)) >> 9)                            \
               : ((((unsigned long) (sz)) >> 12) <= 10)                        \
                     ? 110 + (((unsigned long) (sz)) >> 12)                    \
                     : ((((unsigned long) (sz)) >> 15) <= 4)                   \
                           ? 119 + (((unsigned long) (sz)) >> 15)              \
                           : ((((unsigned long) (sz)) >> 18) <= 2)             \
                                 ? 124 + (((unsigned long) (sz)) >> 18)        \
                                 : 126)

// XXX It remains to be seen whether it is good to keep the widths of
// XXX the buckets the same or whether it should be scaled by a factor
// XXX of two as well.
#define largebin_index_64(sz)                                                  \
    (((((unsigned long) (sz)) >> 6) <= 48)                                     \
         ? 48 + (((unsigned long) (sz)) >> 6)                                  \
         : ((((unsigned long) (sz)) >> 9) <= 20)                               \
               ? 91 + (((unsigned long) (sz)) >> 9)                            \
               : ((((unsigned long) (sz)) >> 12) <= 10)                        \
                     ? 110 + (((unsigned long) (sz)) >> 12)                    \
                     : ((((unsigned long) (sz)) >> 15) <= 4)                   \
                           ? 119 + (((unsigned long) (sz)) >> 15)              \
                           : ((((unsigned long) (sz)) >> 18) <= 2)             \
                                 ? 124 + (((unsigned long) (sz)) >> 18)        \
                                 : 126)

#define largebin_index(sz)                                                     \
    (SIZE_SZ == 8 ? largebin_index_64(sz) : MALLOC_ALIGNMENT == 16             \
                                                ? largebin_index_32_big(sz)    \
                                                : largebin_index_32(sz))
```

#### Unsorted Bin

The unsorted bin can be viewed as a buffer for free `chunk`s before they return to their respective bins.

Its specific description in glibc is as follows:

```c++
/*
   Unsorted chunks

    All remainders from chunk splits, as well as all returned chunks,
    are first placed in the "unsorted" bin. They are then placed
    in regular bins after malloc gives them ONE chance to be used before
    binning. So, basically, the unsorted_chunks list acts as a queue,
    with chunks being placed on it in free (and malloc_consolidate),
    and taken off (to be either used or placed in bins) in malloc.

    The NON_MAIN_ARENA flag is never set for unsorted chunks, so it
    does not have to be taken into account in size comparisons.
 */
```

From the following macro we can see:

```c++
/* The otherwise unindexable 1-bin is used to hold unsorted chunks. */
#define unsorted_chunks(M) (bin_at(M, 1))
```

The unsorted bin is at index 1 of the bin array we mentioned earlier. Therefore, the unsorted bin has only one linked list. Free `chunk`s in the unsorted bin are in a disordered state and mainly come from two sources:

- When a larger `chunk` is split in half, if the remaining part is larger than MINSIZE, it will be placed into the unsorted bin.
- When a chunk that does not belong to a fast bin is freed, and it is not adjacent to the top `chunk`, it will first be placed into the unsorted bin. For an explanation of the top `chunk`, please refer to the introduction below.

Additionally, the unsorted bin uses FIFO traversal order during use.

#### common macro

Here we introduce some common macros.

**Uniformly Get the Index of the Bin a chunk Belongs to Based on Its Size**

```c++
#define bin_index(sz)                                                          \
    ((in_smallbin_range(sz)) ? smallbin_index(sz) : largebin_index(sz))
```

### Top Chunk

The description of the top `chunk` in glibc is as follows:

```c++
/*
   Top

    The top-most available chunk (i.e., the one bordering the end of
    available memory) is treated specially. It is never included in
    any bin, is used only if no other chunk is available, and is
    released back to the system if it is very large (see
    M_TRIM_THRESHOLD).  Because top initially
    points to its own bin with initial zero size, thus forcing
    extension on the first malloc request, we avoid having any special
    code in malloc to check whether it even exists yet. But we still
    need to do so when getting memory from system, so we make
    initial_top treat the bin as a legal but unusable chunk during the
    interval between initialization and the first call to
    sysmalloc. (This is somewhat delicate, since it relies on
    the 2 preceding words to be zero during this interval as well.)
 */

/* Conveniently, the unsorted bin can be used as dummy top on first call */
#define initial_top(M) (unsorted_chunks(M))
```

When the program calls malloc for the first time, the heap is divided into two parts: one is given to the user, and the remaining part becomes the top chunk. In essence, the top `chunk` is the chunk located at the highest physical address of the current heap. This `chunk` does not belong to any bin. Its role is that when all bins cannot satisfy the user's requested size, if the top chunk's size is not less than the requested size, it performs the allocation and makes the remaining part the new top chunk. Otherwise, the heap is extended before allocation. In the main arena, the heap is extended via sbrk, while in thread arenas, new heaps are allocated via mmap.

It is important to note that the top `chunk`'s prev_inuse bit is always set to 1; otherwise, the chunk in front of it would be merged into the top chunk.

**Initially, we can treat the unsorted `chunk` as the top chunk.**

### last remainder

When the user uses malloc to request memory allocation, the `chunk` found by ptmalloc2 may not exactly match the requested memory size. In this case, the remaining part after splitting is called the last remainder `chunk`, and the unsorted bin also stores this piece. The remaining part from splitting the top `chunk` does not become the last remainder.

## Macroscopic Structures

### arena

In the examples we introduced earlier, whether it is the main thread or a newly created thread, each has an independent arena when first requesting memory. But does every thread have an independent arena? Let's discuss this in detail below.

#### Arena Count

For different systems, the [constraints](https://github.com/sploitfun/lsploits/blob/master/glibc/malloc/arena.c#L847) on the number of arenas are as follows:

```text
For 32 bit systems:
     Number of arena = 2 * number of cores.
For 64 bit systems:
     Number of arena = 8 * number of cores.
```

Clearly, not every thread will have a corresponding arena. As for why 64-bit systems are configured that way, I haven't figured it out either. Additionally, since the number of cores in each system is limited, when the number of threads exceeds twice the number of cores (with hyper-threading), some threads will inevitably be in a waiting state, so there is no need to allocate an arena for every thread.

#### Arena Allocation Rules

**To be supplemented.**

#### Differences

Unlike thread arenas, main_arena is not part of the allocated heap but is a global variable located in the data segment of libc.so.

### heap_info

When a program first starts executing, each thread does not have a heap region. When it requests memory, a structure is needed to record the corresponding information — this is what heap_info is for. Moreover, when that heap's resources are exhausted, memory must be requested again. Additionally, since heaps requested at different times are generally not contiguous, the linking structure between different heaps needs to be recorded.

**This data structure is specifically prepared for memory requested from the Memory Mapping Segment, i.e., for non-main threads.**

The main thread can obtain memory by extending the program break location via the `sbrk()` function (until it reaches the Memory Mapping Segment). It has only one heap and does not have the heap_info data structure.

The main structure of heap_info is as follows:

```c++
#define HEAP_MIN_SIZE (32 * 1024)
#ifndef HEAP_MAX_SIZE
# ifdef DEFAULT_MMAP_THRESHOLD_MAX
#  define HEAP_MAX_SIZE (2 * DEFAULT_MMAP_THRESHOLD_MAX)
# else
#  define HEAP_MAX_SIZE (1024 * 1024) /* must be a power of two */
# endif
#endif

/* HEAP_MIN_SIZE and HEAP_MAX_SIZE limit the size of mmap()ed heaps
   that are dynamically created for multi-threaded programs.  The
   maximum size must be a power of two, for fast determination of
   which heap belongs to a chunk.  It should be much larger than the
   mmap threshold, so that requests with a size just below that
   threshold can be fulfilled without creating too many heaps.  */

/***************************************************************************/

/* A heap is a single contiguous memory region holding (coalesceable)
   malloc_chunks.  It is allocated with mmap() and always starts at an
   address aligned to HEAP_MAX_SIZE.  */

typedef struct _heap_info
{
  mstate ar_ptr; /* Arena for this heap. */
  struct _heap_info *prev; /* Previous heap. */
  size_t size;   /* Current size in bytes. */
  size_t mprotect_size; /* Size in bytes that has been mprotected
                           PROT_READ|PROT_WRITE.  */
  /* Make sure the following data is properly aligned, particularly
     that sizeof (heap_info) + 2 * SIZE_SZ is a multiple of
     MALLOC_ALIGNMENT. */
  char pad[-6 * SIZE_SZ & MALLOC_ALIGN_MASK];
} heap_info;
```

This structure mainly describes basic information about the heap, including:

- The address of the arena corresponding to the heap.
- Since a thread may exhaust its heap after requesting one, it must request memory again. Therefore, a thread may have multiple heaps. prev records the address of the previous heap_info. Here we can see that each heap's heap_info is linked via a singly-linked list.
- size represents the current size of the heap.
- The last part ensures alignment.

!!! note "What is the reason for the negative number in pad?"
    `pad` is used to ensure that the allocated space is aligned to `MALLOC_ALIGN_MASK+1` (denoted as `MALLOC_ALIGN_MASK_1`). Before `pad`, the structure has a total of 6 members of `SIZE_SZ` size. To ensure `MALLOC_ALIGN_MASK_1` byte alignment, padding may be needed. Assuming the final size of the structure is `MALLOC_ALIGN_MASK_1*x`, where `x` is a natural number, the space needed for `pad` is `MALLOC_ALIGN_MASK_1 * x - 6 * SIZE_SZ = (MALLOC_ALIGN_MASK_1 * x - 6 * SIZE_SZ) % MALLOC_ALIGN_MASK_1 = 0 - 6 * SIZE_SZ % MALLOC_ALIGN_MASK_1 = -6 * SIZE_SZ % MALLOC_ALIGN_MASK_1 = -6 * SIZE_SZ & MALLOC_ALIGN_MASK`.

This structure seems like it should be quite important, but if we carefully read through the entire malloc implementation, we'll find that it doesn't appear very frequently.

### malloc_state

This structure is used to manage the heap and records the specific state of memory currently requested by each arena, such as whether there are free chunks, what sizes of free chunks exist, etc. Whether it is a thread arena or the main arena, they each have only one malloc_state structure. Since a thread's arena may have multiple instances, the malloc_state structure will be in the most recently requested arena.

**Note that main arena's malloc_state is not part of the heap segment, but is a global variable stored in the data segment of libc.so.**

Its structure is as follows:

```c++
struct malloc_state {
    /* Serialize access.  */
    __libc_lock_define(, mutex);

    /* Flags (formerly in max_fast).  */
    int flags;

    /* Fastbins */
    mfastbinptr fastbinsY[ NFASTBINS ];

    /* Base of the topmost chunk -- not otherwise kept in a bin */
    mchunkptr top;

    /* The remainder from the most recent split of a small request */
    mchunkptr last_remainder;

    /* Normal bins packed as described above */
    mchunkptr bins[ NBINS * 2 - 2 ];

    /* Bitmap of bins, help to speed up the process of determinating if a given bin is definitely empty.*/
    unsigned int binmap[ BINMAPSIZE ];

    /* Linked list, points to the next arena */
    struct malloc_state *next;

    /* Linked list for free arenas.  Access to this field is serialized
       by free_list_lock in arena.c.  */
    struct malloc_state *next_free;

    /* Number of threads attached to this arena.  0 if the arena is on
       the free list.  Access to this field is serialized by
       free_list_lock in arena.c.  */
    INTERNAL_SIZE_T attached_threads;

    /* Memory allocated from the system in this arena.  */
    INTERNAL_SIZE_T system_mem;
    INTERNAL_SIZE_T max_system_mem;
};
```

-   __libc_lock_define(, mutex);
    -   This variable is used to control serialized access to the same arena. When a thread acquires the arena, other threads that want to access the arena must wait until that thread completes its allocation.

-   flags
    -   flags records some flags of the arena. For example, bit0 records whether the arena has `fast bin chunk`s, and bit1 indicates whether the arena can return contiguous virtual address space. Specifically:

```c

/*
   FASTCHUNKS_BIT held in max_fast indicates that there are probably
   some fastbin chunks. It is set true on entering a chunk into any
   fastbin, and cleared only in malloc_consolidate.
   The truth value is inverted so that have_fastchunks will be true
   upon startup (since statics are zero-filled), simplifying
   initialization checks.
 */

#define FASTCHUNKS_BIT (1U)

#define have_fastchunks(M) (((M)->flags & FASTCHUNKS_BIT) == 0)
#define clear_fastchunks(M) catomic_or(&(M)->flags, FASTCHUNKS_BIT)
#define set_fastchunks(M) catomic_and(&(M)->flags, ~FASTCHUNKS_BIT)

/*
   NONCONTIGUOUS_BIT indicates that MORECORE does not return contiguous
   regions.  Otherwise, contiguity is exploited in merging together,
   when possible, results from consecutive MORECORE calls.
   The initial value comes from MORECORE_CONTIGUOUS, but is
   changed dynamically if mmap is ever used as an sbrk substitute.
 */

#define NONCONTIGUOUS_BIT (2U)

#define contiguous(M) (((M)->flags & NONCONTIGUOUS_BIT) == 0)
#define noncontiguous(M) (((M)->flags & NONCONTIGUOUS_BIT) != 0)
#define set_noncontiguous(M) ((M)->flags |= NONCONTIGUOUS_BIT)
#define set_contiguous(M) ((M)->flags &= ~NONCONTIGUOUS_BIT)

/* ARENA_CORRUPTION_BIT is set if a memory corruption was detected on the
   arena.  Such an arena is no longer used to allocate chunks.  Chunks
   allocated in that arena before detecting corruption are not freed.  */

#define ARENA_CORRUPTION_BIT (4U)

#define arena_is_corrupt(A) (((A)->flags & ARENA_CORRUPTION_BIT))
#define set_arena_corrupt(A) ((A)->flags |= ARENA_CORRUPTION_BIT)

```

-   fastbinsY[NFASTBINS]
    -   Stores the head pointers of each fast `chunk` linked list.
-   top
    -   Points to the arena's top chunk.
-   last_reminder
    -   The remaining part after the most recent `chunk` split.
-   bins
    -   Used to store the `chunk` linked lists for unsorted bin, small bins, and large bins.
-   binmap
    -   ptmalloc uses one bit to indicate whether a certain bin contains free `chunk`s.

### malloc_par

**!! To be supplemented !!**
