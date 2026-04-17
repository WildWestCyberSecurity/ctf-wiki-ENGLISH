# ARX: Add-Rotate-Xor

## Overview

ARX operations are a collective term for the following 3 basic operations:
- Add — modular addition over a finite field
- Rotate — circular shift (rotation)
- Xor — exclusive or

Many common block cipher algorithms use only these 3 basic operations in their round functions. Typical examples include Salsa20, Speck, etc. Additionally, [IDEA](./idea.md) also employs similar basic operations to construct its encryption and decryption operations, but replaces rotation with multiplication.

## Advantages and Disadvantages

### Advantages

- Simple operations, fast computation
- Constant execution time, which can avoid timing-based side-channel attacks
- The combined function has sufficient expressive power (see the example problem below)

### Disadvantages

- Among the three basic operations, Rotate and Xor are both fully linear operations for individual bits, which may introduce certain vulnerabilities (see [Rotational cryptanalysis](https://en.wikipedia.org/wiki/Rotational_cryptanalysis))

## Challenges

### 2018 *ctf primitive

#### Analysis

This challenge requires us to combine a limited number of Add-Rotate-Xor operations so that the resulting encryption algorithm can encrypt a fixed plaintext into a specified random ciphertext — in other words, constructing an arbitrary permutation function from basic operations. After successfully constructing it 3 times, we obtain the flag.

#### Solution Approach

For operations modulo 256, a typical ARX-based swap operation can be expressed as the following combination:
```
RotateLeft_1(Add_255(RotateLeft_7(Add_2(x))))
```

The above function corresponds to a permutation that swaps 254 and 255 while leaving all other numbers unchanged.

Intuitively, since only inputs 254 and 255 cause a carry in the first modular addition by 2, this composite function is able to treat this case differently.

Using the above atomic operation, we can construct a permutation of any two numbers `a,b`. Combined with the Xor operation, we can reduce the number of required basic operations to satisfy the constraint given by the challenge. One possible sequence of operations is as follows:

1. For `a,b`, use modular addition to make `a` equal to 0
2. Use right shift to make the least significant bit of b equal to 1
3. If `b` is not 1, perform `Xor 1, Add 255` operations, keeping `a` still at 0 while reducing the value of `b`
4. Repeat steps 2-3 until `b` is 1
5. Perform `Add 254` and the swap operation to exchange `a,b`
6. For all operations other than the swap, add their corresponding inverse operations to ensure that values other than `a,b` remain unchanged

The complete solution script is as follows:

```python
from pwn import *
import string
from hashlib import sha256

#context.log_level='debug'
def dopow():
    chal = c.recvline()
    post = chal[12:28]
    tar = chal[33:-1]
    c.recvuntil(':')
    found = iters.bruteforce(lambda x:sha256(x+post).hexdigest()==tar, string.ascii_letters+string.digits, 4)
    c.sendline(found)

#c = remote('127.0.0.1',10001)
c = remote('47.75.4.252',10001)
dopow()
pt='GoodCipher'

def doswap(a,b):
    if a==b:
        return
    if a>b:
        tmp=b
        b=a
        a=tmp
    ans=[]
    ans.append((0,256-a))
    b-=a
    a=0
    while b!=1:
        tmp=0
        lo=1
        while b&lo==0:
            lo<<=1
            tmp+=1
        if b==lo:
            ans.append((1,8-tmp))
            break
        if tmp!=0:
            ans.append((1,8-tmp))
        b>>=tmp
        ans.append((2,1))
        b^=1
        ans.append((0,255))
        b-=1
    ans.append((0,254))

    for a,b in ans:
        c.sendline('%d %d'%(a,b))
        c.recvline()
    for a,b in [(0,2),(1,7),(0,255),(1,1)]:
        c.sendline('%d %d'%(a,b))
        c.recvline()
    for a,b in ans[::-1]:
        if a==0:
            c.sendline('%d %d'%(a,256-b))
        elif a==1:
            c.sendline('%d %d'%(a,8-b))
        elif a==2:
            c.sendline('%d %d'%(a,b))
        c.recvline()

for i in range(3):
    print i
    m=range(256)
    c.recvuntil('ciphertext is ')
    ct=c.recvline().strip()
    ct=ct.decode('hex')
    assert len(ct)==10
    for i in range(10):
        a=ord(ct[i])
        b=ord(pt[i])
        #print m[a],b
        doswap(m[a],b)
        for j in range(256):
            if m[j]==b:
                m[j]=m[a]
                m[a]=b
                break
    c.sendline('-1')

c.recvuntil('Your flag here.\n')
print c.recvline()
```
