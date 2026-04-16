# Detecting Debugging

There are many methods to detect debuggers, such as checking process names. Here we introduce one method: detecting whether the program is currently being debugged by checking the call status of certain functions.

```c 
int main()
{
	if (ptrace(PTRACE_TRACEME, 0, 1, 0) < 0) {
		printf("DEBUGGING... Bye\n");
		return 1;
	}
	printf("Hello\n");
	return 0;
}
```

A process can only be ptraced by one process. If you call ptrace yourself, other programs cannot debug or inject code into your program via ptrace.

If the program is currently being debugged by gdb, the ptrace function will return an error, which indirectly indicates the presence of a debugger.

## Bypass Method 1

Obviously, ptrace only works against debuggers that use ptrace. We can use a debugger that doesn't use ptrace.

We can also patch out the ptrace function, or more simply, erase the ptrace call code or the subsequent validation.

If the executable was compiled without the -s option (the -s option removes all symbol table information and relocation information) — which is unlikely in real situations — things become much simpler. Let's analyze from this simple case:

```
# objdump -t test_debug | grep ptrace
080482c0 	F *UND* 	00000075 	ptrace@@GLIBC_2.0
```

ptrace is called at position `0x080482c0`

```
# objdump -d -M intel test_debug |grep 80482c0
80482c0: 	ff 25 04 96 04 08 	jmp ds:0x8049604
80483d4: 	e8 e7 fe ff ff 	call 80482c0 <_init+0x28>
```

What if the -s option was enabled? In that case we need to use gdb:

```
# gdb test_debug
gdb> bp ptrace
Breakpoint 1 at 0x80482c0
gdb> run
Breakpoint 1 at 0x400e02f0
......
0x400e02f0 <ptrace>: push %ebp
0x400e02f1 <ptrace+1>: mov %esp,%ebp
0x400e02f3 <ptrace+3>: sub $0x10,%esp
0x400e02f6 <ptrace+6>: mov %edi,0xfffffffc(%ebp)
0x400e02f9 <ptrace+9>: mov 0x8(%ebp),%edi
0x400e02fc <ptrace+12>: mov 0xc(%ebp),%ecx
------------------------------------------------------------------------------
Breakpoint 1, 0x400e02f0 in ptrace () from /lib/tls/libc.so.6
```

We simply break at ptrace. Now type finish to execute until the current function returns, back to the main function:

```
# gdb test_debug
gdb> finish
00x80483d9 <main+29>: 	add $0x10,%esp
0x80483dc   <main+32>: 	test %eax,%eax
0x80483de   <main+34>: 	jns 0x80483fa <main+62>
0x80483e0   <main+36>: 	sub $0xc,%esp
0x80483e3   <main+39>: 	push $0x80484e8
0x80483e8   <main+44>: 	call 0x80482e0
------------------------------------------------------------------------------
0x080483d9 in main ()
```

Modify the function return value eax to the correct return result, and we're done:

```
gdb> set $eax=0
gdb> c
everything ok
Program exited with code 016.
_______________________________________________________________________________
No registers.
gdb>
```

## Bypass Method 2

Method 2 is to write your own ptrace function.

As described in previous sections, the `LD_PRELOAD` environment variable can redirect the executable to our own ptrace function.

We write a ptrace function and generate the object file:

``` c
// -- ptrace.c --
// gcc -shared ptrace.c -o ptrace.so
int ptrace(int i, int j, int k, int l)
{
	printf(" PTRACE CALLED!\n");
}
```

Next, we can use our own ptrace function by setting the LD_PRELOAD environment variable. Of course, this can also be set within gdb:

```
gdb> set environment LD_PRELOAD ./ptrace.so
gdb> run
PTRACE CALLED!
Hello World!
Program exited with code 015.
gdb>
```

As you can see, the program can no longer detect the debugger.



> Reference: [Beginners Guide to Basic Linux Anti Anti Debugging Techniques](http://www.stonedcoder.org/~kd/lib/14-61-1-PB.pdf)



