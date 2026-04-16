# Detecting Breakpoints

gdb implements breakpoints by replacing the byte at the target address with `0xcc`. Here is a simple example of detecting `int 3` breakpoints:

``` c
void foo() {
    printf("Hello\n");
}
int main() {
    if ((*(volatile unsigned *)((unsigned)foo) & 0xff) == 0xcc) {
        printf("BREAKPOINT\n");
        exit(1);
    }
    foo();
}
```

Running the program normally will output Hello, but if a `cc` breakpoint was previously set at the `foo` function and the program is run, gdb will not be able to break there and will output `BREAKPOINT`.

```
# gdb ./x
gdb> bp foo
Breakpoint 1 at 0x804838c
gdb> run
BREAKPOINT
Program exited with code 01.
```

Bypassing this is quite simple: you need to read the assembly code and be careful not to set breakpoints at the `foo` function entry point. In real situations, it depends on where the breakpoint detection is located.

For this type of breakpoint-monitoring anti-debugging technique, the key is not how to bypass it, but how to detect it. In this example, it's easy to discover since the program prints corresponding messages. In real situations, the program won't output any information, and breakpoints won't easily trigger. We can use a `perl` script to filter out code related to `0xcc` from the disassembly for inspection.

We can use a perl script to filter out code related to 0xcc from the disassembly for inspection.


``` perl
#!/usr/bin/perl
while(<>)
{
    if($_ =~ m/([0-9a-f][4]:\s*[0-9a-f \t]*.*0xcc)/ ){ print; }
}
```

Display results:

```
# objdump -M intel -d xxx | ./antibp.pl
      80483be: 3d cc 00 00 00 cmp eax,0xcc
```

Once detected, you can modify 0xcc to 0x00 or 0x90, or perform any other operation you want.

Changing 0xcc can also cause problems. As introduced in the previous section, if the program performs file integrity checks, our changes will be detected. In some cases, the program may not only check the function entry point but also check the entire function in a loop.

Therefore, you can also use a hex editor to manually place an `ICEBP(0xF1)` byte at the location where you need to break (instead of `int 3`), because `ICEBP` can also cause gdb to break.



> Reference: [Beginners Guide to Basic Linux Anti Anti Debugging Techniques](http://www.stonedcoder.org/~kd/lib/14-61-1-PB.pdf)
