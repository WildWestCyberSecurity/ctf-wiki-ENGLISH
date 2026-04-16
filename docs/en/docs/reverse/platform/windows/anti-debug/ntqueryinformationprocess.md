# NtQueryInformationProcess


``` c++
NTSTATUS WINAPI NtQueryInformationProcess(
  _In_      HANDLE           ProcessHandle,
  _In_      PROCESSINFOCLASS ProcessInformationClass,
  _Out_     PVOID            ProcessInformation,
  _In_      ULONG            ProcessInformationLength,
  _Out_opt_ PULONG           ReturnLength
);
```

## ProcessDebugPort

The undocumented `NtQueryInformationProcess()` function from `ntdll` accepts an information class parameter for querying. `ProcessDebugPort(7)` is one such information class. The `CheckRemoteDebuggerPresent()` function from `kernel32` internally calls `NtQueryInformationProcess()` to detect debugging, and `NtQueryInformationProcess` internally queries the `DebugPort` field of the `EPROCESS` structure. When the process is being debugged, the return value is `0xffffffff`.

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
push eax
mov eax, esp
push 0
push 4 ;ProcessInformationLength
push eax
push 7 ;ProcessDebugPort
push -1 ;GetCurrentProcess()
call NtQueryInformationProcess
pop eax
inc eax
je being_debugged
```

The following 64-bit code can be used to detect in a 64-bit environment:

``` asm
xor ebp, ebp
enter 20h, 0
push 8 ;ProcessInformationLength
pop r9
push rbp
pop r8
push 7 ;ProcessDebugPort
pop rdx
or rcx, -1 ;GetCurrentProcess()
call NtQueryInformationProcess
leave
test ebp, ebp
jne being_debugged
```

Since the information comes from the kernel, there is no easy way in user-mode code to prevent this function from detecting the debugger.

## ProcessDebugObjectHandle

Windows XP introduced `debug objects`. When a debug session starts, a `debug` object is created along with an associated handle. We can use the `ProcessDebugObjectHandle (0x1e)` class to query the value of this handle.

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
push 0
mov eax, esp
push 0
push 4 ;ProcessInformationLength
push eax
push 1eh ;ProcessDebugObjectHandle
push -1 ;GetCurrentProcess()
call NtQueryInformationProcess
pop eax
test eax, eax
jne being_debugged
```

The following 64-bit code can be used to detect in a 64-bit environment:

``` asm
xor ebp, ebp
enter 20h, 0
push 8 ;ProcessInformationLength
pop r9
push rbp
pop r8
push 1eh ;ProcessDebugObjectHandle
pop rdx
or rcx, -1 ;GetCurrentProcess()
call NtQueryInformationProcess
leave
test ebp, ebp
jne being_debugged
```

## ProcessDebugFlags

The `ProcessDebugFlags (0x1f)` class returns the negation of the `NoDebugInherit` field from the `EPROCESS` structure. This means that when a debugger is present, the return value is `0`, and when no debugger is present, it returns `1`.

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
push eax
mov eax, esp
push 0
push 4 ;ProcessInformationLength
push eax
push 1fh ;ProcessDebugFlags
push -1 ;GetCurrentProcess()
call NtQueryInformationProcess
pop eax
test eax, eax
je being_debugged
```

The following 64-bit code can be used to detect in a 64-bit environment:

``` asm
xor ebp, ebp
enter 20h, 0
push 4 ;ProcessInformationLength
pop r9
push rbp
pop r8
push 1fh ;ProcessDebugFlags
pop rdx
or rcx, -1 ;GetCurrentProcess()
call NtQueryInformationProcess
leave
test ebp, ebp
je being_debugged
```
