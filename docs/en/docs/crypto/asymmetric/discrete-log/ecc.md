# ECC

## Overview

ECC stands for Elliptic Curve Cryptography, a public key cryptosystem based on elliptic curve mathematics. Unlike traditional encryption methods based on the difficulty of factoring large prime numbers, ECC relies on the difficulty of solving the Elliptic Curve Discrete Logarithm Problem. Its main advantage is that compared to other methods, it can maintain the same cryptographic strength while using shorter key lengths. The finite fields currently mainly used for elliptic curves are:

- The integer field GF(p) modulo a prime, which is generally more efficient on general-purpose processors.
- The Galois field GF(2^m) with characteristic 2, for which specialized hardware can be designed.

## Basic Knowledge

Let's first understand elliptic curves over finite fields. An elliptic curve over a finite field means that in the defining equation of the elliptic curve

$y^2+axy+by=x^3+cx^2+dx+e$

all coefficients are elements of some finite field GF(p), where p is a large prime.

Of course, not all elliptic curves are suitable for encryption. The most commonly used equation is as follows

$y^2=x^3+ax+b$

where $4a^3+27b^2 \bmod p \neq 0$

We call the set consisting of all solutions (x,y) ($x\in Fp , y \in Fp$) of this equation, together with a point called the "point at infinity" (O), an elliptic curve defined over Fp, denoted as E(Fp).

Generally, defining elliptic curve cryptography requires the following conditions:

Assume E(Fp) forms an abelian group (commutative group, inverse elements exist, closure, etc.) under the point operation $\oplus$. Let $p\in E(Fq)$ and suppose t satisfying the following condition is very large:

$p \oplus p \oplus ... \oplus p=O$

where t copies of p participate in the operation. Here we call t the period of p. Furthermore, for $Q\in E(Fq)$, there must exist some positive integer m such that the following equation holds. We define $m=log_pq$:

$Q=m\cdot p =p \oplus p \oplus ... \oplus p$ (m copies of p participate in the operation)

Additionally, assume G is the generator of $E_q (a,b)$, meaning it can generate all elements in the group. Its order is the smallest positive integer n satisfying $nG=O$.

## ElGamal in ECC

Here we assume user B wants to encrypt a message and send it to user A.

### Key Generation

User A first selects an elliptic curve $E_q (a,b)$, then selects a generator G on it. Assume its order is n, and then selects a positive integer $n_a$ as the secret key and computes $P_a=n_aG$.

Among these, $E_q(a,b), q,G$ are all made public.

The public key is $P_a$, and the private key is $n_a $.

### Encryption

User B sends message m to user A. Here we assume message m has already been encoded as a point on the elliptic curve. The encryption steps are as follows:

1. Look up user A's public key $E_q(a,b), q, P_a,G$.
2. Select a random number k in the interval (1,q-1).
3. Compute the point $(x_1,y_1)=kG$ using A's public key.
4. Compute the point $(x_2,y_2)=kP_a$. If it is O, restart from step 2.
5. Compute $C=m+(x_2,y_2)$
6. Send $((x_1,y_1),C)$ to A.

### Decryption

The decryption steps are as follows:

1. Use the private key to compute the point $n_a(x_1,y_1)=n_akG=kP_a=(x_2,y_2)$.
2. Compute the message $m=C-(x_2,y_2)$.

### Key Point

The key point here is that even if we know $(x_1,y_1)$, it is difficult to determine k. This is determined by the difficulty of the discrete logarithm problem.

## 2013 SECCON CTF quals Cryptanalysis

Here we use the Cryptanalysis challenge from 2013 SECCON CTF quals as an example. The problem is as follows:

![img](./figure/2013-seccon-ctf-crypt-desp.png)

Here, we know the elliptic curve equation, the corresponding generator base, the modulus, the public key, and the encrypted result.

However, we can see that our modulus is too small, so we can brute-force enumerate to get the result.

Here we directly reference the sage program from GitHub to brute-force the secret key. After that, we can decrypt.

```python

a = 1234577
b = 3213242
n = 7654319

E = EllipticCurve(GF(n), [0, 0, 0, a, b])

base = E([5234568, 2287747])
pub = E([2366653, 1424308])

c1 = E([5081741, 6744615])
c2 = E([610619, 6218])

X = base

for i in range(1, n):
    if X == pub:
        secret = i
        print "[+] secret:", i
        break
    else:
        X = X + base
        print i

m = c2 - (c1 * secret)

print "[+] x:", m[0]
print "[+] y:", m[1]
print "[+] x+y:", m[0] + m[1]
```

Brute-force results:

```shell
[+] secret: 1584718
[+] x: 2171002
[+] y: 3549912
[+] x+y: 5720914
```

## References

- https://github.com/sonickun/ctf-crypto-writeups/tree/master/2013/seccon-ctf-quals/cryptanalysis
