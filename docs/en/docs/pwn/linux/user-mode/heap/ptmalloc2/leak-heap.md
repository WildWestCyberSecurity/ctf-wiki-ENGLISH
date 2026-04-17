# Information Leakage Through the Heap

## What is Information Leakage
In CTF, Pwn challenges are usually running on a remote server. Therefore, we cannot know address information such as the libc.so address or the heap base address on the server. However, exploitation often requires these addresses, which is when information leakage becomes necessary.

## Targets of Information Leakage
What are the targets of information leakage? We can find out by examining the memory space:

```
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-x /home/pwn
0x0000000000600000 0x0000000000601000 0x0000000000000000 r-- /home/pwn
0x0000000000601000 0x0000000000602000 0x0000000000001000 rw- /home/pwn
0x0000000000602000 0x0000000000623000 0x0000000000000000 rw- [heap]
0x00007ffff7a0d000 0x00007ffff7bcd000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7bcd000 0x00007ffff7dcd000 0x00000000001c0000 --- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dcd000 0x00007ffff7dd1000 0x00000000001c0000 r-- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dd1000 0x00007ffff7dd3000 0x00000000001c4000 rw- /lib/x86_64-linux-gnu/libc-2.23.so
0x00007ffff7dd3000 0x00007ffff7dd7000 0x0000000000000000 rw- 
0x00007ffff7dd7000 0x00007ffff7dfd000 0x0000000000000000 r-x /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7fdb000 0x00007ffff7fde000 0x0000000000000000 rw- 
0x00007ffff7ff6000 0x00007ffff7ff8000 0x0000000000000000 rw- 
0x00007ffff7ff8000 0x00007ffff7ffa000 0x0000000000000000 r-- [vvar]
0x00007ffff7ffa000 0x00007ffff7ffc000 0x0000000000000000 r-x [vdso]
0x00007ffff7ffc000 0x00007ffff7ffd000 0x0000000000025000 r-- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7ffd000 0x00007ffff7ffe000 0x0000000000026000 rw- /lib/x86_64-linux-gnu/ld-2.23.so
0x00007ffff7ffe000 0x00007ffff7fff000 0x0000000000000000 rw- 
0x00007ffffffde000 0x00007ffffffff000 0x0000000000000000 rw- [stack]
0xffffffffff600000 0xffffffffff601000 0x0000000000000000 r-x [vsyscall]
```
First is the base address of the main module. Since the base address of the main module only changes when PIE (Position Independent Executable) is enabled, the main module address usually does not need to be leaked.
Second is the heap address. The heap address changes with each process execution, so when we need to control data in the heap, we may need to leak the heap base address first.
Third is the libc.so address. In many cases, we can only achieve code execution through functions like system in libc, and structures such as malloc_hook, one_gadgets, and IO_FILE are also stored in libc. Therefore, the libc address is also a target for leaking.

## What Can Be Used for Leaking
From our previous knowledge, we know that the heap is divided into unsorted bin, fastbin, smallbin, large bin, etc. Let's examine each of these structures to see how leaking can be done.

## unsorted bin
We construct two unsorted bins and examine their memory. Now there are two chunks in the unsorted bin list: the first chunk is at address 0x602000 and the second is at address 0x6020f0.

```
0x602000:	0x0000000000000000	0x00000000000000d1
0x602010:	0x00007ffff7dd1b78	0x00000000006020f0 <=== points to next chunk
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
```

```
0x6020f0:	0x0000000000000000	0x00000000000000d1
0x602100:	0x0000000000602000	0x00007ffff7dd1b78 <=== points to main_arena
0x602110:	0x0000000000000000	0x0000000000000000
0x602120:	0x0000000000000000	0x0000000000000000
```
Therefore, we know that through the unsorted bin we can obtain the address of a certain heap chunk and the address of main_arena. Once we obtain the address of a certain heap chunk, we can calculate the heap base address using the malloc size. Once we obtain the main_arena address, since main_arena resides in libc.so, we can calculate the offset to derive the base address of libc.so.
Therefore, through the unsorted bin we can obtain: 1. The base address of libc.so 2. The heap base address

## fastbin
We construct two fastbins and examine their memory. Now there are two chunks in the fastbin list: the first chunk is at address 0x602040 and the second is at address 0x602000.

```
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x0000000000000000	0x0000000000000000
```

```
0x602040:	0x0000000000000000	0x0000000000000021
0x602050:	0x0000000000602000 	0x0000000000000000 <=== points to the first chunk
```
From our previous knowledge, we know that the fd field of the last chunk in a fastbin list is 0, and each subsequent chunk's fd field points to the previous chunk. Therefore, through fastbin we can only leak the heap base address.

## smallbin
We construct two smallbins and examine their memory. Now there are two chunks in the smallbin list: the first chunk is at address 0x602000 and the second is at address 0x6020f0.
```
0x602000:	0x0000000000000000	0x00000000000000d1
0x602010:	0x00007ffff7dd1c38	0x00000000006020f0 <=== address of the next chunk
0x602020:	0x0000000000000000	0x0000000000000000
0x602030:	0x0000000000000000	0x0000000000000000
```

```
0x6020f0:	0x0000000000000000	0x00000000000000d1
0x602100:	0x0000000000602000	0x00007ffff7dd1c38 <=== address of main_arena
0x602110:	0x0000000000000000	0x0000000000000000
0x602120:	0x0000000000000000	0x0000000000000000
```
Therefore, through smallbin we can obtain: 1. The base address of libc.so 2. The heap base address

## Which Vulnerabilities Can Be Used for Leaking
From our previous knowledge, we can understand what address information exists in the heap, but obtaining these addresses requires exploitation through vulnerabilities.
Generally speaking, the following vulnerabilities can be used for information leakage:

* Uninitialized heap memory
* Heap overflow
* Use-After-Free
* Out-of-bounds read
* Heap extend 

###  0x01 read UAF

Leaking heapbase through UAF:

```c
p0 = malloc(0x20);
p1 = malloc(0x20);

free(p0);
free(p1);
    
printf('heap base:%p',*p1);
```

 Due to the characteristics of the fastbin list, when we construct a fastbin list:

```bash
(0x30)     fastbin[1]: 0x602030 --> 0x602000 --> 0x0
```

There exists the phenomenon of chunk 1 -> chunk 0. If a UAF vulnerability exists at this point, we can print out the address of chunk 0 by showing chunk 1.



Similarly, leaking libc base:

```c
p0 = malloc(0x100);
free(p0);
printf("libc: %p\n", *p0);

```



### 0x02  overlapping chunks





### 0x03 Partial Overwrite



### 0x04 Relative Write
