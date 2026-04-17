# Heap Flags

## About Heap Flags

`Heap flags` contain two flags that are initialized together with `NtGlobalFlag`: `Flags` and `ForceFlags`. The values of these two fields are not only affected by the debugger, but also vary depending on the Windows version. The position of these fields also depends on the Windows version.

* Flags field:
    * In 32-bit Windows NT, Windows 2000, and Windows XP, `Flags` is located at offset `0x0C` of the heap. In 32-bit Windows Vista and newer systems, it is located at offset `0x40`.
    * In 64-bit Windows XP, the `Flags` field is located at offset `0x14` of the heap, while in 64-bit Windows Vista and newer systems, it is located at offset `0x70`.
* ForceFlags field:
    * In 32-bit Windows NT, Windows 2000, and Windows XP, `ForceFlags` is located at offset `0x10` of the heap. In 32-bit Windows Vista and newer systems, it is located at offset `0x44`.
    * In 64-bit Windows XP, the `ForceFlags` field is located at offset `0x18` of the heap, while in 64-bit Windows Vista and newer systems, it is located at offset `0x74`.

In all versions of Windows, the `Flags` field value is normally set to `HEAP_GROWABLE(2)`, and the `ForceFlags` field is normally set to `0`. However, for a 32-bit process (64-bit programs are not affected by this), these two default values depend on the [`subsystem`](https://msdn.microsoft.com/en-us/library/ms933120.aspx) version of its host process (this does not refer to something like Windows 10's Linux subsystem). Only when the `subsystem` version is `3.51` or higher will the default values be as described above. If the version is between `3.10-3.50`, the `HEAP_CREATE_ALIGN_16 (0x10000)` flag will be set for both fields. If the version is lower than `3.10`, the program file will not run at all.

If an operation sets the `Flags` and `ForceFlags` field values to `2` and `0` respectively, but does not check the `subsystem` version, it indicates that the action was performed to hide a debugger.

When a debugger is present, on `Windows NT`, `Windows 2000`, and 32-bit `Windows XP` systems, the `Flags` field will have the following flags set:

``` c
HEAP_GROWABLE (2)
HEAP_TAIL_CHECKING_ENABLED (0x20)
HEAP_FREE_CHECKING_ENABLED (0x40)
HEAP_SKIP_VALIDATION_CHECKS (0x10000000)
HEAP_VALIDATE_PARAMETERS_ENABLED (0x40000000)
```

On 64-bit `Windows XP`, `Windows Vista`, and newer system versions, the `Flags` field will have the following flags set (without `HEAP_SKIP_VALIDATION_CHECKS (0x10000000)`):

``` c
HEAP_GROWABLE (2)
HEAP_TAIL_CHECKING_ENABLED (0x20)
HEAP_FREE_CHECKING_ENABLED (0x40)
HEAP_VALIDATE_PARAMETERS_ENABLED (0x40000000)
```

For the `ForceFlags` field, it will normally have the following flags set:

``` c
HEAP_TAIL_CHECKING_ENABLED (0x20)
HEAP_FREE_CHECKING_ENABLED (0x40)
HEAP_VALIDATE_PARAMETERS_ENABLED (0x40000000)
```

Because of the `NtGlobalFlag` flags, the `heap` will also have some flags set:

* If the `FLG_HEAP_ENABLE_TAIL_CHECK` flag is set in the `NtGlobalFlag` field, then the `HEAP_TAIL_CHECKING_ENABLED` flag will be set in the `heap` field.
* If the `FLG_HEAP_ENABLE_FREE_CHECK` flag is set in the `NtGlobalFlag` field, then the `FLG_HEAP_ENABLE_FREE_CHECK` flag will be set in the `heap` field.
* If the `FLG_HEAP_VALIDATE_PARAMETERS` flag is set in the `NtGlobalFlag` field, then the `HEAP_VALIDATE_PARAMETERS_ENABLED` flag will be set in the `heap` field (on `Windows NT` and `Windows 2000`, the `HEAP_CREATE_ALIGN_16 (0x10000)` flag will also be set).

`Heap flags`, similar to the `NtGlobalFlag` described in the previous section, are controlled by the `PageHeapFlags` key in the registry at `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<filename>`.

## Obtaining the Heap Location

There are multiple methods to obtain the location of the `heap`. One method is the `GetProcessHeap()` function from `kernel32`. Of course, you can also use the following 32-bit assembly code to detect in a 32-bit environment (in fact, some packers avoid using this API function and directly query the PEB):

``` asm
mov eax, fs:[30h] ;Process Environment Block
mov eax, [eax+18h] ;get process heap base
```

Or use the following 64-bit code to detect in a 64-bit environment:

``` asm
push 60h
pop rsi
gs:lodsq ;Process Environment Block
mov eax, [rax+30h] ;get process heap base
```

Or use the following 32-bit code to detect in a 64-bit environment:

``` asm
mov eax, fs:[30h] ;Process Environment Block
;64-bit Process Environment Block
;follows 32-bit Process Environment Block
mov eax, [eax+1030h] ;get process heap base
```

Another method is to use the `GetProcessHeaps()` function from `kernel32`, which simply forwards the call to `ntdll`'s `RtlGetProcessHeaps()` function. This function returns an array of heaps belonging to the current process, and the first heap in the array is the same one returned by `kernel32`'s `GetProcessHeap()` function.



This process can be implemented using 32-bit code to detect a 32-bit Windows environment:

``` asm
push 30h
pop esi
fs:lodsd ;Process Environment Block
;get process heaps list base
mov esi, [esi+eax+5ch]
lodsd
```

Similarly, using 64-bit code to detect a 64-bit Windows environment:

``` asm
push 60h
pop rsi
gs:lodsq ;Process Environment Block
;get process heaps list base
mov esi, [rsi*2+rax+20h]
lodsd
```

Or using 32-bit code to detect a 64-bit Windows environment:

``` asm
mov eax, fs:[30h] ;Process Environment Block
;64-bit Process Environment Block
;follows 32-bit Process Environment Block
mov esi, [eax+10f0h] ;get process heaps list base
lodsd
```

## Checking the Flags Field

Obviously, to detect a debugger, we can start by checking the flag bits of `Flags` and `ForceFlags`.

First, let's look at the `Flags` field detection code, using 32-bit code to detect a 32-bit Windows environment with `subsystem` version between `3.10-3.50`:

``` asm
call GetVersion
cmp al, 6
cmc
sbb ebx, ebx
and ebx, 34h
mov eax, fs:[30h] ;Process Environment Block
mov eax, [eax+18h] ;get process heap base
mov eax, [eax+ebx+0ch] ;Flags
;neither HEAP_CREATE_ALIGN_16
;nor HEAP_SKIP_VALIDATION_CHECKS
and eax, 0effeffffh
;HEAP_GROWABLE
;+ HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp eax, 40000062h
je being_debugged
```

32-bit code to detect a 32-bit Windows environment with `subsystem` version `3.51` and higher:

``` asm
call GetVersion
cmp al, 6
cmc
sbb ebx, ebx
and ebx, 34h
mov eax, fs:[30h] ;Process Environment Block
mov eax, [eax+18h] ;get process heap base
mov eax, [eax+ebx+0ch] ;Flags
;not HEAP_SKIP_VALIDATION_CHECKS
bswap eax
and al, 0efh
;HEAP_GROWABLE
;+ HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
;reversed by bswap
cmp eax, 62000040h
je being_debugged
```

64-bit code to detect a 64-bit Windows environment (64-bit processes are not affected by the `subsystem` version):

``` asm
push 60h
pop rsi
gs:lodsq ;Process Environment Block
mov ebx, [rax+30h] ;get process heap base
call GetVersion
cmp al, 6
sbb rax, rax
and al, 0a4h
;HEAP_GROWABLE
;+ HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp d [rbx+rax+70h], 40000062h ;Flags
je being_debugged
```

Using 32-bit code to detect a 64-bit Windows environment:

``` asm
push 30h
pop eax
mov ebx, fs:[eax] ;Process Environment Block
;64-bit Process Environment Block
;follows 32-bit Process Environment Block
mov ah, 10h
mov ebx, [ebx+eax] ;get process heap base
call GetVersion
cmp al, 6
sbb eax, eax
and al, 0a4h
;Flags
;HEAP_GROWABLE
;+ HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp [ebx+eax+70h], 40000062h
je being_debugged
```

If the value is obtained directly through the `NtMajorVersion` field of the `KUSER_SHARED_DATA` structure (located at offset `0x7ffe026c` in the 2GB user space) — this value is available on all 32-bit/64-bit versions of Windows — it can further obfuscate the `kernel32` `GetVersion()` function call.


## Checking the ForceFlags Field

Of course, another method is to check the `ForceFlags` field. The following is 32-bit code to detect a 32-bit Windows environment with `subsystem` version between `3.10-3.50`:

``` asm
call GetVersion
cmp al, 6
cmc
sbb ebx, ebx
and ebx, 34h
mov eax, fs:[30h] ;Process Environment Block
mov eax, [eax+18h] ;get process heap base
mov eax, [eax+ebx+10h] ;ForceFlags
;not HEAP_CREATE_ALIGN_16
btr eax, 10h
;HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp eax, 40000060h
je being_debugged
```

32-bit code to detect a 32-bit Windows environment with `subsystem` version `3.51` and higher:

``` asm
call GetVersion
cmp al, 6
cmc
sbb ebx, ebx
and ebx, 34h
mov eax, fs:[30h] ;Process Environment Block
mov eax, [eax+18h] ;get process heap base
;ForceFlags
;HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp [eax+ebx+10h], 40000060h
je being_debugged
```

64-bit code to detect a 64-bit Windows environment (64-bit processes are not affected by the `subsystem` version):

``` asm
push 60h
pop rsi
gs:lodsq ;Process Environment Block
mov ebx, [rax+30h] ;get process heap base
call GetVersion
cmp al, 6
sbb rax, rax
and al, 0a4h
;ForceFlags
;HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp d [rbx+rax+74h], 40000060h
je being_debugged
```
Using 32-bit code to detect a 64-bit Windows environment:

``` asm
call GetVersion
cmp al, 6
push 30h
pop eax
mov ebx, fs:[eax] ;Process Environment Block
;64-bit Process Environment Block
;follows 32-bit Process Environment Block
mov ah, 10h
mov ebx, [ebx+eax] ;get process heap base
sbb eax, eax
and al, 0a4h
;ForceFlags
;HEAP_TAIL_CHECKING_ENABLED
;+ HEAP_FREE_CHECKING_ENABLED
;+ HEAP_VALIDATE_PARAMETERS_ENABLED
cmp [ebx+eax+74h], 40000060h
je being_debugged
```

## References

* [The "Ultimate" Anti-Debugging Reference](http://anti-reversing.com/Downloads/Anti-Reversing/The_Ultimate_Anti-Reversing_Reference.pdf)
