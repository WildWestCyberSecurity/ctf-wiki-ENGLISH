# SHA1

## Basic Description

The input and output of SHA1 are as follows:

- Input: A message of arbitrary length, divided into **512-bit** blocks. First, a bit 1 is appended to the right of the message, then several 0 bits are appended until the bit length of the message satisfies the condition of being congruent to 448 modulo 512.
- Output: A 160-bit message digest.

For a detailed introduction, please search on your own.

Generally, we can determine whether a function is a SHA1 function by its initialization. If a function has the following five initialization variables, we can guess that it is a SHA1 function, as these are the initialization IVs of the SHA1 function.

```
0x67452301
0xEFCDAB89
0x98BADCFE
0x10325476
0xC3D2E1F0
```

The first four are similar to MD5, and the last one is newly added.

## Cracking

As of now, SHA1 is no longer secure, because Google previously published two PDFs with the same SHA1 value. For details, please refer to [shattered](https://shattered.io/).

There is also an interesting website here: https://alf.nu/SHA1.

## 2017 SECCON SHA1 is dead

The challenge description is as follows:

1. file1 != file2
2. SHA1(file1) == SHA1(file2)
3. SHA256(file1) <> SHA256(file2)
4. 2017KiB < sizeof(file1) < 2018KiB
5. 2017KiB < sizeof(file2) < 2018KiB

Where 1KiB = 1024 bytes

We need to find two files satisfying the above constraints.

This immediately brings to mind the documents Google previously published. Moreover, very importantly, as long as the given first 320 bytes are used, appending the same bytes afterwards will still produce the same hash. Here we test as follows:

```shell
➜  2017_seccon_sha1_is_dead git:(master) dd bs=1 count=320 <shattered-1.pdf| sha1sum
320+0 records in
320+0 records out
320 bytes copied, 0.00796817 s, 40.2 kB/s
f92d74e3874587aaf443d1db961d4e26dde13e9c  -
➜  2017_seccon_sha1_is_dead git:(master) dd bs=1 count=320 <shattered-2.pdf| sha1sum
320+0 records in
320+0 records out
320 bytes copied, 0.00397215 s, 80.6 kB/s
f92d74e3874587aaf443d1db961d4e26dde13e9c  -
```

Then we can directly write the program as follows:

```python
from hashlib import sha1
from hashlib import sha256

pdf1 = open('./shattered-1.pdf').read(320)
pdf2 = open('./shattered-2.pdf').read(320)
pdf1 = pdf1.ljust(2017 * 1024 + 1 - 320, "\00")  #padding pdf to 2017Kib + 1
pdf2 = pdf2.ljust(2017 * 1024 + 1 - 320, "\00")
open("upload1", "w").write(pdf1)
open("upload2", "w").write(pdf2)

print sha1(pdf1).hexdigest()
print sha1(pdf2).hexdigest()
print sha256(pdf1).hexdigest()
print sha256(pdf2).hexdigest()
```

## References

- https://www.slideshare.net/herumi/googlesha1
