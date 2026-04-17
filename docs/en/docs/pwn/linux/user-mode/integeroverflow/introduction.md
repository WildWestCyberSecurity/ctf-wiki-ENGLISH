# Integer Overflow

## Introduction

In C, the basic integer data types are divided into short int, int, and long int. These three data types are further divided into signed and unsigned variants, each with its own size and range (since the size ranges of data types are determined by the compiler, everything described below assumes 64-bit with gcc-5.4 by default), as shown below:


| Type | Size | Range |
| :-: | :-: | :-: |
| short int | 2byte(word) | 0\~32767(0\~0x7fff) <br> -32768\~-1(0x8000\~0xffff)  |
| unsigned short int | 2byte(word) | 0\~65535(0\~0xffff) |
| int | 4byte(dword) | 0\~2147483647(0\~0x7fffffff) <br> -2147483648\~-1(0x80000000\~0xffffffff) |
| unsigned int | 4byte(dword) | 0\~4294967295(0\~0xffffffff) |
| long int | 8byte(qword) | Positive: 0\~0x7fffffffffffffff <br> Negative: 0x8000000000000000\~0xffffffffffffffff |
| unsigned long int | 8byte(qword) | 0\~0xffffffffffffffff |

When data in a program exceeds the range of its data type, an overflow occurs. An overflow of an integer type is called an integer overflow.

## Principle

Next, let's briefly explain the principle of integer overflow.

### Upper Bound Overflow

```
# Pseudocode
short int a;

a = a + 1;
# Corresponding assembly
movzx  eax, word ptr [rbp - 0x1c]
add    eax, 1
mov    word ptr [rbp - 0x1c], ax

unsigned short int b;

b = b + 1;
# assembly code
add    word ptr [rbp - 0x1a], 1
```

There are two cases of upper bound overflow: one is `0x7fff + 1`, and the other is `0xffff + 1`.

Because the underlying instructions of the computer do not distinguish between signed and unsigned — data exists only in binary form (the compiler level is where signed and unsigned are distinguished, producing different assembly instructions).

So `add 0x7fff, 1 == 0x8000`. This type of upper bound overflow has no effect on unsigned integers, but in signed short integers, `0x7fff` represents `32767` while `0x8000` represents `-32768`. Expressed mathematically, in signed short integers `32767+1 == -32768`.

The second case is `add 0xffff, 1`. In this case, we need to consider the first operand.

For example, the assembly code for signed addition above is `add eax, 1`. Since `eax=0xffff`, `add eax, 1 == 0x10000`. However, the unsigned assembly code performs addition on memory: `add word ptr [rbp - 0x1a], 1 == 0x0000`.

In signed addition, although the result of `eax` is 0x10000, only the value `ax=0x0000` is stored to memory, so the result looks the same as unsigned.

Let's also look at the result of this overflow from a numeric perspective. In signed short integers, `0xffff==-1, -1 + 1 == 0` — from the signed perspective, this calculation is correct.

But in unsigned short integers, `0xffff == 65535, 65535 + 1 == 0`.

### Lower Bound Overflow

The principle of lower bound overflow is the same as upper bound overflow; in assembly code, `add` is simply replaced with `sub`.

There are also two cases:

The first is `sub 0x0000, 1 == 0xffff`. For signed values, `0 - 1 == -1` is correct, but for unsigned values it becomes `0 - 1 == 65535`.

The second is `sub 0x8000, 1 == 0x7fff`. For unsigned values, `32768 - 1 == 32767` is correct, but for signed values it becomes `-32768 - 1 = 32767`.

## Examples

Among the integer overflow vulnerabilities I have encountered, I believe they can be summarized into two categories.

### Unrestricted Range

This case is easy to understand. Imagine a bucket of fixed size — if you pour water into it without limiting how much, the water will overflow from the bucket.

If something has a fixed size and you don't constrain it, unpredictable consequences will occur.

Here is a simple example:

```c
$ cat test.c
#include<stddef.h>
int main(void)
{
    int len;
    int data_len;
    int header_len;
    char *buf;
    
    header_len = 0x10;
    scanf("%uld", &data_len);
    
    len = data_len+header_len
    buf = malloc(len);
    read(0, buf, data_len);
    return 0;
}
$ gcc test.c
$ ./a.out
-1
asdfasfasdfasdfafasfasfasdfasdf
# gdb a.out
► 0x40066d <main+71>    call   malloc@plt <0x400500>
        size: 0xf
```

Only `0x20` bytes of heap are allocated, but `0xffffffff` bytes of data can be input — from integer overflow to heap overflow.

### Incorrect Type Conversion

Even if variables are correctly constrained, integer overflow vulnerabilities can still occur. I believe these can be generalized as incorrect type conversions. If we further subdivide, they can be categorized as:

1. Assigning a larger-range variable to a smaller-range variable

```c
$ cat test2.c
void check(int n)
{
    if (!n)
        printf("vuln");
    else
        printf("OK");
}

int main(void)
{
    long int a;
    
    scanf("%ld", &a);
    if (a == 0)
        printf("Bad");
    else
        check(a);
    return 0;
}
$ gcc test2.c
$ ./a.out
4294967296
vuln
```

The above code is an example where a larger-range variable (long int a) is passed into the check function and becomes a smaller-range variable (int n), causing an integer overflow.

A long int occupies 8 bytes of memory space, while an int only has 4 bytes. So when long -> int, truncation occurs — only the lower 4 bytes of the long int value are passed to the int variable.

In the above example, `long: 0x100000000 -> int: 0x00000000`.

However, a smaller-range variable can fully pass its value to a larger-range variable without data loss.

2. Only performing single-sided restriction

This case only applies to signed types.

```c
$ cat test3.c
int main(void)
{
    int len, l;
    char buf[11];

    scanf("%d", &len);
    if (len < 10) {
        l = read(0, buf, len);
        *(buf+l) = 0;
        puts(buf);
    } else
        printf("Please len < 10");        
}
$ gcc test3.c
$ ./a.out
-1
aaaaaaaaaaaa
aaaaaaaaaaaa
```

On the surface, we have restricted the variable len. But upon careful consideration, len is a signed integer, so len can be negative. However, in the read function, the third parameter's type is `size_t`, which is equivalent to `unsigned long int`, an unsigned long integer.

Both of the above examples share a common characteristic: the formal parameter and actual parameter types of the functions are different. Therefore, I believe these can be summarized as incorrect type conversions.

## CTF Example

Challenge: [Pwnhub The Beginning of the Story calc](http://atum.li/2016/12/05/calc/)
