# Basic Operations

## unlink

unlink is used to remove an element from a doubly-linked list (which only stores free chunks). It may be used in the following places:

- malloc
    - Getting a chunk from a large bin of exactly the right size.
        - **Note that fastbin and small bin do not use unlink, which is why vulnerabilities often appear in them.**
        - Unlink is also not used when iterating through the unsorted bin.
    - Getting a chunk from a bin larger than the one where the requested chunk would belong.
- free
    - Backward consolidation: merging a physically adjacent free chunk at a lower address.
    - Forward consolidation: merging a physically adjacent free chunk at a higher address (except for the top chunk).
- malloc_consolidate
    - Backward consolidation: merging a physically adjacent free chunk at a lower address.
    - Forward consolidation: merging a physically adjacent free chunk at a higher address (except for the top chunk).
- realloc
    - Forward expansion: merging a physically adjacent free chunk at a higher address (except for the top chunk).

Since unlink is used very frequently, it is implemented as a macro, as follows:

```c
/* Take a chunk off a bin list */
// unlink p
#define unlink(AV, P, BK, FD) {                                            \
    // Since P is already in the doubly-linked list, its size is recorded in two places, so check whether the sizes are consistent.
    if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");			      \
    FD = P->fd;                                                                      \
    BK = P->bk;                                                                      \
    // Prevent attackers from simply tampering with the fd and bk of a free chunk to achieve arbitrary write.
    if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
      malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \
    else {                                                                      \
        FD->bk = BK;                                                              \
        BK->fd = FD;                                                              \
        // The following mainly considers modifications to P's nextsize doubly-linked list
        if (!in_smallbin_range (chunksize_nomask (P))                              \
            // If P->fd_nextsize is NULL, it means P has not been inserted into the nextsize linked list.
            // In that case, there is no need to modify the nextsize fields.
            // The bk_nextsize field is not checked here, which may cause issues.
            && __builtin_expect (P->fd_nextsize != NULL, 0)) {                      \
            // Similar check logic as for small chunks
            if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)              \
                || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    \
              malloc_printerr (check_action,                                      \
                               "corrupted double-linked list (not small)",    \
                               P, AV);                                              \
            // This indicates that P is already in the nextsize linked list.
            // If FD is not in the nextsize linked list
            if (FD->fd_nextsize == NULL) {                                      \
                // If the doubly-linked list formed by nextsize only contains P itself, just remove P
                // and make FD the new nextsize list head
                if (P->fd_nextsize == P)                                      \
                  FD->fd_nextsize = FD->bk_nextsize = FD;                      \
                else {                                                              \
                // Otherwise, we need to insert FD into the doubly-linked list formed by nextsize
                    FD->fd_nextsize = P->fd_nextsize;                              \
                    FD->bk_nextsize = P->bk_nextsize;                              \
                    P->fd_nextsize->bk_nextsize = FD;                              \
                    P->bk_nextsize->fd_nextsize = FD;                              \
                  }                                                              \
              } else {                                                              \
                // If it is already in the list, just remove it directly
                P->fd_nextsize->bk_nextsize = P->bk_nextsize;                      \
                P->bk_nextsize->fd_nextsize = P->fd_nextsize;                      \
              }                                                                      \
          }                                                                      \
      }                                                                              \
}
```

Here we use the small bin unlink as an example. For large bin unlink, it is similar but with additional handling of nextsize.

![](./figure/unlink_smallbin_intro.png)

As we can see, **P's fd and bk pointers do not change in the end**, but when we traverse the entire doubly-linked list, P can no longer be found. The fact that these pointers remain unchanged is quite useful, because sometimes we can use this method to leak addresses:

- libc address
    - P is at the head of the doubly-linked list: leak via bk
    - P is at the tail of the doubly-linked list: leak via fd
    - The doubly-linked list contains only one free chunk, P is in the list: both fd and bk can be used for leaking
- Heap address leak, when the doubly-linked list contains multiple free chunks
    - P is at the head of the doubly-linked list: leak via fd
    - P is in the middle of the doubly-linked list: both fd and bk can be used for leaking
    - P is at the tail of the doubly-linked list: leak via bk

**Note**

- The head here refers to the chunk pointed to by the bin's fd, i.e., the most recently added chunk in the doubly-linked list.
- The tail here refers to the chunk pointed to by the bin's bk, i.e., the earliest added chunk in the doubly-linked list.

At the same time, for fd, bk, fd_nextsize, and bk_nextsize, the program checks whether fd and bk satisfy the corresponding requirements.

```c
// fd bk
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
  malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \

  // next_size related
              if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)              \
                || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    \
              malloc_printerr (check_action,                                      \
                               "corrupted double-linked list (not small)",    \
                               P, AV);
```

This looks quite normal. Taking fd and bk as an example, the bk of P's forward chunk is naturally P, and similarly the fd of P's backward chunk is naturally P. If these checks were not in place, we could modify P's fd and bk to easily achieve arbitrary address write. For more detailed examples, refer to the unlink section in the exploitation part.

**Note: The prev_inuse bit recorded by the first chunk of the heap is set to 1 by default.**

## malloc_printerr

When an error is detected during glibc malloc, the `malloc_printerr` function is called.

```cpp
static void malloc_printerr(const char *str) {
  __libc_message(do_abort, "%s\n", str);
  __builtin_unreachable();
}
```

It mainly calls `__libc_message` to execute the `abort` function, as follows:

```c
  if ((action & do_abort)) {
    if ((action & do_backtrace))
      BEFORE_ABORT(do_abort, written, fd);

    /* Kill the application.  */
    abort();
  }
```

In the `abort` function, when glibc is still version 2.23, it will fflush streams.

```c
  /* Flush all streams.  We cannot close them now because the user
     might have registered a handler for SIGABRT.  */
  if (stage == 1)
    {
      ++stage;
      fflush (NULL);
    }
```
