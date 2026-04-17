# Heap Checks

## _int_malloc

## Initial Checks

| Check Target  |                   Check Condition                   |         Message          |
| :---: | :--------------------------------------: | :-----------------: |
| Requested size | REQUEST_OUT_OF_RANGE(req) : ((unsigned long) (req) >= (unsigned long) (INTERNAL_SIZE_T)(-2 * MINSIZE)) | __set_errno(ENOMEM) |

### fastbin

| Check Target     |                  Check Condition                   |                Error Message                |
| -------- | :-------------------------------------: | :--------------------------------: |
| chunk size | fastbin_index(chunksize(victim)) != idx | malloc(): memory corruption (fast) |

### Unsorted bin

|         Check Target          |                   Check Condition                   |            Error Message             |
| :-------------------: | :--------------------------------------: | :-------------------------: |
| unsorted bin chunk size | chunksize_nomask (victim) <= 2 * SIZE_SZ \|\| chunksize_nomask (victim)  av->system_mem | malloc(): memory corruption |



### top chunk

|      Check Target      |                   Check Condition                   |  Message  |
| :------------: | :--------------------------------------: | :--: |
| top chunk size | (unsigned long) (size) >= (unsigned long) (nb + MINSIZE) | Can proceed |



## __libc_free

### mmap chunk

|      Check Target      |         Check Condition         |  Message  |
| :------------: | :------------------: | :--: |
| chunk size flag bit | chunk_is_mmapped (p) | Can proceed |

### non-mmap chunk

## __int_free

### Initial Checks

|    Check Target    |                   Check Condition                   |          Error Message           |
| :--------: | :--------------------------------------: | :---------------------: |
| Freed chunk location  | (uintptr_t) p > (uintptr_t) -size \|\| misaligned_chunk(p) | free(): invalid pointer |
| Freed chunk size |  size < MINSIZE \|\| !aligned_OK(size)   |  free(): invalid size   |

### fastbin

|         Check Target          |                   Check Condition                   |                Error Message                 |
| :-------------------: | :--------------------------------------: | :---------------------------------: |
|  Next chunk size of freed chunk   | chunksize_nomask(chunk_at_offset(p, size)) <= 2 * SIZE_SZ, chunksize(chunk_at_offset(p, size)) >= av->system_mem |  free(): invalid next size (fast)   |
| First chunk in the freed chunk's corresponding list | fb = &fastbin(av, idx), old= *fb, old == p | double free or corruption (fasttop) |
|       fastbin index       |      old != NULL && old_idx != idx       |    invalid fastbin entry (free)     |

### non-mmapped chunk checks

|     Check Target      |                   Check Condition                   |                Error Message                |
| :-----------: | :--------------------------------------: | :--------------------------------: |
|   Freed chunk location   |               p == av->top               |  double free or corruption (top)   |
| next chunk location | contiguous (av) && (char *) nextchunk  >= ((char *) av->top + chunksize(av->top)) |  double free or corruption (out)   |
| next chunk size | chunksize_nomask (nextchunk) <= 2 * SIZE_SZ \|\|  nextsize >= av->system_mem | free(): invalid next size (normal) |

## unlink

|         Check Target          |                   Check Condition                   |                   Error Message                   |
| :-------------------: | :--------------------------------------: | :--------------------------------------: |
| size **vs** prev_size | chunksize(P) != prev_size (next_chunk(P)) |       corrupted size vs. prev_size       |
|     Fd, bk doubly-linked list check     |       FD->bk != P \|\| BK->fd != P       |       corrupted double-linked list       |
|     nextsize doubly-linked list     | P->fd_nextsize->bk_nextsize != P \|\| P->bk_nextsize->fd_nextsize != P | corrupted double-linked list (not small) |
