# Discrete Logarithm

## Basic Definitions

Before understanding discrete logarithms, let's first learn a few basic definitions.

**Definition 1**

In a group G, g is the generator of G, meaning every element in group G can be written as $y=g^k$. We call k the logarithm of y in group G.

**Definition 2**

Let $m\geq 1$, $(a,m)=1$. The smallest positive integer d such that $a^d \equiv 1\pmod m$ holds is called the order (or index) of a modulo m. We generally denote it as $\delta_m(a)$.

**Definition 3**

When $\delta_m(a)=\varphi(m)$, we say a is a primitive root modulo m, or simply a primitive root of m.

## Some Properties

**Property 1**

The smallest positive integer $d$ such that $a^d \equiv 1\pmod m$ holds must satisfy $d\mid\varphi(m)$.

**Property 2**

A primitive root exists in the residue system modulo $m$ if and only if $m=2,4,p^{\alpha},2p^{\alpha}$, where $p$ is an odd prime and $\alpha$ is a positive integer.

## The Discrete Logarithm Problem

Given $g,p,y$, solving for $x$ in the equation $y\equiv g^x \pmod p$ is a hard problem. However, when $p$ has certain properties, it may be possible to solve. For example, when the order of the group is a smooth number.

It is precisely this problem that forms the basis of a large portion of modern cryptography, including Diffie–Hellman key exchange, the ElGamal algorithm, ECC, and others.

## Methods for Solving the Discrete Logarithm

### Brute Force

Given $y\equiv g^x \pmod p$, we can brute-force enumerate $x$ to obtain the true value of $x$.

### Baby-step giant-step

This method is commonly known as the baby-step giant-step algorithm, and it uses the idea of a meet-in-the-middle attack.

We can let $x=im+j$, where $m= \lceil \sqrt n\rceil$, so that integers i and j are both in the range from 0 to m.

Therefore

$$y=g^x=g^{im+j}$$

That is

$$y(g^{-m})^i=g^j$$

Then we can enumerate all j values, compute them, and store them in a set S. Next, we enumerate i and compute $y(g^{-m})^i$. Once we find that the computed result is in set S, it means we have found a collision, and thus obtained i and j.

This is clearly a time-space tradeoff. We transform an algorithm with $O(n)$ time complexity and $O(1)$ space complexity into an algorithm with $O(\sqrt n)$ time complexity and $O(\sqrt n)$ space complexity.

Where

- Each increment of j represents a "baby-step", multiplying by $g$ once.
- Each increment of i represents a "giant-step", multiplying by $g^{-m}$ once.

```python
def bsgs(g, y, p):
    m = int(ceil(sqrt(p - 1)))
    S = {pow(g, j, p): j for j in range(m)}
    gs = pow(g, p - 1 - m, p)
    for i in range(m):
        if y in S:
            return i * m + S[y]
        y = y * gs % p
    return None
```

### Pollard's ρ algorithm

We can solve the above problem with $O(\sqrt n)$ time complexity and $O(1)$ space complexity. Please search for the specific principles on your own.

### Pollard's kangaroo algorithm

If we know the range of x is $a \leq x \leq b$, then we can solve the above problem with $O(\sqrt{b-a})$ time complexity. Please search for the specific principles on your own.

### Pohlig-Hellman algorithm

Let us assume the order of the group with respect to element $g$ is $n$, and $n$ is a smooth number: $n=\prod\limits_{i=1}^r p_i^{e_i}$.

1. For each $i \in \{1,\ldots,r\}$:
    1. Compute $g_i \equiv g^{n/p_i^{e_i}} \pmod m$. By Lagrange's theorem, the order of $g_i$ in the group is $p_i^{e_i}$.
    2. Compute $y_i \equiv y^{n/p_i^{e_i}} \equiv g^{xn/p_i^{e_i}} \equiv g_i^{x} \equiv g_i^{x \bmod p_i^{e_i}} \equiv g_i^{x_i} \pmod m$. Here we know $y_i,m,g_i$, and the range of $x_i$ is $[0,p_i^{e_i})$. Since $n$ is a smooth number, the range is relatively small, so we can use methods like *Pollard's kangaroo algorithm* to quickly find $x_i$.
2. Based on the above derivation, we can obtain for $i \in \{1,\ldots,r\}$, $x \equiv x_i \pmod{p_i^{e_i}}$, which can be solved using the Chinese Remainder Theorem.


The above process can be briefly described by the following diagram:

<center>
![Pohlig Hellman Algorithm](figure/Pohlig-Hellman-Diagram.png)
</center>

Its complexity is $O\left(\sum\limits _i e_i\left(\log n+\sqrt{p_i}\right)\right)$, which is quite low as we can see.

However, when $n$ is prime and $m=2n+1$, the complexity is almost no different from $O(\sqrt m)$.

## 2018 National Competition crackme java

The code is as follows

```java
import java.math.BigInteger;
import java.util.Random;

public class Test1 {
    static BigInteger two =new BigInteger("2");
    static BigInteger p = new BigInteger("11360738295177002998495384057893129964980131806509572927886675899422214174408333932150813939357279703161556767193621832795605708456628733877084015367497711");
    static BigInteger h= new BigInteger("7854998893567208831270627233155763658947405610938106998083991389307363085837028364154809577816577515021560985491707606165788274218742692875308216243966916");

    /*
     Alice write the below algorithm for encryption.
     The public key {p, h} is broadcasted to everyone.
    @param val: The plaintext to encrypt.
        We suppose val only contains lowercase letter {a-z} and numeric charactors, and is at most 256 charactors in length.
    */
    public static String pkEnc(String val){
        BigInteger[] ret = new BigInteger[2];
        BigInteger bVal=new BigInteger(val.toLowerCase(),36);
        BigInteger r =new BigInteger(new Random().nextInt()+"");
        ret[0]=two.modPow(r,p);
        ret[1]=h.modPow(r,p).multiply(bVal);
        return ret[0].toString(36)+"=="+ret[1].toString(36);
    }

    /* Alice write the below algorithm for decryption. x is her private key, which she will never let you know.
    public static String skDec(String val,BigInteger x){
        if(!val.contains("==")){
            return null;
        }
        else {
            BigInteger val0=new BigInteger(val.split("==")[0],36);
            BigInteger val1=new BigInteger(val.split("==")[1],36);
            BigInteger s=val0.modPow(x,p).modInverse(p);
            return val1.multiply(s).mod(p).toString(36);
        }
    }
   */

    public static void main(String[] args) throws Exception {
        System.out.println("You intercepted the following message, which is sent from Bob to Alice:");
        BigInteger bVal1=new BigInteger("a9hgrei38ez78hl2kkd6nvookaodyidgti7d9mbvctx3jjniezhlxs1b1xz9m0dzcexwiyhi4nhvazhhj8dwb91e7lbbxa4ieco",36);
	BigInteger bVal2=new BigInteger("2q17m8ajs7509yl9iy39g4znf08bw3b33vibipaa1xt5b8lcmgmk6i5w4830yd3fdqfbqaf82386z5odwssyo3t93y91xqd5jb0zbgvkb00fcmo53sa8eblgw6vahl80ykxeylpr4bpv32p7flvhdtwl4cxqzc",36);
	BigInteger r =new BigInteger(new Random().nextInt()+"");
	System.out.println(r);
        System.out.println(bVal1);
	System.out.println(bVal2);
	System.out.println("a9hgrei38ez78hl2kkd6nvookaodyidgti7d9mbvctx3jjniezhlxs1b1xz9m0dzcexwiyhi4nhvazhhj8dwb91e7lbbxa4ieco==2q17m8ajs7509yl9iy39g4znf08bw3b33vibipaa1xt5b8lcmgmk6i5w4830yd3fdqfbqaf82386z5odwssyo3t93y91xqd5jb0zbgvkb00fcmo53sa8eblgw6vahl80ykxeylpr4bpv32p7flvhdtwl4cxqzc");
        System.out.println("Please figure out the plaintext!");
    }
}
```

The basic functionality computes

$r_0=2^r \bmod p$

$r_1 =b*h^r \bmod p$

We can see that the range of r is $[0,2^{32})$, so we can use the BSGS algorithm as follows

```python
from sage.all import *

c1 = int(
    'a9hgrei38ez78hl2kkd6nvookaodyidgti7d9mbvctx3jjniezhlxs1b1xz9m0dzcexwiyhi4nhvazhhj8dwb91e7lbbxa4ieco',
    36
)
c2 = int(
    '2q17m8ajs7509yl9iy39g4znf08bw3b33vibipaa1xt5b8lcmgmk6i5w4830yd3fdqfbqaf82386z5odwssyo3t93y91xqd5jb0zbgvkb00fcmo53sa8eblgw6vahl80ykxeylpr4bpv32p7flvhdtwl4cxqzc',
    36
)
print c1, c2
p = 11360738295177002998495384057893129964980131806509572927886675899422214174408333932150813939357279703161556767193621832795605708456628733877084015367497711
h = 7854998893567208831270627233155763658947405610938106998083991389307363085837028364154809577816577515021560985491707606165788274218742692875308216243966916
# generate the group
const2 = 2
const2 = Mod(const2, p)
c1 = Mod(c1, p)
c2 = Mod(c2, p)
h = Mod(h, p)
print '2', bsgs(const2, c1, bounds=(1, 2 ^ 32))

r = 152351913

num = long(c2 / (h**r))
print num
```

## References

- 初等数论，潘承洞，潘承彪
- https://ee.stanford.edu/~hellman/publications/28.pdf
- https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm#cite_note-Menezes97p108-2
- https://fortenf.org/e/crypto/2017/12/03/survey-of-discrete-log-algos.html
