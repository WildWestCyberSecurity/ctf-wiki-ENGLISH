# Junk Code

## Principle

Junk code is a method used to hide code blocks (or other functionalities) from reverse engineering. It inserts garbage code into the real code while ensuring the original program still executes correctly. However, the program cannot be decompiled properly, making it difficult to understand the program's content — achieving the goal of obfuscation.

Junk code is typically used to increase the difficulty of static analysis.

## Writing Junk Code

The simplest junk code uses inline assembly. Below is an example of adding junk code in VC. GNU compilers can also add junk code in a similar way, but using AT&T assembly syntax:

```c
// Normal function code
int add(int a, int b){
  int c = 0;
  c = a + b;
  return c;
}
// Function code with junk code added
int add_with_junk(int a, int b){
	int c = 0;
	__asm{
		jz label;
		jnz label;
		_emit 0xe8;    call instruction, followed by a 4-byte address offset, causing the disassembler to fail to recognize properly
label:
	}
	c = a + b;
	return c;
}

```

When decompiling with IDA, the function with junk code cannot be properly recognized. The result is as follows:

Pseudo-code:

```asm
// With junk code added
.text:00401070 loc_401070:                             ; CODE XREF: sub_401005↑j
.text:00401070                 push    ebp
.text:00401071                 mov     ebp, esp
.text:00401073                 sub     esp, 44h
.text:00401076                 push    ebx
.text:00401077                 push    esi
.text:00401078                 push    edi
.text:00401079                 lea     edi, [ebp-44h]
.text:0040107C                 mov     ecx, 11h
.text:00401081                 mov     eax, 0CCCCCCCCh
.text:00401086                 rep stosd
.text:00401088                 mov     dword ptr [ebp-4], 0
.text:0040108F                 jz      short near ptr loc_401093+1
.text:00401091                 jnz     short near ptr loc_401093+1
.text:00401093
.text:00401093 loc_401093:                             ; CODE XREF: .text:0040108F↑j
.text:00401093                                         ; .text:00401091↑j
.text:00401093                 call    near ptr 3485623h
.text:00401098                 inc     ebp
.text:00401099                 or      al, 89h
.text:0040109B                 inc     ebp
.text:0040109C                 cld
.text:0040109D                 mov     eax, [ebp-4]
.text:004010A0                 pop     edi
.text:004010A1                 pop     esi
.text:004010A2                 pop     ebx
.text:004010A3                 add     esp, 44h
.text:004010A6                 cmp     ebp, esp
.text:004010A8                 call    __chkesp
.text:004010AD                 mov     esp, ebp
.text:004010AF                 pop     ebp
.text:004010B0                 retn
```

In the example above, patching the obfuscating junk instructions with NOPs will fix it, and then normal analysis can proceed.

It is worth noting that IDA's stack analysis is quite strict, so junk instructions involving `push` and `ret` can interfere with the disassembler's normal operation. Below is a concrete example that readers can compile and reproduce themselves:

```c++
#include <stdio.h>
// Compile with gcc/g++
int main(){
	__asm__(".byte 0x55;");          // push rbp   save the stack 
	__asm__(".byte 0xe8,0,0,0,0;");  // call $5;	
	__asm__(".byte 0x5d;");	         // pop rbp -> get the value of rip 
	__asm__(".byte 0x48,0x83,0xc5,0x08;"); // add rbp, 8
	__asm__(".byte 0x55;");          // push rbp -> equivalent to modifying call's return value to jump below
	__asm__("ret;");
	__asm__(".byte 0xe8;");          // This is an obfuscation instruction that is not executed
	__asm__(".byte 0x5d;");          // pop rbp restore the stack		
	printf("whoami \n");
	return 0;
} 
```



## Example

Here we use the second challenge from the `Kanxue.TSRC 2017 CTF Autumn Competition` as an example. Download link: [ctf2017_Fpc.exe](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/anti-debug/2017_pediy/ctf2017_Fpc.exe)

The program contains several functions designed as decoys to mislead analysis, and the critical verification logic is protected with junk code to prevent IDA's static analysis. Let's open the Fpc challenge in IDA. The program first prints some prompt information, then gets the user's input.

![main.png](./figure/2017_pediy/main.png)

Here the unsafe `scanf` function is used, and the user input buffer is only `0xCh` bytes long. Let's double-click `v1` to enter the stack frame view:

![stack.png](./figure/2017_pediy/stack.png)

Therefore, we can overflow the data to overwrite the return address, thereby redirecting execution to an arbitrary address.

I should also explain that the several decoy functions before `scanf` are simple equations that are actually unsolvable. The program obfuscates the real verification logic with junk code, preventing IDA from decompiling it properly. So our approach for this challenge is to use the overflow to jump to the actual verification code and continue execution.

During analysis, we can find the following data block not far from the code:

![block.png](./figure/2017_pediy/block.png)

Since IDA failed to properly identify the data, we can move the cursor to the beginning of the data block and press the `C` key (code) to disassemble this data block into code:

![real_code.png](./figure/2017_pediy/real_code.png)

It is worth noting that this code is located at address `0x00413131`. `0x41` is the ASCII code for `'A'`, and `0x31` is the ASCII code for `'1'`. Due to the Kanxue competition restrictions, user input can only contain letters and digits, so we can indeed exploit the overflow vulnerability to execute this code.

Open with OllyDbg, then press `Ctrl+G` to navigate to `0x413131` and set a breakpoint. After running and entering `12345612345611A` followed by Enter, the program successfully reaches `0x00413131`. Then `right-click Analysis -> Remove analysis from module` to properly recognize the code.

![entry.png](./figure/2017_pediy/entry.png)

After breaking at `0x413131`, click the `"View"` menu, select `"Run Trace"`, then click `"Debug"`, select `"Trace Into"`. The program will record the execution flow of the junk code, as shown below:

![trace.png](./figure/2017_pediy/trace.png)

The junk code section is originally very long, but after using OllyDbg's trace feature, the execution flow of the junk code becomes very clear. A large number of jumps occur throughout the process — we only need to extract the effective instructions for analysis.

It should be noted that among the effective instructions, we still need to satisfy certain conditional jumps so that the program continues executing along the correct logic path.

For example, at `0x413420` there is `jnz ctf2017_.00413B03`. We need to start over and set a breakpoint at `0x413420`.

![jnz.png](./figure/2017_pediy/jnz.png)

Modify the flags register to satisfy the jump condition. Continue tracing into (there is also `0041362E  jnz ctf2017_.00413B03` that needs to be satisfied). After ensuring the logic is correct, extract the effective instructions and continue the analysis.

![register.png](./figure/2017_pediy/register.png)
