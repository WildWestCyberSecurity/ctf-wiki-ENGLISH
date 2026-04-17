# Modulus-Related Attacks

## Brute Force Factoring N

### Attack Conditions

When N has fewer than 512 bits, we can use integer factorization strategies to obtain p and q.

### JarvisOJ - Easy RSA

Here we use "JarvisOJ - Easy RSA" as an example. The problem is as follows:

> Remember veryeasy RSA? Wasn't it easy? Then let's continue and look at this one, it's also not hard.  
> A piece of RSA encrypted information is known to be: 0xdc2eeeb2782c and the public key used for encryption is known:  
> N=322831561921859 e = 23  
> Please decrypt the plaintext. When submitting, please convert the number to ASCII and submit.  
> For example, if the plaintext you decrypted is 0x6162, then please submit the string ab  
> Submission format: `PCTF{plaintext string}`

As we can see, our N is relatively small. Here we directly use factordb to factor it, and we get

$$
322831561921859 = 13574881 \times 23781539
$$

Then we simply write the following program

```python
import gmpy2
p = 13574881
q = 23781539
n = p * q
e = 23
c = 0xdc2eeeb2782c
phin = (p - 1) * (q - 1)
d = gmpy2.invert(e, phin)
p = gmpy2.powmod(c, d, n)
tmp = hex(p)
print tmp, tmp[2:].decode('hex')
```

The result is as follows

```shell
➜  Jarvis OJ-Basic-easyRSA git:(master) ✗ python exp.py
0x33613559 3a5Y
```

## Improper p & q Leading to Factoring N

### Attack Conditions

When p and q are improperly chosen in RSA, we can also perform attacks.

### |p-q| is Very Large

When p-q is very large, one of the parameters must be relatively small. Here we assume it is p. Then we can use an exhaustive method to perform trial division on the modulus, thereby factoring the modulus to obtain the secret parameters and plaintext information. Basically, this is not very feasible.

### |p-q| is Relatively Small

First

$$
\frac{(p+q)^2}{4}-n=\frac{(p+q)^2}{4}-pq=\frac{(p-q)^2}{4}
$$

Since |p-q| is relatively small, $\frac{(p-q)^2}{4}$ is naturally also relatively small, and thus $\frac{(p+q)^2}{4}$ is only slightly larger than N, so $\frac{p+q}{2}$ is close to $\sqrt{n}$. Then we can factor as follows:

- Sequentially check each integer x starting from $\sqrt{n}$ until we find an x such that $x^2-n$ is a perfect square, denoted as $y^2$
- Then $x^2-n=y^2$, and we can factor N using the difference of squares formula

### p - 1 is Smooth

* Smooth number: A positive integer that can be factored into a product of small primes

* When $p$ is a factor of $N$ and $p-1$ is a smooth number, we can consider using `Pollard's p-1` algorithm to factor $N$

* According to Fermat's Little Theorem

    $$\text{If } p\nmid a,\ \text{then } a^{p-1}\equiv 1\pmod{p}$$

    Then we have

    $$a^{t(p-1)}\equiv 1^t \equiv 1\pmod{p}$$

    That is

    $$a^{t(p-1)} - 1 = k*p$$

* According to `Pollard's p-1` algorithm:

    If $p$ is a $B\text{-smooth number}$, then there exists

    $$M = \prod_{q\le{B}}{q^{\lfloor\log_q{B}\rfloor}}$$

    such that

    $$(p-1)\mid M$$
    
    holds, then we have

    $$\gcd{(a^{M}-1, N)}$$
    
    If the result is not $1$ or $N$, then we have successfully factored $N$.

    Since we only care about the final gcd result, and `N` contains only two prime factors, we don't need to compute $M$. Consider $n=2,3,\dots$, letting $M = n!$ is sufficient to cover the correct $M$ while being convenient to compute.

* In practical computation, we can substitute and reduce the exponent for calculation

    $$
    a^{n!}\bmod{N}=\begin{cases}
        (a\bmod{N})^2\mod{N}&n=2\\
        (a^{(n-1)!}\bmod{N})^n\mod{N}&n\ge{3}
    \end{cases}$$

* Python code implementation

    ```python
    from gmpy2 import *
    a = 2
    n = 2
    while True:
        a = powmod(a, n, N)
        res = gcd(a-1, N)
        if res != 1 and res != N:
            q = N // res
            d = invert(e, (res-1)*(q-1))
            m = powmod(c, d, N)
            print(m)
            break
        n += 1
    ```

### p + 1 is Smooth

* When $p$ is a factor of $N$ and $p+1$ is a smooth number, we can consider using `Williams's p+1` algorithm to factor $N$

* Given that $p$ is a factor of $N$, and $p+1$ is a smooth number

    $$
    p = \left(\prod_{i=1}^k{q_i^{\alpha_i}}\right)+1
    $$

    $q_i$ is the $i$-th prime factor and $q_i^{\alpha_i}\le B_1$. Find $\beta_i$ such that $q_i^{\beta_i}\le B_1$ and $q_i^{\beta_i+1}> B_1$, then let

    $$
    R = \prod_{i=1}^k{q_i^{\beta_i}}
    $$

    Obviously $(p-1)\mid R$ and when $(N, a) = 1$ we have $a^{p-1}\equiv 1 \pmod{p}$, so $a^R\equiv 1\pmod{p}$, that is

    $$
        p\mid(N, a^R-1)
    $$

* Let $P,Q$ be integers, $\alpha,\beta$ be roots of the equation $x^2-Px+Q=0$, define the following Lucas-like sequence

    $$
    \begin{aligned}
        U_n(P, Q) &= (\alpha^n -\beta^n)/(\alpha - \beta)\\
        V_n(P, Q) &= \alpha^n + \beta^n
    \end{aligned}
    $$

    Similarly $\Delta = (\alpha - \beta)^2 = P^2-4Q$, then we have

    $$
    \begin{cases}
        U_{n+1} &= PU_n - QU_{n-1}\\
        V_{n+1} &= PV_n - QV_{n-1}
    \end{cases}\tag{2.2}
    $$

    $$
    \begin{cases}
        U_{2n} &= V_nU_n\\
        V_{2n} &= V_n^2 - 2Q^n
    \end{cases}\tag{2.3}
    $$

    $$
    \begin{cases}
        U_{2n-1} &= U_n^2 - QU_{n-1}^2\\
        V_{2n-1} &= V_nV_{n-1} - PQ^{n-1}
    \end{cases}\tag{2.4}
    $$

    $$
    \begin{cases}
        \Delta U_{n} &= PV_n - 2QV_{n-1}\\
        V_{n} &= PU_n - 2QU_{n-1}
    \end{cases}\tag{2.5}
    $$

    $$
    \begin{cases}
        U_{m+n} &= U_mU_{n+1} - QU_{m-1}U_n\\
        \Delta U_{m+n} &= V_mV_{n+1} - QV_{m-1}V_n
    \end{cases}\tag{2.6}
    $$

    $$
    \begin{cases}
        U_{n}(V_k(P, Q), Q^k) &= U_{nk}(P, Q)/U_k(P, Q)\\
        V_{n}(V_k(P, Q), Q^k) &= V_n(P, Q)
    \end{cases}\tag{2.7}
    $$

    Additionally, if $(N, Q) = 1$ and $P^{'}Q\equiv P^2-2Q\pmod{N}$, then $P^{'}\equiv \alpha/\beta + \beta/\alpha$ and $Q^{'}\equiv \alpha/\beta + \beta/\alpha = 1$, that is

    $$
    U_{2m}(P, Q)\equiv PQ^{m-1}U_m(P^{'}, 1)\pmod{N}\tag{2.8}
    $$

    According to the Extended Lucas Theorem
    > If p is an odd prime, $p\nmid Q$ and the Legendre symbol $(\Delta/p) = \epsilon$, then

    $$
    \begin{aligned}
    U_{(p-\epsilon)m}(P, Q) &\equiv 0\pmod{p}\\
    V_{(p-\epsilon)m}(P, Q) &\equiv 2Q^{m(1-\epsilon)/2}\pmod{p}    
    \end{aligned}
    $$

* `Case 1`: Given that p is a factor of N, and p+1 is a smooth number

    $$
    p = \left(\prod_{i=1}^k{q_i^{\alpha_i}}\right)-1
    $$

    We have $p+1\mid R$. When $(Q, N)=1$ and $(\Delta/p) = -1$, we have $p\mid U_R(P, Q)$, i.e., $p\mid (U_R(P, Q), N)$

    To find $U_R(P, Q)$, `Guy` and `Conway` proposed the following formula

    $$
    \begin{aligned}
        U_{2n-1} &= U_n^2 - QU_n^2 - 1\\
        U_{2n} &= U_n(PU_n - 2QU_{n-1})\\
        U_{2n+1} &= PU_{2n} - QU_{2n-1}
    \end{aligned}
    $$

    However, the values from the above formula are too large and inconvenient for computation. We can consider the following approach

    If $p \mid U_R(P, 1)$, according to `Formula 2.3` we have $p\mid U_{2R}(P, Q)$, so according to `Formula 2.8` we have $p \mid U_R(P^{'}, 1)$. Setting $Q=1$, we get

    $$
    V_{(p-\epsilon)m}(P, 1) \equiv 2\pmod{p}
    $$

    That is, if $p\mid U_R(P, 1)$, then $p\mid(V_R(P, 1) -2)$.

    Case 1 can be summarized as:

    Let $R = r_1r_2r_3\cdots r_m$, and find $P_0$ such that $(P_0^2-4, N) = 1$. Define $V_n(P) = V_n(P, 1), U_n(P) = U_n(P, 1)$ and

    $$
    P_j \equiv V_{r_j}(P_{j-1})\pmod{N}(j = 1,2,3,\dots,m)
    $$

    According to `Formula 2.7`, we have

    $$
    P_m \equiv V_R(P_0)\pmod{N}\tag{3.1}
    $$

    To compute $V_r = V_r(P)$, we can use the following formula
    
    According to `Formula 2.2`, `Formula 2.3`, `Formula 2.4`, we have

    $$
    \begin{cases}
        V_{2f-1}&\equiv V_fV_{f-1}-P\\
        V_{2f}&\equiv V_f^2 - 2\\
        V_{2f+1}&\equiv PV_f^2-V_fV_{f-1}-P\pmod(N)
    \end{cases}
    $$

    Let

    $$
    r = \sum_{i=0}^t{b_t2^{t-i}}\ \ \ \ (b_i=0,1)
    $$
    
    $f_0=1, f_{k+1}=2f_k+b_{k+1}$, then $f_t=r$. Similarly $V_0(P) = 2, V_1(P) = P$, and the final formula is

    $$
    (V_{f_{k+1}}, V_{f_{k+1}-1}) = \begin{cases}
    (V_{2f_k}, V_{2f_k-1})\ \ \ \ if\ b_{k+1}=0\\
    (V_{2f_k+1}, V_{2f_k})\ \ \ \ if\ b_{k+1}=1
    \end{cases}
    $$

* `Case 2`: Given that p+1 is a smooth number

    $$
    p = s\left(\prod_{i=1}^k{q_i^{\alpha_i}}\right)-1
    $$

    When $s$ is a prime and $B_1<s\le B_2$, we have $p\mid(a_m^s-1, N)$. Define $s_j$ and $2d_j$
    
    $$
    2d_j = s_j+1-s_j
    $$
    
    If $(\Delta/p) = -1$ and $p\nmid P_m-2$, then according to `Formula 2.7` and `Formula 3.1` we have $p\mid(U_s(P_m), N)$.

    Let $U[n] \equiv U_n(P_m), V[n]\equiv V_n(P_m)\pmod{N}$. Compute $U[2d_j-1], U[2d_j], U[2d_j+1]$ via

    $$U[0] = 0, U[1] = 1, U[n+1] = P_mU[n] - U[n-1]$$

    Compute

    $$
    T[s_i] \equiv \Delta U_{s_i}(P_m) = \Delta U_{s_iR}(P_0)/U_R(P_0)\pmod{N}
    $$

    Via `Formula 2.6`, `Formula 2.7` and `Formula 3.1` we have

    $$
    \begin{cases}
        T[s_1]&\equiv P_mV[s_1]-2V[s_1-1]\\
        T[s_1-1]&\equiv 2V[s_1]-P_mV[s_1-1]\pmod{N}
    \end{cases}
    $$

    That is

    $$
    \begin{cases}
        T[s_{i+1}]&\equiv T[s_i]U[2d_i+1]-T[s_i-1]U[2d_i]\\
        T[s_{i+1}-1]&\equiv T[s_i]U[2d_i]-T[s_i-1]U[2d_i-1]\pmod{N}
    \end{cases}
    $$

    Compute $T[s_i], i=1,2,3\dots$, then compute

    $$
    H_t = (\prod_{i=0}^c{T[s_{i+t}], N})
    $$

    Where $t = 1, c+1, 2c+1, \dots, c[B_2/c]+1$. We have $p\mid H_i$ when $(\Delta/p)=-1$

* Python code implementation

    ```python
    def mlucas(v, a, n):
        """ Helper function for williams_pp1().  Multiplies along a Lucas sequence modulo n. """
        v1, v2 = v, (v**2 - 2) % n
        for bit in bin(a)[3:]: v1, v2 = ((v1**2 - 2) % n, (v1*v2 - v) % n) if bit == "0" else ((v1*v2 - v) % n, (v2**2 - 2) % n)
        return v1
    
    for v in count(1):
        for p in primegen():
            e = ilog(isqrt(n), p)
            if e == 0: break
            for _ in xrange(e): v = mlucas(v, p, n)
            g = gcd(v-2, n)
            if 1 < g < n: return g # g|n
            if g == n: break
    ```

### 2017 SECCON very smooth

This challenge provides an HTTPS encrypted traffic capture. First, we extract the certificates from it.

```shell
➜  2017_SECCON_verysmooth git:(master) binwalk -e s.pcap      

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
2292          0x8F4           Certificate in DER format (x509 v3), header length: 4, sequence length: 467
4038          0xFC6           Certificate in DER format (x509 v3), header length: 4, sequence length: 467
5541          0x15A5          Certificate in DER format (x509 v3), header length: 4, sequence length: 467

➜  2017_SECCON_verysmooth git:(master) ls
s.pcap  _s.pcap.extracted  very_smooth.zip
```

Here we examine the three certificates separately. All three have the same modulus; only one example is shown here.

```
➜  _s.pcap.extracted git:(master) openssl x509 -inform DER -in FC6.crt  -pubkey -text -modulus -noout 
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDVRqqCXPYd6Xdl9GT7/kiJrYvy
8lohddAsi28qwMXCe2cDWuwZKzdB3R9NEnUxsHqwEuuGJBwJwIFJnmnvWurHjcYj
DUddp+4X8C9jtvCaLTgd+baSjo2eB0f+uiSL/9/4nN+vR3FliRm2mByeFCjppTQl
yioxCqbXYIMxGO4NcQIDAQAB
-----END PUBLIC KEY-----
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 11640506567126718943 (0xa18b630c7b3099df)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=JP, ST=Kawasaki, O=SRL
        Validity
            Not Before: Oct  8 02:47:17 2017 GMT
            Not After : Oct  8 02:47:17 2018 GMT
        Subject: C=JP, ST=Kawasaki, O=SRL
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (1024 bit)
                Modulus:
                    00:d5:46:aa:82:5c:f6:1d:e9:77:65:f4:64:fb:fe:
                    48:89:ad:8b:f2:f2:5a:21:75:d0:2c:8b:6f:2a:c0:
                    c5:c2:7b:67:03:5a:ec:19:2b:37:41:dd:1f:4d:12:
                    75:31:b0:7a:b0:12:eb:86:24:1c:09:c0:81:49:9e:
                    69:ef:5a:ea:c7:8d:c6:23:0d:47:5d:a7:ee:17:f0:
                    2f:63:b6:f0:9a:2d:38:1d:f9:b6:92:8e:8d:9e:07:
                    47:fe:ba:24:8b:ff:df:f8:9c:df:af:47:71:65:89:
                    19:b6:98:1c:9e:14:28:e9:a5:34:25:ca:2a:31:0a:
                    a6:d7:60:83:31:18:ee:0d:71
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         78:92:11:fb:6c:e1:7a:f7:2a:33:b8:8b:08:a7:f7:5b:de:cf:
         62:0b:a0:ed:be:d0:69:88:38:93:94:9d:05:41:73:bd:7e:b3:
         32:ec:8e:10:bc:3a:62:b0:56:c7:c1:3f:60:66:a7:be:b9:46:
         f7:46:22:6a:f3:5a:25:d5:66:94:57:0e:fc:b5:16:33:05:1c:
         6f:f5:85:74:57:a4:a0:c6:ce:4f:fd:64:53:94:a9:83:b8:96:
         bf:5b:a7:ee:8b:1e:48:a7:d2:43:06:0e:4f:5a:86:62:69:05:
         e2:c0:bd:4e:89:c9:af:04:4a:77:a2:34:86:6a:b8:d2:3b:32:
         b7:39
Modulus=D546AA825CF61DE97765F464FBFE4889AD8BF2F25A2175D02C8B6F2AC0C5C27B67035AEC192B3741DD1F4D127531B07AB012EB86241C09C081499E69EF5AEAC78DC6230D475DA7EE17F02F63B6F09A2D381DF9B6928E8D9E0747FEBA248BFFDFF89CDFAF4771658919B6981C9E1428E9A53425CA2A310AA6D760833118EE0D71
```

We can see that the modulus is only 1024 bits. Moreover, based on the challenge name "very smooth", one of the factors should be smooth. Here we use primefac to try both Pollard's p − 1 and Williams's p + 1 algorithms, as follows:

```shell
➜  _s.pcap.extracted git:(master) python -m primefac -vs -m=p+1  149767527975084886970446073530848114556615616489502613024958495602726912268566044330103850191720149622479290535294679429142532379851252608925587476670908668848275349192719279981470382501117310509432417895412013324758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897

149767527975084886970446073530848114556615616489502613024958495602726912268566044330103850191720149622479290535294679429142532379851252608925587476670908668848275349192719279981470382501117310509432417895412013324758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897: p+1 11807485231629132025602991324007150366908229752508016230400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001 12684117323636134264468162714319298445454220244413621344524758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897
Z309  =  P155 x P155  =  11807485231629132025602991324007150366908229752508016230400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001 x 12684117323636134264468162714319298445454220244413621344524758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897
```

We can see that when using Williams's *p* + 1 algorithm, it was factored directly. Logically, this factor's p-1 seems smoother, but Pollard's p − 1 algorithm cannot factor it. Here we perform further testing:

```shell
➜  _s.pcap.extracted git:(master) python -m primefac -vs 1180748523162913202560299132400715036690822975250801623040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002

1180748523162913202560299132400715036690822975250801623040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002: 2 7 43 503 761429 5121103123294685745276806480148867612214394022184063853387799606010231770631857868979139305712805242051823263337587909550709296150544706624823
Z154  =  P1 x P1 x P2 x P3 x P6 x P142  =  2 x 7 x 43 x 503 x 761429 x 5121103123294685745276806480148867612214394022184063853387799606010231770631857868979139305712805242051823263337587909550709296150544706624823

➜  _s.pcap.extracted git:(master) python -m primefac -vs 1180748523162913202560299132400715036690822975250801623040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000 

1180748523162913202560299132400715036690822975250801623040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000: 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5 5
Z154  =  P1^185 x P1^62 x P1^97  =  2^185 x 3^62 x 5^97
```

As we can see, p-1 indeed has many small factors, but the count is too large, which causes exponential explosion during enumeration, and thus the factorization failed.

Then we construct the private key based on the factored numbers:

```python
from Crypto.PublicKey import RSA
import gmpy2


def main():
    n = 149767527975084886970446073530848114556615616489502613024958495602726912268566044330103850191720149622479290535294679429142532379851252608925587476670908668848275349192719279981470382501117310509432417895412013324758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897L
    p = 11807485231629132025602991324007150366908229752508016230400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001L
    q = 12684117323636134264468162714319298445454220244413621344524758865071052169170753552224766744798369054498758364258656141800253652826603727552918575175830897L
    e = 65537L
    priv = RSA.construct((n, e, long(gmpy2.invert(e, (p - 1) * (q - 1)))))
    open('private.pem', 'w').write(priv.exportKey('PEM'))


main()
```

Finally, import the private key into Wireshark to obtain the plaintext (Edit -> Preferences -> Protocols -> SSL -> RSA Key List).

```html
<html>
<head><title>Very smooth</title></head>
<body>
<h1>
Answer: One of these primes is very smooth.
</h1>
</body>
</html>
```

### Extensions

For more methods of factoring the modulus N, refer to https://en.wikipedia.org/wiki/Integer_factorization.

## Non-Coprime Moduli

### Attack Principle

When two public keys have non-coprime N values, we can obviously compute the greatest common divisor of these two numbers directly, and then directly obtain p and q, thereby obtaining the corresponding private keys.

### SCTF RSA2 

Here we use SCTF rsa2 as an example. Opening the pcap file directly, we find a bunch of messages containing N and e. Then we tested whether different N values are coprime. I tried the first two:

```python
import gmpy2
n1 = 20823369114556260762913588844471869725762985812215987993867783630051420241057912385055482788016327978468318067078233844052599750813155644341123314882762057524098732961382833215291266591824632392867716174967906544356144072051132659339140155889569810885013851467056048003672165059640408394953573072431523556848077958005971533618912219793914524077919058591586451716113637770245067687598931071827344740936982776112986104051191922613616045102859044234789636058568396611030966639561922036712001911238552391625658741659644888069244729729297927279384318252191421446283531524990762609975988147922688946591302181753813360518031
n2 = 19083821613736429958432024980074405375408953269276839696319265596855426189256865650651460460079819368923576109723079906759410116999053050999183058013281152153221170931725172009360565530214701693693990313074253430870625982998637645030077199119183041314493288940590060575521928665131467548955951797198132001987298869492894105525970519287000775477095816742582753228905458466705932162641076343490086247969277673809512472546919489077884464190676638450684714880196854445469562733561723325588433285405495368807600668761929378526978417102735864613562148766250350460118131749533517869691858933617013731291337496943174343464943
print gmpy2.gcd(n1, n2)
```

The result shows that they are indeed not coprime.

```shell
➜  scaf-rsa2 git:(master) ✗ python exp.py
122281872221091773923842091258531471948886120336284482555605167683829690073110898673260712865021244633908982705290201598907538975692920305239961645109897081011524485706755794882283892011824006117276162119331970728229108731696164377808170099285659797066904706924125871571157672409051718751812724929680249712137
```

Then we can decrypt directly. Here we use the first pair of public key and ciphertext. The code is as follows:

```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5, PKCS1_OAEP
import gmpy2
from base64 import b64decode
n1 = 20823369114556260762913588844471869725762985812215987993867783630051420241057912385055482788016327978468318067078233844052599750813155644341123314882762057524098732961382833215291266591824632392867716174967906544356144072051132659339140155889569810885013851467056048003672165059640408394953573072431523556848077958005971533618912219793914524077919058591586451716113637770245067687598931071827344740936982776112986104051191922613616045102859044234789636058568396611030966639561922036712001911238552391625658741659644888069244729729297927279384318252191421446283531524990762609975988147922688946591302181753813360518031
n2 = 19083821613736429958432024980074405375408953269276839696319265596855426189256865650651460460079819368923576109723079906759410116999053050999183058013281152153221170931725172009360565530214701693693990313074253430870625982998637645030077199119183041314493288940590060575521928665131467548955951797198132001987298869492894105525970519287000775477095816742582753228905458466705932162641076343490086247969277673809512472546919489077884464190676638450684714880196854445469562733561723325588433285405495368807600668761929378526978417102735864613562148766250350460118131749533517869691858933617013731291337496943174343464943
p1 = gmpy2.gcd(n1, n2)
q1 = n1 / p1
e = 65537
phin = (p1 - 1) * (q1 - 1)
d = gmpy2.invert(e, phin)
cipher = 0x68d5702b70d18238f9d4a3ac355b2a8934328250efd4efda39a4d750d80818e6fe228ba3af471b27cc529a4b0bef70a2598b80dd251b15952e6a6849d366633ed7bb716ed63c6febd4cd0621b0c4ebfe5235de03d4ee016448de1afbbe61144845b580eed8be8127a8d92b37f9ef670b3cdd5af613c76f58ca1a9f6f03f1bc11addba30b61bb191efe0015e971b8f78375faa257a60b355050f6435d94b49eab07075f40cb20bb8723d02f5998d5538e8dafc80cc58643c91f6c0868a7a7bf3bf6a9b4b6e79e0a80e89d430f0c049e1db4883c50db066a709b89d74038c34764aac286c36907b392bc299ab8288f9d7e372868954a92cdbf634678f7294096c7
plain = gmpy2.powmod(cipher, d, n1)
plain = hex(plain)[2:]
if len(plain) % 2 != 0:
    plain = '0' + plain
print plain.decode('hex')
```

The final decryption result:

```shell
➜  scaf-rsa2 git:(master) ✗ python exp.py       
sH1R3_PRlME_1N_rsA_iS_4ulnEra5le
```

Decompress the archive to get the flag.

## Common Modulus Attack

### Attack Conditions

When two users use the same modulus N with different private keys to encrypt the same plaintext message, a common modulus attack exists.

### Attack Principle

Let the public keys of two users be $e_1$ and $e_2$ respectively, and the two are coprime. The plaintext message is $m$, and the ciphertexts are:

$$
c_1 = m^{e_1}\bmod N \\
c_2 = m^{e_2}\bmod N
$$

When the attacker intercepts $c_1$ and $c_2$, they can recover the plaintext. Using the Extended Euclidean Algorithm, find two integers $r$ and $s$ such that $re_1+se_2=1\bmod n$. From this we obtain:

$$
\begin{align*}
c_{1}^{r}c_{2}^{s} &\equiv m^{re_1}m^{se_2}\bmod n\\
&\equiv m^{(re_1+se_2)} \bmod n\\
&\equiv m\bmod n
\end{align*}
$$

### XMan Season 1 Summer Camp Class Exercise

Problem description:

```
{6266565720726907265997241358331585417095726146341989755538017122981360742813498401533594757088796536341941659691259323065631249,773}

{6266565720726907265997241358331585417095726146341989755538017122981360742813498401533594757088796536341941659691259323065631249,839}

message1=3453520592723443935451151545245025864232388871721682326408915024349804062041976702364728660682912396903968193981131553111537349

message2=5672818026816293344070119332536629619457163570036305296869053532293105379690793386019065754465292867769521736414170803238309535
```

> Source: XMan Season 1 Summer Camp Class Exercise 

We can see that the two public keys have the same N, and their e values are coprime. Let's write a script to run it:

```python
import gmpy2
n = 6266565720726907265997241358331585417095726146341989755538017122981360742813498401533594757088796536341941659691259323065631249
e1 = 773

e2 = 839

message1 = 3453520592723443935451151545245025864232388871721682326408915024349804062041976702364728660682912396903968193981131553111537349

message2 = 5672818026816293344070119332536629619457163570036305296869053532293105379690793386019065754465292867769521736414170803238309535
# s & t
gcd, s, t = gmpy2.gcdext(e1, e2)
if s < 0:
    s = -s
    message1 = gmpy2.invert(message1, n)
if t < 0:
    t = -t
    message2 = gmpy2.invert(message2, n)
plain = gmpy2.powmod(message1, s, n) * gmpy2.powmod(message2, t, n) % n
print plain
```

We get

```shell
➜  Xman-1-class-exercise git:(master) ✗ python exp.py
1021089710312311910410111011910111610410511010710511610511511211111511510598108101125
```

At this point, we need to consider how the plaintext was originally converted to this number. Generally, it is hexadecimal conversion, ASCII character conversion, or Base64 decoding. This should be ASCII character conversion, and we use the following code to obtain the flag:

```python
i = 0
flag = ""
plain = str(plain)
while i < len(plain):
    if plain[i] == '1':
        flag += chr(int(plain[i:i + 3]))
        i += 3
    else:
        flag += chr(int(plain[i:i + 2]))
        i += 2
print flag
```

The reason we use 1 to determine whether the length is three digits is that flags are generally printable characters, and numbers starting with 1 that are only 1 or 2 digits long are generally non-printable characters.

flag

```shell
➜  Xman-1-class-exercise git:(master) ✗ python exp.py
flag{whenwethinkitispossible}
```

## Challenges
