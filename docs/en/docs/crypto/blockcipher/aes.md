# AES

## Basic Introduction

Advanced Encryption Standard (AES) is a typical block cipher designed to replace DES. It was designed by Joan Daemen and Vincent Rijmen. Its basic information is as follows:

- Input: 128 bits.
- Output: 128 bits.
- SPN (Substitution-Permutation Network) structure.

The number of iteration rounds depends on the key length, as shown below:

| Key Length (bits) | Number of Rounds |
| :--------------: | :------: |
|       128        |    10    |
|       192        |    12    |
|       256        |    14    |

## Basic Process

### Basic Concepts

In the AES encryption and decryption process, each block is 128 bits. Let us clarify some basic concepts here.

![](./figure/aes_data_unit.png)

In AES, the conversion process between a block and a State is as follows:

![](./figure/aes_block2state.png)

As we can see, the bytes in each block are arranged by column into the state array.

For plaintext, we generally choose to encode it in hexadecimal.

![](./figure/aes_plain2state.png)



### Encryption and Decryption Process

Here is a nice [illustration](http://bbs.pediy.com/thread-90722.htm) from Kanxue to help introduce the basic process. Each round mainly includes:

- AddRoundKey
- SubBytes
- ShiftRows
- MixColumns

![](./figure/aes_details.jpg)

Note: In the MixColumns matrix multiplication above, the column vector on the left side of the equals sign should be on the right side.

Here is another complete diagram of encryption and decryption. The correctness of the decryption algorithm is obvious.

![](./figure/aes_enc_dec.png)

Let us focus on the following aspects.

#### SubBytes

Behind the byte substitution, there are corresponding mathematical rules that define the substitution table, as follows:

![](./figure/aes_subbytes.png)

The operations here are all defined within $GF(2^8)$.

#### MixColumns

The operations here are also defined over $GF(2^8)$, using the reduction polynomial $x^8+x^4+x^3+1$.

#### Key Expansion

![](./figure/aes_key_expansion.png)

## Equivalent Decryption Algorithm

Through simple analysis, we can find that:

- Swapping Inverse ShiftRows and Inverse SubBytes does not affect the result.
- Swapping AddRoundKey and Inverse MixColumns does not affect the result. The key insight is:
  - First, XOR can be viewed as polynomial addition over a field.
  - Then, multiplication distributes over addition in a polynomial ring.

## Attack Methods

- Integral attack

## 2018 National Competition Crackmec

Through simple analysis of this algorithm, we can find that it is a simplified version of AES. Its basic operations are:

- 9 rounds of iteration
    - ShiftRows
    - Modified SubBytes

As follows:

```c
  memcpy(cipher, plain, 0x10uLL);
  for ( i = 0LL; i <= 8; ++i )
  {
    shift_row(cipher);
    for ( j = 0LL; j <= 3; ++j )
      *(_DWORD *)&cipher[4 * j] =
        box[((4 * j + 3 + 16 * i) << 8) + (unsigned __int8)cipher[4 * j + 3]] ^
        box[((4 * j + 2 + 16 * i) << 8) + (unsigned __int8)cipher[4 * j + 2]] ^
        box[((4 * j + 1 + 16 * i) << 8) + (unsigned __int8)cipher[4 * j + 1]] ^
        box[((4 * j + 16 * i) << 8) + (unsigned __int8)cipher[4 * j]];
  }
  result = shift_row(cipher);
  for ( k = 0LL; k <= 0xF; ++k )
  {
    result = subbytes[256 * k + (unsigned __int8)cipher[k]];
    cipher[k] = result;
  }
  return result;
```

Based on the program flow, we already know the encryption result, and since subbytes and shift_row are both invertible, we can obtain the result after the last round of encryption. At this point, we also know the constants corresponding to box, but we don't know the value of `cipher[4*j]` from the previous round — a total of 32 bits. If we brute-force directly, it's clearly infeasible because every round would require the same brute-force, and the time would be unacceptable. Is there another way? Indeed, we can consider a meet-in-the-middle attack: first enumerate all byte combinations of `cipher[4*j]` and `cipher[4*j+1]`, a total of 256×256 possibilities. When enumerating the remaining two bytes, we can first compute their XOR with the ciphertext, then look it up in the previous combinations. If found, we consider it correct. This instantly reduces the complexity to $O(2^{16})$.

The code is as follows:

```python
encflag = [
    0x16, 0xEA, 0xCA, 0xCC, 0xDA, 0xC8, 0xDE, 0x1B, 0x16, 0x03, 0xF8, 0x84,
    0x69, 0x23, 0xB2, 0x25
]
subbytebox = eval(open('./subbytes').read())
box = eval(open('./box').read())
print subbytebox[-1], box[-1]


def inv_shift_row(now):
    tmp = now[13]
    now[13] = now[9]
    now[9] = now[5]
    now[5] = now[1]
    now[1] = tmp

    tmp = now[10]
    now[10] = now[2]
    now[2] = tmp
    tmp = now[14]
    now[14] = now[6]
    now[6] = tmp

    tmp = now[15]
    now[15] = now[3]
    now[3] = now[7]
    now[7] = now[11]
    now[11] = tmp

    return now


def byte2num(a):
    num = 0
    for i in range(3, -1, -1):
        num = num * 256
        num += a[i]
    return num


def getbytes(i, j, target):
    """
    box[((4 * j + 3 + 16 * i) << 8) + a2[4 * j + 3]]
    box[((4 * j + 2 + 16 * i) << 8 )+ a2[4 * j + 2]]
    box[((4 * j + 1 + 16 * i) << 8) + a2[4 * j + 1]]
    box[((4 * j + 16 * i) << 8) + a2[4 * j]];
    """
    box01 = dict()
    for c0 in range(256):
        for c1 in range(256):
            num0 = ((4 * j + 16 * i) << 8) + c0
            num1 = ((4 * j + 1 + 16 * i) << 8) + c1
            num = box[num0] ^ box[num1]
            box01[num] = (c0, c1)
    for c2 in range(256):
        for c3 in range(256):
            num2 = ((4 * j + 2 + 16 * i) << 8) + c2
            num3 = ((4 * j + 3 + 16 * i) << 8) + c3
            num = box[num2] ^ box[num3]
            calc = num ^ target
            if calc in box01:
                c0, c1 = box01[calc]
                return c0, c1, c2, c3
    print 'not found'
    print i, j, target, calc
    exit(0)


def solve():
    a2 = [0] * 16
    """
      for ( k = 0LL; k <= 0xF; ++k )
      {
        result = subbytesbox[256 * k + a2[k]];
        a2[k] = result;
      }
    """
    for i in range(15, -1, -1):
        tag = 0
        for j in range(256):
            if subbytebox[256 * i + j] == encflag[i]:
                # j = a2[k]
                tag += 1
                a2[i] = j
                if tag == 2:
                    print 'two number', i
                    exit(0)
    """
      result = shift_row(a2);
    """
    a2 = inv_shift_row(a2)
    """
      for ( i = 0LL; i <= 8; ++i )
      {
        shift_row(a2);
        for ( j = 0LL; j <= 3; ++j )
          *(_DWORD *)&a2[4 * j] = box[((4 * j + 3 + 16 * i) << 8) + a2[4 * j + 3]] ^ box[((4 * j + 2 + 16 * i) << 8)
                                                                                       + a2[4 * j + 2]] ^ box[((4 * j + 1 + 16 * i) << 8) + a2[4 * j + 1]] ^ box[((4 * j + 16 * i) << 8) + a2[4 * j]];
      }
    """
    for i in range(8, -1, -1):
        tmp = [0] * 16
        print 'round ', i
        for j in range(0, 4):
            num = byte2num(a2[4 * j:4 * j + 4])
            #print num, a2[4 * j:4 * j + 4]
            tmp[4 * j
               ], tmp[4 * j + 1], tmp[4 * j + 2], tmp[4 * j + 3] = getbytes(
                   i, j, num
               )
        a2 = inv_shift_row(tmp)
    print a2
    print ''.join(chr(c) for c in a2)


if __name__ == "__main__":
    solve()
```

Running result:

```shell
➜  cracemec git:(master) ✗ python exp.py
211 3549048324
round  8
round  7
round  6
round  5
round  4
round  3
round  2
round  1
round  0
[67, 73, 83, 67, 78, 98, 35, 97, 100, 102, 115, 64, 70, 122, 57, 51]
CISCNb#adfs@Fz93
```

## Challenges

- 2018 Qiangwang Cup Finals revolver

## References

- https://zh.wikipedia.org/wiki/%E9%AB%98%E7%BA%A7%E5%8A%A0%E5%AF%86%E6%A0%87%E5%87%86
- Cryptography and Network Security， Advanced Encryption Standard  ppt
