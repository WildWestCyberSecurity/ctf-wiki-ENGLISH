# Stack Overflow Principle 

## Introduction 

For an introduction to the stack, you can read the introduction in [Linux Pwn](https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stack-intro/).

## Basic Example 

Below is a typical example. In this example, a last-byte overflow exists due to the order of variable declarations and the size of the buffer declaration.

```c
#include <stdio.h>
#include <string.h>
#define PASSWORD "666666"
int verify_password(char *password)
{
	int authenticated;
	char buffer[8];
	authenticated = strcmp(password,PASSWORD);
	strcpy(buffer,password); 
	return authenticated;
}
void main()
{
	int valid_flag =0;
	char password[128];
	while(1)
	{
		printf("please input password:  ");
		scanf("%s",password);
		valid_flag = verify_password(password);
		if (valid_flag !=0)
		{
			printf("incorrect password!\n");
		}
		else
		{
			printf("Congratulation! You have passed the verification!\n");
			break;
		}
	}
}
```

This is a simple password verification program that checks whether the input string is equal to 666666. Use vc6.0 to compile this program, and after a successful compilation, use winchecksec to check the enabled protections. We can see that GS is enabled, but this does not prevent our overflow.

```
C:\Users\CarlStar\Desktop>winchecksec.exe demo1.exe
Dynamic Base    : false
ASLR            : true
High Entropy VA : false
Force Integrity : false
Isolation       : true
NX              : true
SEH             : true
CFG             : false
RFG             : false
SafeSEH         : true
GS              : true
Authenticode    : false
.NET            : true
```

Use OllyDbg to dynamically debug this program. Enter aaaaaa to observe the normal execution flow of the program. To facilitate understanding of the entire process, set a breakpoint after the **strcmp** function and after **strcpy** finishes executing.

![demo1](./figure/demo1-1.png)

Now we can let the program run. After entering aaaaaa, the program will hit our first breakpoint. Enter the **strcmp** function and observe its return value. Since the ASCII value of 'a' is greater than the ASCII value of '6', the function should return **1** as expected. In x86, the return value is stored in the EAX register. After the function returns normally, since the program still needs to use these registers to complete its remaining functionality, this return value will be saved on the stack at **ss:[0012FEA0]**.

![demo2](./figure/demo1-2.png)

![demo3](./figure/demo1-3.png)

When execution reaches the second breakpoint, let's look at the stack structure. Here, 61 is the ASCII representation of the 'a' we entered, and **00** is the string terminator. The **buffer** is 8 bytes in size, so if we enter 8 a's, the final string terminator will overflow into **0012FEA0**, overwriting the original value to 0. This allows us to change the program's execution flow and output "Congratulation! You have passed the verification!"

```
0012FE90   CCCCCCCC
0012FE94   CCCCCCCC
0012FE98   61616161
0012FE9C   CC006161
0012FEA0   00000001
```

OK, let's first let the program continue running normally.

![demo4](./figure/demo1-4.png)

This time we enter 8 a's to verify whether it works as we expected: **the string terminator will overflow into the return value of strcmp**. We can see that the return value of strcmp is still 1.

![demo5](./figure/demo1-5.png)

Continue running to the second breakpoint and check the current stack values. **The return value of strcmp has been successfully overflowed from 1 to 0**.

```
0012FE90   CCCCCCCC
0012FE94   CCCCCCCC
0012FE98   61616161
0012FE9C   61616161
0012FEA0   00000000
```

Now let the program continue running. It successfully outputs the expected string.

![demo6](./figure/demo1-6.png)



## Recommended Reading 

[stack buffer overflow](https://en.wikipedia.org/wiki/Stack_buffer_overflow)

[0day Security: Software Vulnerability Analysis Techniques]()

[Winchecksec](https://github.com/trailofbits/winchecksec)
