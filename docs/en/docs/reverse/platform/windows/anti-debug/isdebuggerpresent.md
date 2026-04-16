# IsDebuggerPresent

## About IsDebuggerPresent

When a debugger is present, the `IsDebuggerPresent()` function from `kernel32` returns a `non-zero value`.

``` c++
BOOL WINAPI IsDebuggerPresent(void);
```

## Detection Code

Its detection method is very simple. For example, the following code (the same code works for both 32-bit and 64-bit) can be used to detect in 32-bit/64-bit environments:

``` asm
call IsDebuggerPresent
test al, al
jne being_debugged
```

In fact, this function simply returns the value of the `BeingDebugged` flag. The method of checking the `BeingDebugged` flag can also be implemented using the following 32-bit code to check a 32-bit environment:

``` asm
mov eax, fs:[30h] ;Process Environment Block
cmp b [eax+2], 0 ;check BeingDebugged
jne being_debugged
```

Or using 64-bit code to detect a 64-bit environment:

``` asm
push 60h
pop rsi
gs:lodsq ;Process Environment Block
cmp b [rax+2], 0 ;check BeingDebugged
jne being_debugged
```

Or using 32-bit code to detect a 64-bit environment:

``` asm
mov eax, fs:[30h] ;Process Environment Block
;64-bit Process Environment Block
;follows 32-bit Process Environment Block
cmp b [eax+1002h], 0 ;check BeingDebugged
jne being_debugged
```

## How to Bypass

To overcome these checks, simply set the `BeingDebugged` flag to `0` (or change the return value).
