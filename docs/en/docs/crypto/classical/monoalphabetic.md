# Monoalphabetic Substitution Cipher

## Common Characteristics

In monoalphabetic substitution ciphers, almost all encryption methods share a common characteristic: there is a one-to-one correspondence between plaintext and ciphertext. Therefore, there are generally two approaches for cracking:

- When the key space is small, use brute force
- When the ciphertext is long enough, use frequency analysis, http://quipqiup.com/

When the key space is large enough and the ciphertext is short enough, cracking becomes difficult.

## Caesar Cipher

### Principle

The Caesar cipher encrypts by shifting **each letter** in the plaintext by a fixed number of positions forward (or backward) in the alphabet (**circular shift**). For example, when the offset is a left shift of 3 (the decryption key is 3):

```
Plaintext alphabet:  ABCDEFGHIJKLMNOPQRSTUVWXYZ
Ciphertext alphabet: DEFGHIJKLMNOPQRSTUVWXYZABC
```

When encrypting, the encryptor looks up each letter of the message in the plaintext alphabet and writes down the corresponding letter from the ciphertext alphabet. The decryptor reverses the operation using the previously known key to recover the original plaintext. For example:

```
Plaintext:  THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG
Ciphertext: WKH TXLFN EURZQ IRA MXPSV RYHU WKH ODCB GRJ
```

Depending on the offset, there are **several specifically named Caesar ciphers**:

- Offset of 10: Avocat (A→K)
- Offset of 13: [ROT13](https://zh.wikipedia.org/wiki/ROT13)
- Offset of -5: Cassis (K 6)
- Offset of -6: Cassette (K 7)

Additionally, there is a key-based Caesar cipher called Keyed Caesar. Its basic principle is to **use a key, convert each character of the key to a number (generally converted to the corresponding position number in the alphabet), and encrypt each letter of the plaintext using that number as the key.**

Here we use **XMan Season 1 Summer Camp Sharing Competition - Gongbao Jiding Team Crypto 100** as an example.

```
Ciphertext: s0a6u3u1s0bv1a
Key:        guangtou
Offset:     6,20,0,13,6,19,14,20
Plaintext:  y0u6u3h1y0uj1u
```

### Cracking

For Caesar ciphers without a key, there are two basic cracking methods:

1. Enumerate all 26 offsets, applicable in general cases
2. Use frequency analysis, applicable when the ciphertext is long enough.

The first method is guaranteed to yield the plaintext, while the second method may not always produce the correct plaintext.

For key-based Caesar ciphers, the corresponding key must generally be known.

### Tools

Generally, we have the following tools, among which JPK is the most versatile.

- JPK, can solve both keyed and non-keyed variants
- http://planetcalc.com/1434/
- http://www.qqxiuzi.cn/bianma/ROT5-13-18-47.php

## Shift Cipher

Similar to the Caesar cipher, the difference is that the shift cipher processes not only letters but also numbers and special characters, commonly using the ASCII table for shifting. The cracking method is also to enumerate all possibilities to obtain possible results.

## Atbash Cipher

### Principle

The Atbash Cipher can actually be considered a special case of the simple substitution cipher introduced below. It uses the last letter of the alphabet to represent the first letter, the second-to-last letter to represent the second letter, and so on. In the Roman alphabet, it appears as follows:

```
Plaintext:  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Ciphertext: Z Y X W V U T S R Q P O N M L K J I H G F E D C B A
```

Here is an example:

```
Plaintext:  the quick brown fox jumps over the lazy dog
Ciphertext: gsv jfrxp yildm ulc qfnkh levi gsv ozab wlt
```

### Cracking

As we can see, its key space is sufficiently short, and when the ciphertext is long enough, frequency analysis can still be used to solve it.

### Tools

- http://www.practicalcryptography.com/ciphers/classical-era/atbash-cipher/

## Simple Substitution Cipher

### Principle

The Simple Substitution Cipher encrypts by replacing each plaintext letter with a uniquely corresponding and different letter. The difference from the Caesar cipher is that its cipher alphabet is not simply shifted but completely scrambled, which makes it more difficult to crack than the Caesar cipher. For example:

```
Plaintext alphabet: abcdefghijklmnopqrstuvwxyz
Key alphabet:       phqgiumeaylnofdxjkrcvstzwb
```

a maps to p, d maps to h, and so on.

```
Plaintext:  the quick brown fox jumps over the lazy dog
Ciphertext: cei jvaql hkdtf udz yvoxr dsik cei npbw gdm
```

For decryption, we generally need to know the mapping rule for each letter in order to decrypt properly.

### Cracking

Since this encryption method results in a total of $26!$ possible keys, brute force is virtually impossible. Therefore, we generally use frequency analysis.

### Tools

- http://quipqiup.com/

## Affine Cipher

### Principle

The encryption function of the affine cipher is $E(x)=(ax+b)\pmod m$, where

- $x$ represents the number obtained by encoding the plaintext according to some encoding scheme
- $a$ and $m$ are coprime
- $m$ is the number of letters in the encoding system.

The decryption function is $D(x)=a^{-1}(x-b)\pmod m$, where $a^{-1}$ is the multiplicative inverse of $a$ in the group $\mathbb{Z}_{m}$.

Below we use the function $E(x) = (5x + 8) \bmod 26$ as an example to encrypt the string `AFFINE CIPHER`, using the 26-letter alphabet as the encoding system:

| Plaintext | A   | F   | F   | I   | N   | E   | C   | I   | P   | H   | E   | R   |
| --------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| x         | 0   | 5   | 5   | 8   | 13  | 4   | 2   | 8   | 15  | 7   | 4   | 17  |
| $y=5x+8$  | 8   | 33  | 33  | 48  | 73  | 28  | 18  | 48  | 83  | 43  | 28  | 93  |
| $y\mod26$ | 8   | 7   | 7   | 22  | 21  | 2   | 18  | 22  | 5   | 17  | 2   | 15  |
| Ciphertext | I   | H   | H   | W   | V   | C   | S   | W   | F   | R   | C   | P   |

The corresponding encryption result is `IHHWVCSWFRCP`.

For the decryption process, a legitimate decryptor who has a and b can calculate $a^{-1}$ as 21, so the decryption function is $D(x)=21(x-8)\pmod {26}$. The decryption is as follows:

| Ciphertext  | I    | H    | H   | W   | V   | C    | S   | W   | F   | R   | C    | P   |
| ----------- | :--- | :--- | --- | --- | --- | ---- | --- | --- | --- | --- | ---- | --- |
| $y$         | 8    | 7    | 7   | 22  | 21  | 2    | 18  | 22  | 5   | 17  | 2    | 15  |
| $x=21(y-8)$ | 0    | -21  | -21 | 294 | 273 | -126 | 210 | 294 | -63 | 189 | -126 | 147 |
| $x\mod26$   | 0    | 5    | 5   | 8   | 13  | 4    | 2   | 8   | 15  | 7   | 4    | 17  |
| Plaintext   | A    | F    | F   | I   | N   | E    | C   | I   | P   | H   | E    | R   |

As we can see, its characteristic is that it only involves 26 English letters.

### Cracking

First, we can see that in the affine cipher, any two different letters will necessarily produce different ciphertext, so it also has the most general characteristic. When the ciphertext is long enough, we can use frequency analysis to solve it.

Second, we can consider how to attack this cipher. We can see that when $a=1$, the affine cipher becomes the Caesar cipher. Generally speaking, when using the affine cipher, the character set used is the alphabet with only 26 letters, and the number of integers less than or equal to 26 that are coprime with 26 is

$$
\phi(26)=\phi(2) \times \phi(13) = 12
$$

Counting the possible offsets of b, the total possible key space size is only

$$
12 \times 26 = 312
$$

Generally speaking, for this type of cipher, we need at least some known plaintext to mount an attack. Below is a brief analysis.

This cipher is controlled by two parameters. If we know either one of them, we can easily enumerate the other parameter quickly to find the answer.

However, assuming we already know the alphabet being used (here we assume 26 letters), we have another decryption method — we only need to know two encrypted letters $y_1,y_2$ to perform decryption. Then we can also know

$$
y_1=(ax_1+b)\pmod{26} \\
y_2=(ax_2+b)\pmod{26}
$$

Subtracting the two equations, we get

$$
y_1-y_2=a(x_1-x_2)\pmod{26}
$$

Here $y_1,y_2$ are known. If we know the two different plaintext characters $x_1$ and $x_2$ corresponding to the ciphertext, then we can easily obtain $a$, and consequently $b$.

### Example

Here we use TWCTF 2016 super_express as an example. Let's take a quick look at the provided source code:

```python
import sys
key = '****CENSORED***************'
flag = 'TWCTF{*******CENSORED********}'

if len(key) % 2 == 1:
    print("Key Length Error")
    sys.exit(1)

n = len(key) / 2
encrypted = ''
for c in flag:
    c = ord(c)
    for a, b in zip(key[0:n], key[n:2*n]):
        c = (ord(a) * c + ord(b)) % 251
    encrypted += '%02x' % c

print encrypted
```

We can see that although each letter of the flag is encrypted n times, if we analyze it carefully, we can find that

$$
\begin{align*}
c_1&=a_1c+b_1 \\
c_2&=a_2c_1+b_2 \\
   &=a_1a_2c+a_2b_1+b_2 \\
   &=kc+d
\end{align*}  
$$

Based on the derivation in the second line, we can see that $c_n$ also takes this form and can be expressed as $c_n=xc+y$. Moreover, we know that the key never changes, so this is essentially an affine cipher.

In addition, the challenge also provides the ciphertext and some known plaintext corresponding to part of the ciphertext, so we can easily use a known-plaintext attack. The code is as follows:

```python
import gmpy

key = '****CENSORED****************'
flag = 'TWCTF{*******CENSORED********}'

f = open('encrypted', 'r')
data = f.read().strip('\n')
encrypted = [int(data[i:i + 2], 16) for i in range(0, len(data), 2)]
plaindelta = ord(flag[1]) - ord(flag[0])
cipherdalte = encrypted[1] - encrypted[0]
a = gmpy.invert(plaindelta, 251) * cipherdalte % 251
b = (encrypted[0] - a * ord(flag[0])) % 251
a_inv = gmpy.invert(a, 251)
result = ""
for c in encrypted:
    result += chr((c - b) * a_inv % 251)
print result
```

The result is as follows:

```shell
➜  TWCTF2016-super_express git:(master) ✗ python exploit.py
TWCTF{Faster_Than_Shinkansen!}
```
