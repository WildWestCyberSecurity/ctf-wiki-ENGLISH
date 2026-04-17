# The Heap

During initialization, the heap checks the `heap flags` and makes additional changes to the environment depending on whether certain flags are set. `Themida` is one example that uses this method to detect debuggers.

For example:

* If the `HEAP_TAIL_CHECKING_ENABLED` flag is set (see the `Heap Flags` section), then in 32-bit Windows, two `0xABABABAB` values are appended to the end of allocated heap blocks (four in 64-bit environments).
* If the `HEAP_FREE_CHECKING_ENABLED` flag is set (see the `Heap Flags` section), then when extra bytes are needed to fill the tail of a heap block, `0xFEEEFEEE` (or part of it) is used for padding.

Therefore, a new method of detecting debuggers is to check for these values.

## Heap Pointer Known

If a heap pointer is known, we can directly check the data in the heap block. However, on `Windows Vista` and later versions, heap protection is used (for both 32-bit and 64-bit), employing an XOR key to encrypt the heap block size. While you can choose whether to use the key, it is used by default. Additionally, the position of the heap block header differs between `Windows NT/2000/XP` and `Windows Vista and later`. Therefore, the `Windows version` must also be taken into account.

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
    xor ebx, ebx
    call GetVersion
    cmp al, 6
    sbb ebp, ebp
    jb l1
    ;Process Environment Block
    mov eax, fs:[ebx+30h]
    mov eax, [eax+18h] ;get process heap base
    mov ecx, [eax+24h] ;check for protected heap
    jecxz l1
    mov ecx, [ecx]
    test [eax+4ch], ecx
    cmovne ebx, [eax+50h] ;conditionally get heap key
l1: mov eax, <heap ptr>
    movzx edx, w [eax-8] ;size
    xor dx, bx
    movzx ecx, b [eax+ebp-1] ;overhead
    sub eax, ecx
    lea edi, [edx*8+eax]
    mov al, 0abh
    mov cl, 8
    repe scasb
    je being_debugged
```

Or use the following 64-bit code to detect in a 64-bit environment:

```
    xor ebx, ebx
    call GetVersion
    cmp al, 6
    sbb rbp, rbp
    jb l1
    ;Process Environment Block
    mov rax, gs:[rbx+60h]
    mov eax, [rax+30h] ;get process heap base
    mov ecx, [rax+40h] ;check for protected heap
    jrcxz l1
    mov ecx, [rcx+8]
    test [rax+7ch], ecx
    cmovne ebx, [rax+88h] ;conditionally get heap key
l1: mov eax, <heap ptr>
    movzx edx, w [rax-8] ;size
    xor dx, bx
    add edx, edx
    movzx ecx, b [rax+rbp-1] ;overhead
    sub eax, ecx
    lea edi, [rdx*8+rax]
    mov al, 0abh
    mov cl, 10h
    repe scasb
    je being_debugged
```

There is no example of using 32-bit code to detect a 64-bit environment here, because the 64-bit heap cannot be parsed by 32-bit heap functions.


## Heap Pointer Unknown

If the heap pointer is unknown, we can use `kernel32`'s `HeapWalk()` function or `ntdll`'s `RtlWalkHeap()` function (or even `kernel32`'s `GetCommandLine()` function). The returned heap size value is automatically decrypted, so there is no need to worry about the Windows version.

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
    mov ebx, offset l2
    ;get a pointer to a heap block
l1: push ebx
    mov eax, fs:[30h] ;Process Environment Block
    push d [eax+18h] ;save process heap base
    call HeapWalk
    cmp w [ebx+0ah], 4 ;find allocated block
    jne l1
    mov edi, [ebx] ;data pointer
    add edi, [ebx+4] ;data size
    mov al, 0abh
    push 8
    pop ecx
    repe scasb
    je being_debugged
    ...
l2: db 1ch dup (0) ;sizeof(PROCESS_HEAP_ENTRY)
```

Or use the following 64-bit code to detect in a 64-bit environment:

``` asm
    mov rbx, offset l2
    ;get a pointer to a heap block
l1: push rbx
    pop rdx
    push 60h
    pop rsi
    gs:lodsq ;Process Environment Block
    ;get a pointer to process heap base
    mov ecx, [rax+30h]
    call HeapWalk
    cmp w [rbx+0eh], 4 ;find allocated block
    jne l1
    mov edi, [rbx] ;data pointer
    add edi, [rbx+8] ;data size
    mov al, 0abh
    push 10h
    pop rcx
    repe scasb
    je being_debugged
    ...
l2: db 28h dup (0) ;sizeof(PROCESS_HEAP_ENTRY)
```

There is no example of using 32-bit code to detect a 64-bit environment here, because the 64-bit heap cannot be parsed by 32-bit heap functions.
