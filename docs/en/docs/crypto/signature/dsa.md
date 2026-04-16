# DSA

The ElGamal signature algorithm described above is not commonly used in practice; its variant DSA is more commonly used.

## Basic Principles

### Key Generation

1. Choose a suitable hash function. Currently SHA1 is generally chosen, though a stronger hash function H can also be selected.
2. Choose key lengths L and N, which determine the security level of the signature. In the original DSS (**Digital Signature Standard**), it was recommended that L must be a multiple of 64, and $512 \leq L \leq 1024$, though it can also be larger. N must not exceed the output length of the hash function H. FIPS 186-3 provides some suggested examples of L and N values: (1024, 160), (2048, 224), (2048, 256), and (3,072, 256).
3. Choose an N-bit prime q.
4. Choose an L-bit prime p such that p-1 is a multiple of q.
5. Choose g such that the smallest positive integer k satisfying $g^k \equiv 1 \bmod p$ is q, i.e., in the context of modulo p, ord(g)=q. That is, g can generate a subgroup with q elements under exponentiation modulo p. Here, we can compute $g=h^{\frac{p-1}{q}} \bmod p$ where $1< h < p-1$.
6. Choose private key x, $0<x<q$, and compute $y \equiv g^x \bmod p$.

The public key is (p,q,g,y), and the private key is (x).

### Signing

The signing steps are as follows:

1. Choose a random integer k as a temporary key, $0<k<q$.
2. Compute $r\equiv (g^k \bmod p) \bmod q$
3. Compute $s\equiv (H(m)+xr)k^{-1} \bmod q$

The signature result is (r,s). Note that an important difference from ElGamal is that a hash function is used here to hash the message.

### Verification

The verification process is as follows:

1. Compute auxiliary value $w=s^{-1} \bmod q$
2. Compute auxiliary value $u_1=H(m)w \bmod q$
3. Compute auxiliary value $u_2=rw \bmod q$
4. Compute $v=(g^{u_1}y^{u_2} \bmod p) \bmod q$
5. If v equals r, the verification succeeds.

### Correctness Proof

First, g satisfies that the smallest positive integer k for which $g^k \equiv 1 \bmod p$ is q. So $g^q \equiv 1 \bmod p$. Therefore $g^x \equiv g^{x \bmod q} \bmod p$. Furthermore,

$v=(g^{u_1}y^{u_2} \bmod p) \bmod q=g^{u_1}g^{xu_2} \equiv g^{H(m)w}g^{xrw} \equiv g^{H(m)w+xrw}$

Since $s\equiv (H(m)+xr)k^{-1} \bmod q$ and $w=s^{-1} \bmod q$, we have

$k \equiv s^{-1}(H(m)+xr) \equiv H(m)w+xrw \bmod q$

Therefore $v \equiv g^k$. The correctness is proven.

## Security

### Known k

#### Principle

If the random key k is known, then we can compute the private key d from $s\equiv (H(m)+xr)k^{-1} \bmod q$, essentially breaking DSA.

Here, in general cases, the hash value of the message will be given.

$x \equiv r^{-1}(ks-H(m)) \bmod q$

### Shared k

#### Principle

If k is shared across two signing operations, we can perform an attack.

Assume the signed messages are m1 and m2. Obviously, the r values for both are the same. Additionally,

$s_1\equiv (H(m_1)+xr)k^{-1} \bmod q$

$s_2\equiv (H(m_2)+xr)k^{-1} \bmod q$

Here, everything except x and k is known, so

$s_1k \equiv H(m_1)+xr$

$s_2k \equiv H(m_2)+xr$

Subtracting the two equations:

$k(s_1-s_2) \equiv H(m_1)-H(m_2) \bmod q$

At this point, k can be solved, and further we can solve for x.

#### Example

Here we use the DSA challenge from the 湖湘杯 (Huxiang Cup) as an example, but it cannot be solved directly because the signature verification fails for message4. I no longer have the original challenge. Here we use the modified DSA challenge from Jarvis OJ as an example.

```shell
➜  2016湖湘杯DSA git:(master) ✗ openssl sha1 -verify dsa_public.pem -signature packet1/sign1.bin  packet1/message1  
Verified OK
➜  2016湖湘杯DSA git:(master) ✗ openssl sha1 -verify dsa_public.pem -signature packet2/sign2.bin  packet2/message1 
packet2/message1: No such file or directory
➜  2016湖湘杯DSA git:(master) ✗ openssl sha1 -verify dsa_public.pem -signature packet2/sign2.bin  packet2/message2 
Verified OK
➜  2016湖湘杯DSA git:(master) ✗ openssl sha1 -verify dsa_public.pem -signature packet3/sign3.bin  packet3/message3 
Verified OK
➜  2016湖湘杯DSA git:(master) ✗ openssl sha1 -verify dsa_public.pem -signature packet4/sign4.bin  packet4/message4
Verified OK
```

We can see that all four messages pass verification. The reason we think of shared k is that the challenge hints that this method was used to crack the PS3, and searching online reveals this attack.

Next, let's look at the signed values. The command used here is as follows:

```shell
➜  2016湖湘杯DSA git:(master) ✗ openssl asn1parse -inform der -in packet4/sign4.bin  
    0:d=0  hl=2 l=  44 cons: SEQUENCE          
    2:d=1  hl=2 l=  20 prim: INTEGER           :5090DA81FEDE048D706D80E0AC47701E5A9EF1CC
   24:d=1  hl=2 l=  20 prim: INTEGER           :5E10DED084203CCBCEC3356A2CA02FF318FD4123
➜  2016湖湘杯DSA git:(master) ✗ openssl asn1parse -inform der -in packet3/sign3.bin  
    0:d=0  hl=2 l=  44 cons: SEQUENCE          
    2:d=1  hl=2 l=  20 prim: INTEGER           :5090DA81FEDE048D706D80E0AC47701E5A9EF1CC
   24:d=1  hl=2 l=  20 prim: INTEGER           :30EB88E6A4BFB1B16728A974210AE4E41B42677D
➜  2016湖湘杯DSA git:(master) ✗ openssl asn1parse -inform der -in packet2/sign2.bin  
    0:d=0  hl=2 l=  44 cons: SEQUENCE          
    2:d=1  hl=2 l=  20 prim: INTEGER           :60B9F2A5BA689B802942D667ED5D1EED066C5A7F
   24:d=1  hl=2 l=  20 prim: INTEGER           :3DC8921BA26B514F4D991A85482750E0225A15B5
➜  2016湖湘杯DSA git:(master) ✗ openssl asn1parse -inform der -in packet1/sign1.bin  
    0:d=0  hl=2 l=  45 cons: SEQUENCE          
    2:d=1  hl=2 l=  21 prim: INTEGER           :8158B477C5AA033D650596E93653C730D26BA409
   25:d=1  hl=2 l=  20 prim: INTEGER           :165B9DD1C93230C31111E5A4E6EB5181F990F702

```

The first value obtained is r and the second value is s. We can see that packet 4 and packet 3 share k because their r values are the same.

Here we can use openssl to view the public key:

```shell
➜  2016湖湘杯DSA git:(master) ✗ openssl dsa -in dsa_public.pem -text -noout  -pubin 
read DSA key
pub: 
    45:bb:18:f6:0e:b0:51:f9:d4:82:18:df:8c:d9:56:
    33:0a:4f:f3:0a:f5:34:4f:6c:95:40:06:1d:53:83:
    29:2d:95:c4:df:c8:ac:26:ca:45:2e:17:0d:c7:9b:
    e1:5c:c6:15:9e:03:7b:cc:f5:64:ef:36:1c:18:c9:
    9e:8a:eb:0b:c1:ac:f9:c0:c3:5d:62:0d:60:bb:73:
    11:f1:cf:08:cf:bc:34:cc:aa:79:ef:1d:ad:8a:7a:
    6f:ac:ce:86:65:90:06:d4:fa:f0:57:71:68:57:ec:
    7c:a6:04:ad:e2:c3:d7:31:d6:d0:2f:93:31:98:d3:
    90:c3:ef:c3:f3:ff:04:6f
P:   
    00:c0:59:6c:3b:5e:93:3d:33:78:be:36:26:be:31:
    5e:e7:0c:a6:b5:b1:1a:51:9b:55:23:d4:0e:5b:a7:
    45:66:e2:2c:c8:8b:fe:c5:6a:ad:66:91:8b:9b:30:
    ad:28:13:88:f0:bb:c6:b8:02:6b:7c:80:26:e9:11:
    84:be:e0:c8:ad:10:cc:f2:96:be:cf:e5:05:05:38:
    3c:b4:a9:54:b3:7c:b5:88:67:2f:7c:09:57:b6:fd:
    f2:fa:05:38:fd:ad:83:93:4a:45:e4:f9:9d:38:de:
    57:c0:8a:24:d0:0d:1c:c5:d5:fb:db:73:29:1c:d1:
    0c:e7:57:68:90:b6:ba:08:9b
Q:   
    00:86:8f:78:b8:c8:50:0b:eb:f6:7a:58:e3:3c:1f:
    53:9d:35:70:d1:bd
G:   
    4c:d5:e6:b6:6a:6e:b7:e9:27:94:e3:61:1f:41:53:
    cb:11:af:5a:08:d9:d4:f8:a3:f2:50:03:72:91:ba:
    5f:ff:3c:29:a8:c3:7b:c4:ee:5f:98:ec:17:f4:18:
    bc:71:61:01:6c:94:c8:49:02:e4:00:3a:79:87:f0:
    d8:cf:6a:61:c1:3a:fd:56:73:ca:a5:fb:41:15:08:
    cd:b3:50:1b:df:f7:3e:74:79:25:f7:65:86:f4:07:
    9f:ea:12:09:8b:34:50:84:4a:2a:9e:5d:0a:99:bd:
    86:5e:05:70:d5:19:7d:f4:a1:c9:b8:01:8f:b9:9c:
    dc:e9:15:7b:98:50:01:79
```

Now, we can directly write a program based on the above principles. The program is as follows:

```python
#coding=utf8
from Crypto.PublicKey import DSA
from hashlib import sha1
import gmpy2
with open('./dsa_public.pem') as f:
    key = DSA.importKey(f)
    y = key.y
    g = key.g
    p = key.p
    q = key.q
f3 = open(r"packet3/message3", 'r')
f4 = open(r"packet4/message4", 'r')
data3 = f3.read()
data4 = f4.read()
sha = sha1()
sha.update(data3)
m3 = int(sha.hexdigest(), 16)
sha = sha1()
sha.update(data4)
m4 = int(sha.hexdigest(), 16)
print m3, m4
s3 = 0x30EB88E6A4BFB1B16728A974210AE4E41B42677D
s4 = 0x5E10DED084203CCBCEC3356A2CA02FF318FD4123
r = 0x5090DA81FEDE048D706D80E0AC47701E5A9EF1CC
ds = s4 - s3
dm = m4 - m3
k = gmpy2.mul(dm, gmpy2.invert(ds, q))
k = gmpy2.f_mod(k, q)
tmp = gmpy2.mul(k, s3) - m3
x = tmp * gmpy2.invert(r, q)
x = gmpy2.f_mod(x, q)
print int(x)
```

**I found that the pycrypto installed via pip doesn't have the DSA importKey function... So I had to download and install pycrypto from GitHub...**

The result is as follows:

```shell
➜  2016湖湘杯DSA git:(master) ✗ python exp.py
1104884177962524221174509726811256177146235961550 943735132044536149000710760545778628181961840230
520793588153805320783422521615148687785086070744
```
