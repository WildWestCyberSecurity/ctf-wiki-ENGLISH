# Summary

## Classical Cipher Analysis Approach

In CTF challenges involving classical ciphers, the task is usually to recover the plaintext from the ciphertext. Therefore, **ciphertext-only attacks** are most commonly used. The basic analysis approach is summarized as follows:

1. Determine the cipher type: Based on challenge hints, encryption method, ciphertext character set, ciphertext format, and other information.
2. Determine the attack method: Including direct analysis, brute force attack, statistical analysis, and other methods. For special ciphers whose type cannot be determined, an appropriate attack method should be selected based on the cipher's characteristics.
3. Determine the analysis tools: Primarily use online cipher analysis tools and Python script toolkits, supplemented by offline cipher analysis tools and manual analysis.

The applicable scenarios and examples for the above ciphertext-only attack methods are as follows:

| Attack Method      | Applicable Scenarios                                              | Examples                                                   |
| ------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------- |
| Direct Analysis    | Substitution ciphers where the mapping can be determined by type  | Caesar cipher, Pigpen cipher, Keyboard cipher, etc.        |
| Brute Force Attack | Substitution or transposition ciphers with small key spaces       | Shift cipher, Rail Fence cipher, etc.                      |
| Statistical Analysis | Substitution ciphers with large key spaces                      | Simple substitution cipher, Affine cipher, Vigenère cipher, etc. |

## Shiyanbar - Love Enclosed in a Fence

Problem Description

> I've been curious about a question lately, does QWE equal ABC or not?
>
> -.- .. --.- .-.. .-- - ..-. -.-. --.- --. -. ... --- ---
>
> flag format: CTF{xxx}

First, based on the cipher style, it is identified as Morse code. Decrypting it yields `KIQLWTFCQGNSOO`, which doesn't look like a flag. The challenge also mentions a fence and `does QWE equal ABC`. After trying both, it turns out that applying QWE first and then the Rail Fence cipher gives the result.

First, QWE keyboard decryption gives `IILYOAVNEBSAHR`. Then Rail Fence decryption gives `ILOVESHIYANBAR`.

## 2017 SECCON Vigenere3d

The program is as follows

```python
# Vigenere3d.py
import sys
def _l(idx, s):
    return s[idx:] + s[:idx]
def main(p, k1, k2):
    s = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz_{}"
    t = [[_l((i+j) % len(s), s) for j in range(len(s))] for i in range(len(s))]
    i1 = 0
    i2 = 0
    c = ""
    for a in p:
        c += t[s.find(a)][s.find(k1[i1])][s.find(k2[i2])]
        i1 = (i1 + 1) % len(k1)
        i2 = (i2 + 1) % len(k2)
    return c
print main(sys.argv[1], sys.argv[2], sys.argv[2][::-1])

$ python Vigenere3d.py SECCON{**************************} **************
POR4dnyTLHBfwbxAAZhe}}ocZR3Cxcftw9
```

**Solution 1**:

First, let's analyze the structure of t
$$
t[i][j]=s[i+j:]+s[:i+j] \\
t[i][k]=s[i+k:]+s[:i+k]
$$

$t[i][j][k]$ is the k-th character in $t[i][j]$, and $t[i][k][j]$ is the j-th character in $t[i][k]$. Regardless of whether $i+j+k$ exceeds `len(s)`, the two always remain consistent, i.e., $t[i][j][k]=t[i][k][j]$.

Therefore, for the same plaintext, there may be multiple keys that generate the same ciphertext.

However, the above analysis is purely analytical. Now let's get to the main topic.

It is easy to see that each character of the ciphertext only depends on the corresponding character of the plaintext, and the space for each key character is at most the size of s, so we can use brute force to recover the key. Based on the command line hint above, we know the key length is 14, and the first 7 bytes of the plaintext are already known. The key recovery script is as follows

```python
def get_key(plain, cipher):
    s = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz_{}"
    t = [[_l((i + j) % len(s), s) for j in range(len(s))]
         for i in range(len(s))]
    i1 = 0
    i2 = 0
    key = ['*'] * 14
    for i in range(len(plain)):
        for i1 in range(len(s)):
            for i2 in range(len(s)):
                if t[s.find(plain[i])][s.find(s[i1])][s.find(s[i2])] == cipher[
                        i]:
                    key[i] = s[i1]
                    key[13 - i] = s[i2]
    return ''.join(key)
```

The plaintext recovery script is as follows

```python
def decrypt(cipher, k1, k2):
    s = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz_{}"
    t = [[_l((i + j) % len(s), s) for j in range(len(s))]
         for i in range(len(s))]
    i1 = 0
    i2 = 0
    plain = ""
    for a in cipher:
        for i in range(len(s)):
            if t[i][s.find(k1[i1])][s.find(k2[i2])] == a:
                plain += s[i]
                break
        i1 = (i1 + 1) % len(k1)
        i2 = (i2 + 1) % len(k2)
    return plain
```

The recovered plaintext is as follows

```shell
➜  2017_seccon_vigenere3d git:(master) python exp.py
SECCON{Welc0me_to_SECCON_CTF_2017}
```
**Solution 2**

Analysis of this challenge:

1. Considering that under normal program execution array accesses do not go out of bounds, we adopt the following convention in our discussion: $arr[index] \Leftrightarrow arr[index \% len(arr)]$
2. Regarding the `_l` function defined in the Python program, we find the following equivalence: $\_l(offset, arr)[index] \Leftrightarrow arr[index + offset]$
3. Regarding the three-dimensional matrix t defined in Python's main function, we find the following equivalence: $t[a][b][c] \Leftrightarrow \_l(a+b, s)[c]$
4. Combining the observations from points 2 and 3, we have the following equivalence: $t[a][b][c] \Leftrightarrow s[a+b+c]$
5. We treat s as an encoding format, i.e., encoding process s.find(x), decoding process s[x]. Using the numeric result of the encoding directly to represent the corresponding string, the encryption process can be expressed by the following formula:
   - $e = f +  k1 +k2$
   - Where e is the ciphertext, f is the plaintext, and k1 and k2 are keys obtained by repetition to match the length of f, **the addition is vector addition**.

So we only need to compute `k1+k2` to simulate the key for decryption. The Python decryption script for this challenge:

```python
# exp2.py
enc_str = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz_{}'
dec_dic = {k:v for v,k in enumerate(enc_str)}
encrypt = 'POR4dnyTLHBfwbxAAZhe}}ocZR3Cxcftw9'
flag_bg = 'SECCON{**************************}'

sim_key = [dec_dic[encrypt[i]]-dec_dic[flag_bg[i]] for i in range(7)] # 破解模拟密钥
sim_key = sim_key + sim_key[::-1]

flag_ed = [dec_dic[v]-sim_key[k%14] for k,v in enumerate(encrypt)] # 模拟密钥解密
flag_ed = ''.join([enc_str[i%len(enc_str)] for i in flag_ed]) # 解码
print(flag_ed)
```

The recovered plaintext is as follows:

```bash
$ python exp2.py
SECCON{Welc0me_to_SECCON_CTF_2017}
```

## The Vanishing Triple Cipher

Ciphertext
```
of zit kggd zitkt qkt ygxk ortfzoeqs wqlatzwqssl qfr zvg ortfzoeqs yggzwqssl. fgv oy ngx vqfz zg hxz zitd of gft soft.piv dgfn lgsxzogfl qkt zitkt? zohl:hstqlt eiqfut zit ygkd gy zit fxdwtk ngx utz.zit hkgukqddtkl!
```

Decrypt directly using quipquip.
