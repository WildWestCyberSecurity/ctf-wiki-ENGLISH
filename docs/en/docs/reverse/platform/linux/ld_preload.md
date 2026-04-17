# LD_PRELOAD

## Principle

Under normal circumstances, the Linux dynamic loader `ld-linux` (see man page ld-linux(8)) searches for and loads the shared library files needed by a program. `LD_PRELOAD` is an optional environment variable that contains one or more paths to shared library files. The loader will load the shared libraries specified by `LD_PRELOAD` before the C runtime library, which is known as preloading.

Preloading means that its functions will be called before functions with the same name in other library files, which allows library functions to be intercepted or replaced. Multiple shared library file paths can be separated by `colons` or `spaces`. Obviously, the only programs not affected by `LD_PRELOAD` are those that are statically linked.

Of course, to avoid being used for malicious attacks, the loader will not use `LD_PRELOAD` for preloading when `ruid != euid`.

More reading: [https://blog.fpmurphy.com/2012/09/all-about-ld_preload.html#ixzz569cbyze4](https://blog.fpmurphy.com/2012/09/all-about-ld_preload.html#ixzz569cbyze4)

## Example

Below we use the 2014 `Hack In The Box Amsterdam: Bin 100` challenge as an example. Challenge download link: [hitb_bin100.elf](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/linux-re/2014_hitb/hitb_bin100.elf)

This is a 64-bit ELF file. The execution result is shown in the image below:

![run.png](./figure/2014_hitb/run.png)

The program seems to be continuously printing some sentences with no sign of stopping. Let's open it with IDA. First press `Shift+F12` to search for strings.

![ida_strings.png](./figure/2014_hitb/ida_strings.png)

Obviously, besides the sentences being continuously printed, we found some interesting strings:

```
.rodata:0000000000400A53 00000006 C KEY:
.rodata:0000000000400A5F 0000001F C OK YOU WIN. HERE'S YOUR FLAG:
```

We follow the cross-reference of `OK YOU WIN. HERE'S YOUR FLAG: ` to reach the key code (I've removed some unnecessary code).

```  c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  qmemcpy(v23, &unk_400A7E, sizeof(v23));
  v3 = v22;
  for ( i = 9LL; i; --i )
  {
    *(_DWORD *)v3 = 0;
    v3 += 4;
  }
  v20 = 0x31337;
  v21 = time(0LL);
  do
  {
    v11 = 0LL;
    do
    {
      v5 = 0LL;
      v6 = time(0LL);
      srand(233811181 - v21 + v6); // initialize random seed
      v7 = v22[v11];
      v22[v11] = rand() ^ v7;   // pseudo-random number
      v8 = (&funny)[8 * v11];
      while ( v5 < strlen(v8) )
      {
        v9 = v8[v5];
        if ( (_BYTE)v9 == 105 )
        {
          v24[(signed int)v5] = 105;
        }
        else
        {
          if ( (_DWORD)v5 && v8[v5 - 1] != 32 )
            v10 = __ctype_toupper_loc();    // uppercase
          else
            v10 = __ctype_tolower_loc();    // lowercase
          v24[(signed int)v5] = (*v10)[v9];
        }
        ++v5;
      }
      v24[(signed int)v5] = 0;
      ++v11;
      __printf_chk(1LL, " 鈾%80s 鈾玕n", v24); // the garbled text is actually a musical note
      sleep(1u);
    }
    while ( v11 != 36 );
    --v20;
  }
  while ( v20 );
  v13 = v22;    // key is stored in the v22 array
  __printf_chk(1LL, "KEY: ", v12);
  do
  {
    v14 = (unsigned __int8)*v13++;
    __printf_chk(1LL, "%02x ", v14); // output key
  }
  while ( v13 != v23 );
  v15 = 0LL;
  putchar(10);
  __printf_chk(1LL, "OK YOU WIN. HERE'S YOUR FLAG: ", v16);
  do
  {
    v17 = v23[v15] ^ v22[v15];  // XOR with key values
    ++v15;
    putchar(v17);   // output flag
  }
  while ( v15 != 36 );
  putchar(10);      // output newline
  result = 0;
  return result;
}
```

The entire code flow mainly consists of continuously looping and outputting sentences from `funny`. After meeting the loop condition, it outputs the `key` and XORs with the `key` to get the `flag` value.

However, we can see that the total number of loop iterations is relatively small. So we can use some methods to make the loop run faster. For example, I can manually patch the program to prevent it from outputting strings (in fact, `printf` takes a considerable amount of time), and secondly, use `LD_PRELOAD` to disable the program's `sleep()`. This can noticeably save time.

The manual patching process is relatively simple. We can find the code location and modify it with a hex editor. Of course, we can also use `IDA` for the patching work.

``` asm
.text:00000000004007B7                 call    ___printf_chk
.text:00000000004007BC                 xor     eax, eax
```

Click the cursor on `call    ___printf_chk`, then select menu `Edit->Patch Program->Assemble` (of course you can use other patching methods — the effect is the same). Then change it to `nop(0x90)`, as shown below:

![ida_patch.png](./figure/2014_hitb/ida_patch.png)

Change all assembly code between `4007B7` and `4007BD` to `nop`. Then select menu `Edit->Patch Program->Apply patches to input file`. It's best to make a backup (check `Create a backup`), then click OK (I renamed it to `patched.elf`, download link: [patched.elf](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/linux-re/2014_hitb/patched.elf)).

![ida_apply.png](./figure/2014_hitb/ida_apply.png)

Now for the `LD_PRELOAD` part. Here we simply write some C code, download link: [time.c](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/linux-re/2014_hitb/time.c)

``` c
static int t = 0x31337;

void sleep(int sec) {
	t += sec;
}

int time() {
	return t;
}
```

Then use the command `gcc --shared time.c -o time.so` to generate the shared library file. A download link is also provided: [time.so](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/linux-re/2014_hitb/time.so)

Then open a Linux terminal and run the command: `LD_PRELOAD=./time.so ./patched.elf`

![LD_PRELOAD.png](./figure/2014_hitb/ld_preload.png)

After a while, you'll hear the CPU spinning frantically, and then the flag will appear quickly.
