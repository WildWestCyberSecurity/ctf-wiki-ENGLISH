# Introduction

First, let's provide a brief introduction to the principles of format string vulnerabilities.

## Introduction to Format String Functions

Format string functions accept a variable number of arguments and use **the first argument as the format string to parse the subsequent arguments**. In simple terms, format string functions convert data represented in computer memory into human-readable string format. Almost all C/C++ programs use format string functions to **output information, debug programs, or process strings**. Generally speaking, format strings in exploitation are mainly divided into three parts:

- Format string function
- Format string
- Subsequent arguments, **optional**

Here we give a simple example. In fact, most people have already encountered functions like printf. We will introduce them one by one afterwards.

![](./figure/printf.png)

### Format String Functions

Common format string functions include:

-   Input
    -   scanf
-   Output

|           Function            |        Basic Description         |
| :---------------------: | :-----------------: |
|         printf          |      Output to stdout      |
|         fprintf         |     Output to specified FILE stream      |
|         vprintf         | Format output to stdout based on argument list |
|        vfprintf         | Format output to specified FILE stream based on argument list |
|         sprintf         |       Output to string        |
|        snprintf         |     Output specified number of bytes to string     |
|        vsprintf         |   Format output to string based on argument list   |
|        vsnprintf        | Format output specified bytes to string based on argument list |
|      setproctitle       |       Set argv        |
|         syslog          |        Output log         |
| err, verr, warn, vwarn, etc. |         ...         |

### Format String

Here we'll learn about the format of format strings. The basic format is as follows:

```
%[parameter][flags][field width][.precision][length]type
```
For the meaning of each pattern, please refer to the Wikipedia article on [Format string](https://zh.wikipedia.org/wiki/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2). The following options within the patterns need special attention:

-   parameter
    -   n$, get the specified parameter in the format string
-   flag
-   field width
    -   Minimum width of the output
-   precision
    -   Maximum length of the output
-   length, length of the output
    -   hh, output one byte
    -   h, output a two-byte value
-   type
    -   d/i, signed integer
    -   u, unsigned integer
    -   x/X, hexadecimal unsigned int. x uses lowercase letters; X uses uppercase letters. If precision is specified, the output is left-padded with zeros when the number of digits is insufficient. Default precision is 1. If precision is 0 and the value is 0, the output is empty.
    -   o, octal unsigned int. If precision is specified, the output is left-padded with zeros when the number of digits is insufficient. Default precision is 1. If precision is 0 and the value is 0, the output is empty.
    -   s, if the l flag is not used, outputs a null-terminated string up to the limit specified by precision; if no precision is specified, all bytes are output. If the l flag is used, the corresponding function argument points to a wchar\_t array, and each wide character is converted to a multibyte character during output, equivalent to calling the wcrtomb function.
    -   c, if the l flag is not used, the int argument is converted to unsigned char for output; if the l flag is used, the wint\_t argument is converted to a wchart_t array containing two elements, where the first element contains the character to output and the second element is a null wide character.
    -   p, void \* type, outputs the value of the corresponding variable. printf("%p",a) prints the value of variable a in address format, printf("%p", &a) prints the address where variable a is located.
    -   n, does not output characters, but writes the number of characters successfully output so far into the variable pointed to by the corresponding integer pointer argument.
    -   %, '``%``' literal value, does not accept any flags or width.

### Parameters

These are the corresponding variables to be output.

## Format String Vulnerability Principles

At the beginning, we provided a basic introduction to format strings. Here we'll discuss some more detailed content. As mentioned above, format string functions parse based on the format string. **Therefore, the number of arguments to be parsed is naturally controlled by this format string**. For example, '%s' indicates that we will output a string argument.

Let's continue with the example above:

![Basic Example](./figure/printf.png)

For such an example, before entering the printf function (i.e., before printf is called), the stack layout from high address to low address is as follows:

```text
some value
3.14
123456
addr of "red"
addr of format string: Color %s...
```

**Note: Here we assume the value above 3.14 is some unknown value.**

After entering printf, the function first obtains the first argument and reads its characters one by one, encountering two situations:

-   The current character is not %, output it directly to standard output.
-   The current character is %, continue reading the next character
    -   If there is no character, report an error
    -   If the next character is %, output %
    -   Otherwise, obtain the corresponding argument based on the character, parse it, and output it

Now suppose that when writing the program, we wrote it like this:

```C
printf("Color %s, Number %d, Float %4.2f");
```

At this point we can see that we did not provide arguments, so how will the program run? The program will still run, and it will parse the three variables above the format string address on the stack as:

1. Parse the string corresponding to that address
2. Parse the content as an integer value
3. Parse the content as a floating-point value

For cases 2 and 3, it doesn't matter much, but for case 1, if an inaccessible address is provided, such as 0, the program will crash because of this.

This is basically the fundamental principle of format string vulnerabilities.

## Recommended Reading

- https://zh.wikipedia.org/wiki/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2
