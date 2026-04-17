# CheckRemoteDebuggerPresent

## About CheckRemoteDebuggerPresent

The `CheckRemoteDebuggerPresent()` function from `kernel32` is used to detect whether a specified process is being debugged. The word `Remote` here refers to a different process on the same machine.

``` c
BOOL WINAPI CheckRemoteDebuggerPresent(
  _In_    HANDLE hProcess,
  _Inout_ PBOOL  pbDebuggerPresent
);
```

If a debugger is present (typically used to check whether the current process itself is being debugged), the function sets the value pointed to by `pbDebuggerPresent` to `0xffffffff`.

## Detection Code

The following 32-bit code can be used to detect in a 32-bit environment:

``` asm
push eax
push esp
push -1 ;GetCurrentProcess()
call CheckRemoteDebuggerPresent
pop eax
test eax, eax
jne being_debugged
```

Or use 64-bit code to detect in a 64-bit environment:

``` asm
enter 20h, 0
mov edx, ebp
or rcx, -1 ;GetCurrentProcess()
call CheckRemoteDebuggerPresent
leave
test ebp, ebp
jne being_debugged
```

## How to Bypass

For example, given the following code:

``` c++
int main(int argc, char *argv[])
{
    BOOL isDebuggerPresent = FALSE;
    if (CheckRemoteDebuggerPresent(GetCurrentProcess(), &isDebuggerPresent ))
    {
        if (isDebuggerPresent )
        {
            std::cout << "Stop debugging program!" << std::endl;
            exit(-1);
        }
    }
    return 0;
}
```

We can directly modify the value of `isDebuggerPresent` or change the jump condition to bypass the check (note: this is not the return value of `CheckRemoteDebuggerPresent` — its return value indicates whether the function executed successfully).

However, if you want to specifically modify the `CheckRemoteDebuggerPresent` API function itself, you first need to know that `CheckRemoteDebuggerPresent` internally works by calling `NtQueryInformationProcess` to accomplish its functionality. Therefore, we need to modify the return value of `NtQueryInformationProcess`. We will cover this in the [NtQueryInformationProcess section](./ntqueryinformationprocess.md).
