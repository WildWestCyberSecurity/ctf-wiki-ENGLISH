# ZwSetInformationThread

## About ZwSetInformationThread

ZwSetInformationThread is equivalent to NtSetInformationThread. By setting ThreadHideFromDebugger for a thread, it can prevent the thread from generating debug events. The code is as follows:  
```c
#include <Windows.h>
#include <stdio.h>

typedef DWORD(WINAPI* ZW_SET_INFORMATION_THREAD) (HANDLE, DWORD, PVOID, ULONG);
#define ThreadHideFromDebugger 0x11
VOID DisableDebugEvent(VOID)
{
    HINSTANCE hModule;
    ZW_SET_INFORMATION_THREAD ZwSetInformationThread;
    hModule = GetModuleHandleA("Ntdll");
    ZwSetInformationThread = (ZW_SET_INFORMATION_THREAD)GetProcAddress(hModule, "ZwSetInformationThread");
    ZwSetInformationThread(GetCurrentThread(), ThreadHideFromDebugger, 0, 0);
}

int main()
{
    printf("Begin\n");
    DisableDebugEvent();
    printf("End\n");
    return 0;
}
```

The key line of code is `ZwSetInformationThread(GetCurrentThread(), ThreadHideFromDebugger, 0, 0);`. If the program is being debugged, execution of this line will cause the program to exit.  

## How to Bypass

Note that the second parameter of the ZwSetInformationThread function is ThreadHideFromDebugger, with a value of 0x11. When debugging and reaching this function, if the second parameter value is 0x11, you can either skip over it or modify 0x11 to another value to bypass the check.
