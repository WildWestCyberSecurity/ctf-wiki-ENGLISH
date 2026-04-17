# Common Encryption Algorithm and Encoding Identification

## Introduction

In the process of transforming data, besides simple byte operations, some common encoding and encryption algorithms are also used. Therefore, if you can quickly identify the corresponding encoding or encryption algorithm, you can analyze the complete algorithm more quickly. Encryption algorithms commonly seen in CTF reverse engineering include base64, TEA, AES, RC4, MD5, etc.

## Base64

Base64 is a representation method that uses 64 printable characters to represent binary data. During conversion, 3 bytes of data are placed into a 24-bit buffer, with earlier bytes occupying higher bits. If the data is less than 3 bytes, the remaining bits in the buffer are padded with 0. Each time 6 bits are taken out (because ![{\displaystyle 2^{6}=64}](https://wikimedia.org/api/rest_v1/media/math/render/svg/c4becc8d811901597b9807eccff60f0897e3701a)), a character from `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ ` is selected as the encoded output based on its value, until all input data has been converted.





Typically, the identifying feature of Base64 is the index table. When we can find an index table like `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ `, after simple analysis we can basically determine it is Base64 encoding.

![](http://ob2hrvcxg.bkt.clouddn.com/20180822140744.png)



Of course, in some challenges the base64 index table may vary. Some Base64 variants mainly modify this index table.





## Tea

In [cryptography](https://en.wikipedia.org/wiki/Cryptography), the **Tiny Encryption Algorithm** (TEA) is a [block cipher](https://en.wikipedia.org/wiki/Block_cipher) that is easy to describe and implement, usually requiring only a small amount of code. Its designers are [David Wheeler](https://en.wikipedia.org/wiki/David_Wheeler_(computer_scientist)) and [Roger Needham](https://en.wikipedia.org/wiki/Roger_Needham) from the [Cambridge University Computer Laboratory](https://en.wikipedia.org/wiki/University_of_Cambridge).



Reference code:

```c
#include <stdint.h>

void encrypt (uint32_t* v, uint32_t* k) {
    uint32_t v0=v[0], v1=v[1], sum=0, i;           /* set up */
    uint32_t delta=0x9e3779b9;                     /* a key schedule constant */
    uint32_t k0=k[0], k1=k[1], k2=k[2], k3=k[3];   /* cache key */
    for (i=0; i < 32; i++) {                       /* basic cycle start */
        sum += delta;
        v0 += ((v1<<4) + k0) ^ (v1 + sum) ^ ((v1>>5) + k1);
        v1 += ((v0<<4) + k2) ^ (v0 + sum) ^ ((v0>>5) + k3);  
    }                                              /* end cycle */
    v[0]=v0; v[1]=v1;
}

void decrypt (uint32_t* v, uint32_t* k) {
    uint32_t v0=v[0], v1=v[1], sum=0xC6EF3720, i;  /* set up */
    uint32_t delta=0x9e3779b9;                     /* a key schedule constant */
    uint32_t k0=k[0], k1=k[1], k2=k[2], k3=k[3];   /* cache key */
    for (i=0; i<32; i++) {                         /* basic cycle start */
        v1 -= ((v0<<4) + k2) ^ (v0 + sum) ^ ((v0>>5) + k3);
        v0 -= ((v1<<4) + k0) ^ (v1 + sum) ^ ((v1>>5) + k1);
        sum -= delta;                                   
    }                                              /* end cycle */
    v[0]=v0; v[1]=v1;
}
```



The most important identifying feature of the TEA algorithm is the magic number: 0x9e3779b9. Of course, there are also modified versions of TEA. If interested, you can look at 2018 0ctf Quals milk-tea.





## RC4

In [cryptography](https://en.wikipedia.org/wiki/Cryptography), **RC4** (short for Rivest Cipher 4) is a [stream cipher](https://en.wikipedia.org/wiki/Stream_cipher) algorithm with variable [key](https://en.wikipedia.org/wiki/Key_(cryptography)) length. It uses the same key for both encryption and decryption, so it is also a [symmetric encryption algorithm](https://en.wikipedia.org/wiki/Symmetric-key_algorithm). RC4 is the encryption algorithm used in [Wired Equivalent Privacy](https://en.wikipedia.org/wiki/Wired_Equivalent_Privacy) (WEP), and was also one of the algorithms that could be used with [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security).



```C
void rc4_init(unsigned char *s, unsigned char *key, unsigned long Len) //initialization function
{
    int i =0, j = 0;
    unsigned char k[256] = {0}; // must be unsigned type, otherwise it will cause partial ciphertext errors
    unsigned char tmp = 0;
    for (i=0;i<256;i++) {
        s[i] = i;
        k[i] = key[i%Len];
    }
    for (i=0; i<256; i++) {
        j=(j+s[i]+k[i])%256;
        tmp = s[i];
        s[i] = s[j]; //swap s[i] and s[j]
        s[j] = tmp;
    }
 }

void rc4_crypt(unsigned char *s, unsigned char *Data, unsigned long Len) //encryption/decryption
{
    int i = 0, j = 0, t = 0;
    unsigned long k = 0;
    unsigned char tmp;
    for(k=0;k<Len;k++) {
        i=(i+1)%256;
        j=(j+s[i])%256;
        tmp = s[i];
        s[i] = s[j]; //swap s[x] and s[y]
        s[j] = tmp;
        t=(s[i]+s[j])%256;
        Data[k] ^= s[t];
     }
} 
```

By analyzing the initialization code, we can see that the character array s is initialized with incrementing values. Then 256 swap operations are performed on s. By identifying the initialization code, we can recognize the RC4 algorithm.

Its pseudocode representation is:

Initialize an [S-box](https://en.wikipedia.org/wiki/S-box) of length 256. The first for loop fills the S-box with non-repeating elements from 0 to 255. The second for loop shuffles the S-box based on the key.

```c
  for i from 0 to 255
     S[i] := i
 endfor
 j := 0
 for( i=0 ; i<256 ; i++)
     j := (j + S[i] + key[i mod keylength]) % 256
     swap values of S[i] and S[j]
 endfor
```

Below, i and j are two pointers. Each time a byte is received, the while loop executes. Through certain algorithms ((a),(b)), an element in the S-box is located and XORed with the input byte to obtain k. The loop also modifies the S-box ((c)). If the input is [plaintext](https://en.wikipedia.org/wiki/Plaintext), the output is [ciphertext](https://en.wikipedia.org/wiki/Ciphertext); if the input is ciphertext, the output is plaintext.

```c
 i := 0
 j := 0
 while GeneratingOutput:
     i := (i + 1) mod 256   //a
     j := (j + S[i]) mod 256 //b
     swap values of S[i] and S[j]  //c
     k := inputByte ^ S[(S[i] + S[j]) % 256]
     output K
 endwhile
```

This algorithm guarantees that every element of the S-box is swapped at least once in every 256 iterations.



### Python Decryption Script

Corresponding challenge: "From 0 to 1" RE Chapter — BabyAlgorithm

[Challenge link](https://buuoj.cn/challenges#[%E7%AC%AC%E4%BA%94%E7%AB%A0%20CTF%E4%B9%8BRE%E7%AB%A0]BabyAlgorithm)

```python
import base64
def rc4_main(key = "init_key", message = "init_message"):
    print("RC4解密主函数调用成功")
    print('\n')
    s_box = rc4_init_sbox(key)
    crypt = rc4_excrypt(message, s_box)
    return crypt
def rc4_init_sbox(key):
    s_box = list(range(256))
    print("原来的 s 盒：%s" % s_box)
    print('\n')
    j = 0
    for i in range(256):
        j = (j + s_box[i] + ord(key[i % len(key)])) % 256
        s_box[i], s_box[j] = s_box[j], s_box[i]
    print("混乱后的 s 盒：%s"% s_box)
    print('\n')
    return s_box
def rc4_excrypt(plain, box):
    print("调用解密程序成功。")
    print('\n')
    plain = base64.b64decode(plain.encode('utf-8'))
    plain = bytes.decode(plain)
    res = []
    i = j = 0
    for s in plain:
        i = (i + 1) % 256
        j = (j + box[i]) % 256
        box[i], box[j] = box[j], box[i]
        t = (box[i] + box[j]) % 256
        k = box[t]
        res.append(chr(ord(s) ^ k))
    print("res用于解密字符串，解密后是：%res" %res)
    print('\n')
    cipher = "".join(res)
    print("解密后的字符串是：%s" %cipher)
    print('\n')
    print("解密后的输出(没经过任何编码):")
    print('\n')
    return cipher
a=[] #cipher
key=""
s=""
for i in a:
    s+=chr(i)
s=str(base64.b64encode(s.encode('utf-8')), 'utf-8')
rc4_main(key, s)
```

## MD5

The **MD5 message-digest algorithm** is a widely used [cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function) that produces a 128-bit (16-[byte](https://en.wikipedia.org/wiki/Byte)) hash value, used to ensure information transmission integrity. MD5 was designed by American cryptographer [Ronald Rivest](https://en.wikipedia.org/wiki/Ron_Rivest), published in 1992, to replace the [MD4](https://en.wikipedia.org/wiki/MD4) algorithm. The algorithm is specified in [RFC 1321](https://tools.ietf.org/html/rfc1321).



Pseudocode representation:

```
/Note: All variables are unsigned 32 bits and wrap modulo 2^32 when calculating
var int[64] r, k

//r specifies the per-round shift amounts
r[ 0..15]：= {7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22} 
r[16..31]：= {5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20}
r[32..47]：= {4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23}
r[48..63]：= {6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21}

//Use binary integer part of the sines of integers as constants:
for i from 0 to 63
    k[i] := floor(abs(sin(i + 1)) × 2^32)

//Initialize variables:
var int h0 := 0x67452301
var int h1 := 0xEFCDAB89
var int h2 := 0x98BADCFE
var int h3 := 0x10325476

//Pre-processing:
append "1" bit to message
append "0" bits until message length in bits ≡ 448 (mod 512)
append bit length of message as 64-bit little-endian integer to message

//Process the message in successive 512-bit chunks:
for each 512-bit chunk of message
    break chunk into sixteen 32-bit little-endian words w[i], 0 ≤ i ≤ 15

    //Initialize hash value for this chunk:
    var int a := h0
    var int b := h1
    var int c := h2
    var int d := h3

    //Main loop:
    for i from 0 to 63
        if 0 ≤ i ≤ 15 then
            f := (b and c) or ((not b) and d)
            g := i
        else if 16 ≤ i ≤ 31
            f := (d and b) or ((not d) and c)
            g := (5×i + 1) mod 16
        else if 32 ≤ i ≤ 47
            f := b xor c xor d
            g := (3×i + 5) mod 16
        else if 48 ≤ i ≤ 63
            f := c xor (b or (not d))
            g := (7×i) mod 16
 
        temp := d
        d := c
        c := b
        b := leftrotate((a + f + k[i] + w[g]),r[i]) + b
        a := temp
    Next i
    //Add this chunk's hash to result so far:
    h0 := h0 + a
    h1 := h1 + b 
    h2 := h2 + c
    h3 := h3 + d
End ForEach
var int digest := h0 append h1 append h2 append h3 //(expressed as little-endian)
```

Its distinctive features are:

```c
    h0 = 0x67452301;
    h1 = 0xefcdab89;
    h2 = 0x98badcfe;
    h3 = 0x10325476;
```
