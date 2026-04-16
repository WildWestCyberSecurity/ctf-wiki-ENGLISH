# Junk Code

## Introduction

Junk code is a type of instruction fragment specifically designed to confuse decompilers. These instruction fragments do not affect the program's original functionality but cause deviations in the disassembler's results, leading to failed analysis by crackers. Classic junk code techniques include using `jmp`, `call`, and `ret` instructions to alter execution flow, causing the disassembler to parse incorrect code that doesn't match runtime behavior.

## Example: N1CTF2020 - oflo

### Reverse Analysis

As usual, we drag it into IDA and find that the `main()` function cannot be decompiled. Examining the assembly code reveals an in-place `jmp` at `0x400BB1` that causes disassembly errors:

```c
.text:0000000000400B54 ; int __fastcall main(int, char **, char **)
.text:0000000000400B54 main:                                   ; DATA XREF: start+1D↑o
.text:0000000000400B54                                         ; .text:0000000000400C21↓o
.text:0000000000400B54 ; __unwind {
.text:0000000000400B54                 push    rbp
.text:0000000000400B55                 mov     rbp, rsp
.text:0000000000400B58                 sub     rsp, 240h
.text:0000000000400B5F                 mov     rax, fs:28h
.text:0000000000400B68                 mov     [rbp-8], rax
.text:0000000000400B6C                 xor     eax, eax
.text:0000000000400B6E                 lea     rdx, [rbp-210h]
.text:0000000000400B75                 mov     eax, 0
.text:0000000000400B7A                 mov     ecx, 40h ; '@'
.text:0000000000400B7F                 mov     rdi, rdx
.text:0000000000400B82                 rep stosq
.text:0000000000400B85                 mov     qword ptr [rbp-230h], 0
.text:0000000000400B90                 mov     qword ptr [rbp-228h], 0
.text:0000000000400B9B                 mov     qword ptr [rbp-220h], 0
.text:0000000000400BA6                 mov     qword ptr [rbp-218h], 0
.text:0000000000400BB1
.text:0000000000400BB1 loc_400BB1:                             ; CODE XREF: .text:loc_400BB1↑j
.text:0000000000400BB1                 jmp     short near ptr loc_400BB1+1
.text:0000000000400BB3 ; ---------------------------------------------------------------------------
.text:0000000000400BB3                 ror     byte ptr [rax-70h], 90h
.text:0000000000400BB7                 call    loc_400BBF
.text:0000000000400BB7 ; ---------------------------------------------------------------------------
.text:0000000000400BBC                 db 0E8h, 0EBh, 12h
.text:0000000000400BBF ; ---------------------------------------------------------------------------
```

Change the first byte at `0x400BB1` to `0x90` (`nop`), then continue disassembly. Next we encounter a strange call:

```c
.text:0000000000400BB1                 nop
.text:0000000000400BB2                 inc     eax
.text:0000000000400BB4                 xchg    rax, rax
.text:0000000000400BB6                 nop
.text:0000000000400BB7                 call    loc_400BBF
.text:0000000000400BB7 ; ---------------------------------------------------------------------------
.text:0000000000400BBC                 db 0E8h, 0EBh, 12h
.text:0000000000400BBF ; ---------------------------------------------------------------------------
.text:0000000000400BBF
.text:0000000000400BBF loc_400BBF:                             ; CODE XREF: .text:0000000000400BB7↑j
.text:0000000000400BBF                 pop     rax
.text:0000000000400BC0                 add     rax, 1
.text:0000000000400BC4                 push    rax
.text:0000000000400BC5                 mov     rax, rsp
.text:0000000000400BC8                 xchg    rax, [rax]
.text:0000000000400BCB                 pop     rsp
.text:0000000000400BCC                 mov     [rsp], rax
.text:0000000000400BD0                 retn
```

The `call` instruction pushes the address of the next instruction (`0x400BBC`) onto the stack. In the code fragment at `0x400BBF`, the return address is popped from the stack, incremented by one, pushed back, and then `retn` is executed. Therefore, the actual execution flow starts from `0x400BBD`. So we can patch both the `call` instruction and code fragment `0x400BBF` to `nop`:

```python
import idc

for i in range(0x400BB7, 0x400BBC + 1):
    idc.patch_byte(i, 0x90)

for i in range(0x400BBF, 0x400BD0 + 1):
    idc.patch_byte(i, 0x90)
```

After patching, the logic is straightforward — it jumps to `0x400BD1`, where `sub_4008B9()` is called followed by `exit()`:

```c
.text:0000000000400BB6                 nop
.text:0000000000400BB7                 nop
.text:0000000000400BB8                 nop
.text:0000000000400BB9                 nop
.text:0000000000400BBA                 nop
.text:0000000000400BBB                 nop
.text:0000000000400BBC                 nop
.text:0000000000400BBD                 jmp     short loc_400BD1
.text:0000000000400BBF ; ---------------------------------------------------------------------------
.text:0000000000400BBF
.text:0000000000400BBF loc_400BBF:                             ; CODE XREF: .text:0000000000400BB7↑j
.text:0000000000400BBF                 nop
.text:0000000000400BC0                 nop
.text:0000000000400BC1                 nop
.text:0000000000400BC2                 nop
.text:0000000000400BC3                 nop
.text:0000000000400BC4                 nop
.text:0000000000400BC5                 nop
.text:0000000000400BC6                 nop
.text:0000000000400BC7                 nop
.text:0000000000400BC8                 nop
.text:0000000000400BC9                 nop
.text:0000000000400BCA                 nop
.text:0000000000400BCB                 nop
.text:0000000000400BCC                 nop
.text:0000000000400BCD                 nop
.text:0000000000400BCE                 nop
.text:0000000000400BCF                 nop
.text:0000000000400BD0                 nop
.text:0000000000400BD1 ; ---------------------------------------------------------------------------
.text:0000000000400BD1
.text:0000000000400BD1 loc_400BD1:                             ; CODE XREF: .text:0000000000400BBD↑j
.text:0000000000400BD1                 lea     rax, [rbp-210h]
.text:0000000000400BD8                 mov     rdi, rax
.text:0000000000400BDB                 call    sub_4008B9
.text:0000000000400BE0                 cmp     eax, 0FFFFFFFFh
.text:0000000000400BE3                 jnz     short loc_400BEF
.text:0000000000400BE5                 mov     edi, 0
.text:0000000000400BEA                 call    exit
```

At this point `main()` still cannot be decompiled with `F5`. We continue looking downward for more junk code and find the same obfuscation at `0x400CB5`:

```c
.text:0000000000400CB5                 call    loc_400CBD
.text:0000000000400CB5 ; ---------------------------------------------------------------------------
.text:0000000000400CBA                 dw 0EBE8h
.text:0000000000400CBC                 db 12h
.text:0000000000400CBD ; ---------------------------------------------------------------------------
.text:0000000000400CBD
.text:0000000000400CBD loc_400CBD:                             ; CODE XREF: .text:0000000000400CB5↑j
.text:0000000000400CBD                 pop     rax
.text:0000000000400CBE                 add     rax, 1
.text:0000000000400CC2                 push    rax
.text:0000000000400CC3                 mov     rax, rsp
.text:0000000000400CC6                 xchg    rax, [rax]
.text:0000000000400CC9                 pop     rsp
.text:0000000000400CCA                 mov     [rsp], rax
.text:0000000000400CCE                 retn
```

Directly patch to `nop`:

```python
import idc

for i in range(0x400CB5, 0x400CBA + 1):
    idc.patch_byte(i, 0x90)

for i in range(0x400CBD, 0x400CCE + 1):
    idc.patch_byte(i, 0x90)
```

We get a jump to `0x400CCF`:

```c
.text:0000000000400CB9                 nop
.text:0000000000400CBA                 nop
.text:0000000000400CBB                 jmp     short loc_400CCF
.text:0000000000400CBD ; ---------------------------------------------------------------------------
.text:0000000000400CBD
.text:0000000000400CBD loc_400CBD:                             ; CODE XREF: .text:0000000000400CB5↑j
.text:0000000000400CBD                 nop
.text:0000000000400CBE                 nop
.text:0000000000400CBF                 nop
.text:0000000000400CC0                 nop
```

Continuing downward, at `0x400D04` we find another very classic junk code pattern. Again, just patch the first byte to `nop`:

```c
.text:0000000000400D04 loc_400D04:                             ; CODE XREF: .text:0000000000400CEE↑j
.text:0000000000400D04                                         ; .text:loc_400D04↑j
.text:0000000000400D04                 jmp     short near ptr loc_400D04+1
.text:0000000000400D04 ; ---------------------------------------------------------------------------
.text:0000000000400D06                 db 0C0h
.text:0000000000400D07                 db  48h ; H
.text:0000000000400D08                 db  90h
.text:0000000000400D09                 db  90h
.text:0000000000400D0A                 db 0BFh
```

Now it looks like a perfectly normal function ending:

```c
.text:0000000000400D04 loc_400D04:                             ; CODE XREF: .text:0000000000400CEE↑j
.text:0000000000400D04                 nop
.text:0000000000400D05                 inc     eax
.text:0000000000400D07                 xchg    rax, rax
.text:0000000000400D09                 nop
.text:0000000000400D0A                 mov     edi, 0
.text:0000000000400D0F                 call    exit
.text:0000000000400D14 ; ---------------------------------------------------------------------------
.text:0000000000400D14                 nop
.text:0000000000400D15                 mov     rax, [rbp-8]
.text:0000000000400D19                 xor     rax, fs:28h
.text:0000000000400D22                 jz      short locret_400D29
.text:0000000000400D24                 call    ___stack_chk_fail
.text:0000000000400D29 ; ---------------------------------------------------------------------------
.text:0000000000400D29
.text:0000000000400D29 locret_400D29:                          ; CODE XREF: .text:0000000000400D22↑j
.text:0000000000400D29                 leave
.text:0000000000400D2A                 retn
.text:0000000000400D2A ; } // starts at 400B54
```

Now we go back to the beginning of `main`, press `p` to recreate the function, and then we can decompile with `F5` normally. The `main()` logic is as follows:

- First calls `sub_4008B9()`
- Then reads 19 bytes from input
- Calls `mprotect()` to change permissions at `main & 0xFFFFC000` to `r | w | x`. Since permission control granularity is at the memory page level, this actually modifies the permissions of an entire memory page
- Modifies the first 10 bytes of `sub_400A69()`
- Calls `sub_400A69()` to check the flag

```c
void __fastcall __noreturn main(int a1, char **a2, char **a3)
{
  int v3; // ecx
  int v4; // er8
  int v5; // er9
  int i; // [rsp+4h] [rbp-23Ch]
  __int64 v7[4]; // [rsp+10h] [rbp-230h] BYREF
  char v8[520]; // [rsp+30h] [rbp-210h] BYREF
  unsigned __int64 v9; // [rsp+238h] [rbp-8h]

  v9 = __readfsqword(0x28u);
  memset(v8, 0, 0x200uLL);
  v7[0] = 0LL;
  v7[1] = 0LL;
  v7[2] = 0LL;
  v7[3] = 0LL;
  if ( (unsigned int)sub_4008B9(v8, a2, v8) == -1 )
    exit(0LL);
  read(0LL, v7, 19LL);
  qword_602048 = (__int64)sub_400A69;
  mprotect((unsigned int)main & 0xFFFFC000, 16LL, 7LL);
  for ( i = 0; i <= 9; ++i )
  {
    v3 = i % 5;
    *(_BYTE *)(qword_602048 + i) ^= *((_BYTE *)v7 + i % 5);
  }
  if ( (unsigned int)sub_400A69((unsigned int)v8, (unsigned int)v7 + 5, (unsigned int)v8, v3, v4, v5) )
    write(1LL, "Cong!\n", 6LL);
  exit(0LL);
}
```

`sub_4008B9()` calls `fork()` to create parent and child processes. The child process requests the parent process to debug and execute `/bin/cat /proc/version`:

```c
__int64 __fastcall sub_4008B9(__int64 a1)
{
  unsigned int v2; // [rsp+14h] [rbp-5Ch] BYREF
  int v3; // [rsp+18h] [rbp-58h]
  unsigned int v4; // [rsp+1Ch] [rbp-54h]
  __int64 v5; // [rsp+20h] [rbp-50h]
  __int64 v6; // [rsp+28h] [rbp-48h]
  __int64 v7; // [rsp+30h] [rbp-40h]
  __int64 v8; // [rsp+38h] [rbp-38h]
  __int64 v9; // [rsp+40h] [rbp-30h] BYREF
  __int64 v10[4]; // [rsp+50h] [rbp-20h] BYREF

  v10[3] = __readfsqword(0x28u);
  v4 = fork();
  if ( (v4 & 0x80000000) != 0 )
    v2 = -1;
  if ( !v4 )
  {
    v10[0] = (__int64)&unk_400DB8;
    v10[1] = (__int64)"/proc/version";
    v10[2] = 0LL;
    v9 = 0LL;
    ptrace(PTRACE_TRACEME, 0LL, 0LL, 0LL);
    execve("/bin/cat", v10, &v9);
    exit(127LL);
  }
```

The parent process **performs single-step debugging using individual system calls as step granularity**. The `PTRACE_PEEKUSER` here retrieves the value of the child process's corresponding register based on the provided offset. The relationship between offsets and registers can be found in the kernel source at `/arch/x86/include/asm/user_64.h` in the `user_regs_struct` structure.

Here offset `120` retrieves `orig_rax`, which is the **system call number. In other words, the parent process only enters the core logic when the system call number is `1` (i.e., `write`): it retrieves `rsi` (offset `104`) and `rdx` (offset `96`) and calls `sub_4007D1()`**:

```c
  v3 = 0;
  v5 = a1;
  while ( 1 )
  {
    wait4(v4, &v2, 0LL, 0LL);
    if ( (v2 & 0x7F) == 0 )
      break;
    v6 = ptrace(PTRACE_PEEKUSER, v4, 120LL, 0LL);
    if ( v6 == 1 )
    {
      if ( v3 )
      {
        v3 = 0;
      }
      else
      {
        v3 = 1;
        v7 = ptrace(PTRACE_PEEKUSER, v4, 104LL, 0LL);
        v8 = ptrace(PTRACE_PEEKUSER, v4, 96LL, 0LL);
        sub_4007D1(v4, v7, v5, v8);
        v5 += v8;
      }
    }
    ptrace(PTRACE_SYSCALL, v4, 0LL, 0LL);
  }
  return v2;
}
```

`sub_4007D1()` is relatively simple — it mainly copies the output from the child process's `write()` call back to the parent process:

```c
__int64 __fastcall sub_4007D1(unsigned int a1, __int64 a2, __int64 a3, __int64 a4)
{
  __int64 result; // rax
  int v6; // [rsp+20h] [rbp-20h]
  int v7; // [rsp+24h] [rbp-1Ch]

  v6 = 0;
  v7 = a4 / 8;
  while ( v6 < v7 )
  {
    *(_QWORD *)(4 * v6 + a3) = ptrace(PTRACE_PEEKDATA, a1, a2 + 4 * v6, 0LL);
    ++v6;
  }
  result = a4 % 8;
  if ( (unsigned int)(a4 % 8) )
  {
    result = ptrace(PTRACE_PEEKDATA, a1, a2 + 4 * v6, 0LL);
    *(_QWORD *)(4 * v6 + a3) = result;
  }
  return result;
}
```

Next, returning to `main()`, since `sub_400A69()` will be modified at runtime, we need to apply the modifications directly in IDA to get correct disassembly results. It takes the first 5 bytes of the flag and performs operations with the code section. Since the first 5 bytes of the flag are always `n1ctf`, we fix `sub_400A69()` like this:

```python
import idc

s = 'n1ctf'

for i in range(0x400A69, 0x400A69 + 10):
    c = idc.get_db_byte(i)
    c ^= ord(s[(i - 0x400A69) % 5])
    idc.patch_byte(i, c)
```

Inside `sub_400A69()` there is also a pseudo junk instruction. Here we simply patch the `jmp` at `0x400AC4` to `nop`:

```c
.text:0000000000400A69 sub_400A69      proc near               ; CODE XREF: main+193↓p
.text:0000000000400A69                                         ; DATA XREF: main+B4↓o
.text:0000000000400A69
.text:0000000000400A69 var_40          = qword ptr -40h
.text:0000000000400A69
.text:0000000000400A69 ; __unwind {
.text:0000000000400A69                 push    rbp
.text:0000000000400A6A                 mov     rbp, rsp
.text:0000000000400A6D                 sub     rsp, 40h
.text:0000000000400A71                 mov     [rbp-38h], rdi
.text:0000000000400A75                 mov     [rbp-40h], rsi
.text:0000000000400A79                 mov     rax, fs:28h
.text:0000000000400A82                 mov     [rbp-8], rax
.text:0000000000400A86                 xor     eax, eax
.text:0000000000400A88                 mov     byte ptr [rbp-20h], 35h ; '5'
.text:0000000000400A8C                 mov     byte ptr [rbp-1Fh], 2Dh ; '-'
.text:0000000000400A90                 mov     byte ptr [rbp-1Eh], 11h
.text:0000000000400A94                 mov     byte ptr [rbp-1Dh], 1Ah
.text:0000000000400A98                 mov     byte ptr [rbp-1Ch], 49h ; 'I'
.text:0000000000400A9C                 mov     byte ptr [rbp-1Bh], 7Dh ; '}'
.text:0000000000400AA0                 mov     byte ptr [rbp-1Ah], 11h
.text:0000000000400AA4                 mov     byte ptr [rbp-19h], 14h
.text:0000000000400AA8                 mov     byte ptr [rbp-18h], 2Bh ; '+'
.text:0000000000400AAC                 mov     byte ptr [rbp-17h], 3Bh ; ';'
.text:0000000000400AB0                 mov     byte ptr [rbp-16h], 3Eh ; '>'
.text:0000000000400AB4                 mov     byte ptr [rbp-15h], 3Dh ; '='
.text:0000000000400AB8                 mov     byte ptr [rbp-14h], 3Ch ; '<'
.text:0000000000400ABC                 mov     byte ptr [rbp-13h], 5Fh ; '_'
.text:0000000000400AC0                 jz      short loc_400AC9
.text:0000000000400AC2                 jnz     short loc_400AC9
.text:0000000000400AC4                 jmp     near ptr 801AC9h
```

At `0x400B0E` there is yet another instance of the same junk code pattern, continue patching:

```c
.text:0000000000400B0E                 call    loc_400B16
.text:0000000000400B0E ; ---------------------------------------------------------------------------
.text:0000000000400B13                 db 0E8h
.text:0000000000400B14                 db 0EBh
.text:0000000000400B15                 db  12h
.text:0000000000400B16 ; ---------------------------------------------------------------------------
.text:0000000000400B16
.text:0000000000400B16 loc_400B16:                             ; CODE XREF: sub_400A69+A5↑j
.text:0000000000400B16                 pop     rax
.text:0000000000400B17                 add     rax, 1
.text:0000000000400B1B                 push    rax
.text:0000000000400B1C                 mov     rax, rsp
.text:0000000000400B1F                 xchg    rax, [rax]
.text:0000000000400B22                 pop     rsp
.text:0000000000400B23                 mov     [rsp+40h+var_40], rax
.text:0000000000400B27                 retn
```

Now we can properly decompile `sub_400A69()`. The core logic is actually very simple. Note that the flag passed in from `main()` starts after `n1ctf`:

```c
__int64 __fastcall sub_400A69(__int64 a1, __int64 a2)
{
  __int64 v2; // rbp
  int i; // [rsp+14h] [rbp-2Ch]
  char v5[8]; // [rsp+18h] [rbp-28h]
  _BYTE v6[6]; // [rsp+20h] [rbp-20h] BYREF
  unsigned __int64 v7; // [rsp+30h] [rbp-10h]
  __int64 v8; // [rsp+38h] [rbp-8h]

  v8 = v2;
  v7 = __readfsqword(0x28u);
  v5[0] = 53;
  v5[1] = 45;
  v5[2] = 17;
  v5[3] = 26;
  v5[4] = 73;
  v5[5] = 125;
  v5[6] = 17;
  v5[7] = 20;
  qmemcpy(v6, "+;>=<_", sizeof(v6));
  for ( i = 0; i <= 13; ++i )
  {
    if ( v5[i] != ((*(char *)(i + a1) + 2) ^ *(char *)(i + a2)) )
      return 0LL;
  }
  return 1LL;
}
```

### Solution

Since the first 14 bytes of `/proc/version` are always `"Linux version "`, we can easily obtain the flag content:

```python
s = "5-\x11\x1AI}\x11\x14+;>=<_"
b = "Linux version "
ans = ""

for i in range(14):
    ans += chr(ord(s[i]) ^ (ord(b[i]) + 2))
print(ans)
# {Fam3_is_NULL}
```
