# Executing Shellcode

## Introduction

Shellcode is a piece of code used to exploit software vulnerabilities. Shellcode consists of hexadecimal machine code, and it gets its name from the fact that it often provides attackers with a shell. Shellcode is usually written in machine language. After the EIP register overflows, a piece of shellcode machine code can be injected for the CPU to execute, allowing the computer to execute arbitrary instructions from the attacker. When compiling with ASLR, NX, and CANARY options disabled, shellcode can be placed on the stack during input. Through dynamic debugging, the required padding can be determined to overflow the return address to the shellcode's address on the stack, so that the program will execute the shellcode upon returning.

![demo](./figure/demo2-1.png)

### Example

Below is a classic example that demonstrates the execution of **shellcode** after a program overflow. The compilation environment is WinXP with the VC6.0 tool.

```c
#include <stdio.h>
#include <windows.h>

#define PASSWORD "1234567"

int verify_password(char *password)
{
	int authenticated;
	char buffer[50];
	authenticated = strcmp(password,PASSWORD);
	memcpy(buffer,password,strlen(password)); 
	return authenticated;
}

void main()
{
	int valid_flag =0;
	char password[1024];
	FILE *fp;

	LoadLibrary("user32.dll");

	if (!(fp=fopen("password.txt","rw+")))
	{
		exit(0);
	}
	fscanf(fp,"%s",password);

	valid_flag = verify_password(password);

	if (valid_flag !=0)
	{
		printf("incorrect password!\n\n");
	}
	else
	{
		printf("Congratulation! You have passed the verification!\n");
	}
	fclose(fp);
	getchar();
}
```



After compilation, drag it into OllyDbg for dynamic debugging to determine the length of the **padding**. Set a breakpoint at **memcpy** for convenient debugging. You can first generate 50 BYTES of padding to compare the distance to the return address, and ultimately determine that the return address starts after 60 BYTES.

![demo](./figure/demo2-2.png)



The input string will be copied to the position **0012FAE4** on the stack.

![demo](./figure/demo2-3.png)



Because through proper padding we controlled the return address to **0012FAE4**, when the function returns, the value of register **EIP** will be **0012FAE4**. At this point, the system will treat the data on the stack as machine code, and the program will execute the code at address **0012FAE4**.

![demo](./figure/demo2-4.png)



The content in **password.txt** consists of carefully crafted machine code, whose function is to pop up a message box with the content **hackhack**. How to write the content of **password.txt** will be covered in later chapters; this chapter focuses on introducing the entire execution flow.

As expected, the program executed the pop-up function after returning.

![demo](./figure/demo2-5.png)



### References

[0day Security: Software Vulnerability Analysis Techniques]()

[cve-2015-8277](https://www.securifera.com/advisories/cve-2015-8277/)
