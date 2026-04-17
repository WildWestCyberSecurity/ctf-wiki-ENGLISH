# RSA Chosen Plaintext/Ciphertext Attack

## Chosen Plaintext Attack

Here is an example. Suppose we have an encryption oracle, but we don't know n and e, then:

1. We can obtain n through the encryption oracle.
2. When e is relatively small ($e<2^{64}$), we can use *Pollard's kangaroo algorithm* to obtain e. This point is fairly obvious.

We can encrypt 2, 4, 8, 16. Then we can know

$c_2=2^{e} \bmod n$

$c_4=4^{e} \bmod n$

$c_8=8^{e} \bmod n$

Then

$c_2^2 \equiv c_4 \bmod n$

$c_2^3 \equiv c_8 \bmod n$

Therefore

$c_2^2-c_4=kn$

$c_2^3-c_8=tn$

We can compute the greatest common divisor of kn and tn, which with high probability is n. We can also construct more examples to determine n with greater certainty.

## Arbitrary Ciphertext Decryption

Suppose Alice creates the ciphertext $C = P^e \bmod n$ and sends C to Bob. Also suppose we want to decrypt any ciphertext encrypted by Alice, not just C. Then we can intercept C and use the following steps to find P:

1. Choose any $X\in Z_n^{*}$, i.e., X is coprime to N
2. Compute $Y=C \times X^e \bmod n$ 
3. Since we can perform a chosen ciphertext attack, we obtain the decryption result of Y: $Z=Y^d$
4. Then, since $Z=Y^d=(C \times X^e)^d=C^d X=P^{ed} X= P X\bmod n$, and since X is coprime to N, we can easily compute the corresponding inverse, and thus obtain P

## RSA parity oracle

Suppose there exists an Oracle that decrypts a given ciphertext, checks the parity of the decrypted plaintext, and returns a corresponding value based on the parity, e.g., 1 for odd and 0 for even. Then given an encrypted ciphertext, we only need log(N) queries to recover the plaintext message corresponding to this ciphertext.

### Principle

Assume

$C=P^e \bmod N$

For the first query, we can send to the server

$C*2^e=(2P)^e \bmod N$

The server will compute

$2P \bmod N$

Here

- 2P is even, and its powers are also even.
- N is odd, because it is the product of two large primes.

Then

- If the server returns odd, i.e., $2P \bmod N$ is odd, it means 2P is greater than N and an odd number of N's were subtracted. Since $2P<2N$, exactly one N was subtracted, i.e., $\frac{N}{2} \leq P < N$. We can also consider floor division.
- If the server returns even, it means 2P is less than N, i.e., $0\leq P < \frac{N}{2}$. We can also use floor division.

Here we use mathematical induction, assuming that at the i-th step, $\frac{xN}{2^{i}} \leq P < \frac{xN+N}{2^{i}}$

Furthermore, at the (i+1)-th step, we can send

$C*2^{(i+1)e}$

The server will compute

$2^{i+1}P \bmod N=2^{i+1}P-kN$

$0 \leq 2^{i+1}P-kN<N$ 

$\frac{kN}{2^{i+1}} \leq P < \frac{kN+N}{2^{i+1}}$

Based on the result from the i-th step

$\frac{2xN}{2^{i+1}} \leq P < \frac{2xN+2N}{2^{i+1}}$

Then

- If the server returns odd, then k must be an odd number, k=2y+1, so $\frac{2yN+N}{2^{i+1}} \leq P < \frac{2yN+2N}{2^{i+1}}$. At the same time, since P must exist, the range obtained at step i+1 and the range obtained at step i must have an intersection. Therefore y must be equal to x.
- If the server returns even, then k must be an even number, k=2y, and here y must also be equal to x, so $\frac{2xN}{2^{i+1}} \leq P < \frac{2xN+N}{2^{i+1}}$

Furthermore, we can summarize inductively as follows

```c
lb = 0
ub = N
if server returns 1
	lb = (lb+ub)/2
else:
	ub = (lb+ub)/2
```

Although this uses integer division (floor division), it doesn't matter since we already analyzed this issue at the beginning.

### 2018 Google CTF Perfect Secrecy

Here we use the 2018 Google CTF challenge as an example for analysis

```python
#!/usr/bin/env python3
import sys
import random

from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend


def ReadPrivateKey(filename):
  return serialization.load_pem_private_key(
      open(filename, 'rb').read(), password=None, backend=default_backend())


def RsaDecrypt(private_key, ciphertext):
  assert (len(ciphertext) <=
          (private_key.public_key().key_size // 8)), 'Ciphertext too large'
  return pow(
      int.from_bytes(ciphertext, 'big'),
      private_key.private_numbers().d,
      private_key.public_key().public_numbers().n)


def Challenge(private_key, reader, writer):
  try:
    m0 = reader.read(1)
    m1 = reader.read(1)
    ciphertext = reader.read(private_key.public_key().key_size // 8)
    dice = RsaDecrypt(private_key, ciphertext)
    for rounds in range(100):
      p = [m0, m1][dice & 1]
      k = random.randint(0, 2)
      c = (ord(p) + k) % 2
      writer.write(bytes((c,)))
    writer.flush()
    return 0

  except Exception as e:
    return 1


def main():
  private_key = ReadPrivateKey(sys.argv[1])
  return Challenge(private_key, sys.stdin.buffer, sys.stdout.buffer)


if __name__ == '__main__':
  sys.exit(main())
```

We can observe that

- We can send two numbers to the server, and the server will decide which one to use based on the decrypted ciphertext content.
- The server uses `random.randint(0, 2)` to generate a random number and outputs a related random 01 byte c.

At first glance, it seems completely random. However, upon closer inspection, `random.randint(0, 2)` generates random numbers inclusive of both boundaries, so the probability of generating an even number is greater than the probability of generating an odd number. Therefore, the probability that c has the same parity as p is 2/3. Thus, by setting m0 and m1, we can determine whether the last bit of the decrypted ciphertext is 0 or 1. This is essentially an RSA parity oracle.

The exploit is as follows

```python
import gmpy2
from pwn import *
encflag = open('./flag.txt').read()
encflag = encflag.encode('hex')
encflag = int(encflag, 16)
#context.log_level = 'debug'
m = ['\x00', '\x07']
n = 0xDA53A899D5573091AF6CC9C9A9FC315F76402C8970BBB1986BFE8E29CED12D0ADF61B21D6C281CCBF2EFED79AA7DD23A2776B03503B1AF354E35BF58C91DB7D7C62F6B92C918C90B68859C77CAE9FDB314F82490A0D6B50C5DC85F5C92A6FDF19716AC8451EFE8BBDF488AE098A7C76ADD2599F2CA642073AFA20D143AF403D1
e = 65537
flag = ""



def guessvalue(cnt):
    if cnt[0] > cnt[1]:
        return 0
    return 1


i = 0
while True:
    cnt = dict()
    cnt[0] = cnt[1] = 0
    p = remote('perfect-secrecy.ctfcompetition.com', 1337)
    p.send(m[0])
    p.send(m[1])
    tmp = pow(2, i)
    two_inv = gmpy2.invert(tmp, n)
    two_cipher = gmpy2.powmod(two_inv, e, n)
    tmp = encflag * two_cipher % n
    tmp = hex(tmp)[2:].strip('L')
    tmp = '0' * (256 - len(tmp)) + tmp
    tmp = tmp.decode('hex')
    assert (len(tmp) == 128)
    p.send(tmp)
    #print tmp
    data = ""
    while (len(data) != 100):
        data += p.recv()
    for c in data:
        cnt[u8(c)] += 1
    p.close()
    flag = str(guessvalue(cnt)) + flag
    print i, flag
    i += 1
```

The result is as follows

```shell
6533021797450432625003726192285181680054061843303961161444459679874621880787893445342698029728203298974356255732086344166897556918532195998159983477294838449903429031335408290610431938507208444225296242342845578895553611385588996615744823221415296689514934439749745119968629875229882861818946483594948270 6533021797450432625003726192285181680054061843303961161444459679874621880787893445342698029728203298974356255732086344166897556918532195998159983477294838449903429031335408290610431938507208444225296242342845578895553611385588996615744823221415296689514934439749745119968629875229882861818946483594948270
```

After decoding, we can obtain the flag

```shell
CTF{h3ll0__17_5_m3_1_w45_w0nd3r1n6_1f_4f73r_4ll_7h353_y34r5_y0u_d_l1k3_70_m337}
```

### Challenges

- 2016 Plaid CTF rabit
- 2016 sharif CTF lsb-oracle-150
- 2018 Backdoor CTF  BIT-LEAKER
- 2018 XMAN Selection Contest baby RSA

## RSA Byte Oracle

Suppose there exists an Oracle that decrypts a given ciphertext and reveals the last byte of the plaintext. Then given an encrypted ciphertext, we only need $\log_{256}n$ queries to recover the plaintext message corresponding to this ciphertext.

### Principle

This is actually an extension of the RSA parity Oracle. Since the last byte can be leaked, the number of queries needed to recover the plaintext corresponding to the ciphertext should be reduced.

Assume

$C=P^e \bmod N$

For the first query, we can send to the server

$C*256^e=(256P)^e \bmod N$

The server will compute

$256P \bmod N$

Here

- 256P is even.
- N is odd, because it is the product of two large primes.

Since P is generally less than N, we have $256P \bmod N=256P-kn, k<256$. Moreover, for two different $k_1,k_2$, we have

$256P-k_1n \not\equiv 256P-k_2n \bmod 256$

We can prove the above inequality by contradiction. At the same time, the last byte of $256P-kn$ is actually obtained from $-kn$ modulo 256. So in fact, we can first enumerate the last byte for cases 0~255, constructing a mapping table (map) between k and the last byte.

When the server returns the last byte b, we can look up k from the mapping table constructed above, meaning k copies of N were subtracted, i.e., $kN \leq 256 P \leq (k+1)N$.

After that, we use mathematical induction to obtain the range of P, assuming that at the i-th step, $\frac{xN}{256^{i}} \leq P < \frac{xN+N}{256^{i}}$

Furthermore, at the (i+1)-th step, we can send

$C*256^{(i+1)e}$

The server will compute

$256^{i+1}P \bmod N=256^{i+1}P-kN$

$0 \leq 256^{i+1}P-kN<N$ 

$\frac{kN}{256^{i+1}} \leq P < \frac{kN+N}{256^{i+1}}$

Based on the result from the i-th step

$\frac{256xN}{256^{i+1}} \leq P < \frac{256xN+256N}{256^{i+1}}$

Here we can assume $k=256y+t$, where t is what we can obtain through the mapping table.

 $\frac{256yN+tN}{256^{i+1}} \leq P < \frac{256yN+(t+1)N}{256^{i+1}}$

At the same time, since P must exist, the range obtained at step i+1 and the range obtained at step i must have an intersection.

Therefore y must be equal to x.

Furthermore, we can summarize inductively as follows. Initially:

```
lb = 0
ub = N
```

Suppose the server returns b, then

```c
k = mab[b]
interval = (ub-lb)/256
lb = lb + interval * k
ub = lb + interval
```

### 2018 HITCON lost key

This is a comprehensive challenge. First, n is not given, so we can use a chosen plaintext attack to obtain n. Of course, we can also further obtain e. The final exploit code is as follows

```python
from pwn import *
import gmpy2
from fractions import Fraction
p = process('./rsa.py')
#p = remote('18.179.251.168', 21700)
#context.log_level = 'debug'
p.recvuntil('Here is the flag!\n')
flagcipher = int(p.recvuntil('\n', drop=True), 16)


def long_to_hex(n):
    s = hex(n)[2:].rstrip('L')
    if len(s) % 2: s = '0' + s
    return s


def send(ch, num):
    p.sendlineafter('cmd: ', ch)
    p.sendlineafter('input: ', long_to_hex(num))
    data = p.recvuntil('\n')
    return int(data, 16)


if __name__ == "__main__":
    # get n
    cipher2 = send('A', 2)
    cipher4 = send('A', 4)
    nset = []
    nset.append(cipher2 * cipher2 - cipher4)

    cipher3 = send('A', 3)
    cipher9 = send('A', 9)
    nset.append(cipher3 * cipher3 - cipher9)
    cipher5 = send('A', 5)
    cipher25 = send('A', 25)
    nset.append(cipher5 * cipher5 - cipher25)
    n = nset[0]
    for item in nset:
        n = gmpy2.gcd(item, n)

    # get map between k and return byte
    submap = {}
    for i in range(0, 256):
        submap[-n * i % 256] = i

    # get cipher256
    cipher256 = send('A', 256)

    back = flagcipher

    L = Fraction(0, 1)
    R = Fraction(1, 1)
    for i in range(128):
        print i
        flagcipher = flagcipher * cipher256 % n
        b = send('B', flagcipher)
        k = submap[b]
        L, R = L + (R - L) * Fraction(k, 256
                                     ), L + (R - L) * Fraction(k + 1, 256)
    low = int(L * n)
    print long_to_hex(low - low % 256 + send('B', back)).decode('hex')
```

## RSA parity oracle variant
### Principle
If the oracle's parameters change after a certain time or number of operation cycles, or if the network is unstable causing session disconnection or reset, the binary search method is no longer applicable. To reduce errors, bit-by-bit recovery should be considered.
To recover the second least significant bit, consider

$$\{(c(2^{-1*e_1}\mod N_1))^{d_1}\mod N_1\}\pmod2\equiv m*2^{-1}$$

$$
\begin{aligned}
&m*(2^{-1}\mod N_1)\mod2\\
&=(\displaystyle\sum_{i=0}^{logm-1}a_i*2^i)*2^{-1}\mod2\\
&=[2(\displaystyle\sum_{i=1}^{logm-1}a_i*2^{i-1})+a_0*2^0]*2^{-1}\mod 2\\
&=\displaystyle\sum_{i=1}^{logm-1}a_i*2^{i-1}+a_0*2^0*2^{-1}\mod2\\
&\equiv a_1+a_0*2^0*2^{-1}\equiv y\pmod2
\end{aligned}
$$

$$
y-(a_0*2^0)*2^{-1}=(m*2^{-1}\mod2)-(a_0*2^0)*2^{-1}\equiv a_1\pmod2
$$

Similarly

$$\{(c(2^{-2*e_2}\mod N_2))^{d_2}\mod N_2\}\pmod2\equiv m*2^{-2}$$

$$
\begin{aligned}
&m*(2^{-2}\mod N_2)\mod2\\
&=(\displaystyle\sum_{i=0}^{logm-1}a_i*2^i)*2^{-2}\mod2\\
&=[2^2(\displaystyle\sum_{i=2}^{logm-1}a_i*2^{i-2})+a_1*2^1+a_0*2^0]*2^{-2}\mod 2\\
&=\displaystyle\sum_{i=2}^{logm-1}a_i*2^{i-1}+(a_1*2^1+a_0*2^0)*2^{-2}\mod2\\
&\equiv a_2+(a_1*2^1+a_0*2^0)*2^{-2}\equiv y\pmod2
\end{aligned}
$$

$$
\begin{aligned}
    &y-(a_1*2^1+a_0*2^0)*2^{-2}\\
    &=(m*2^{-2}\mod2)-(a_1*2^1+a_0*2^0)*2^{-2}\equiv a_2\pmod2
\end{aligned}
$$

We can use the previous i-1 bits together with the oracle's result to obtain the i-th bit. Note that $2^{-1}$ here is the modular inverse of $2^1$ modulo $N_1$. So for the remaining bits, we have

$$
\begin{aligned}
    &\{(c(2^{-i*e_i}\mod N_i))^{d_i}\mod N_i\}\pmod2\equiv m*2^{-i}\\
    &a_i\equiv (m*2^{-i}\mod2) -\sum_{j=0}^{i-1}a_j*2^j\pmod2,i=1,2,...,logm-1
\end{aligned}
$$

Where $2^{-i}$ is the modular inverse of $2^i$ modulo $N_i$.

This allows us to progressively recover all the bits of the original plaintext. The time complexity of this approach is $O(logm)$.

exp:
```python
from Crypto.Util.number import *
mm = bytes_to_long(b'12345678')
l = len(bin(mm)) - 2

def genkey():
    while 1:
        p = getPrime(128)
        q = getPrime(128)
        e = getPrime(32)
        n = p * q
        phi = (p - 1) * (q - 1)
        if GCD(e, phi) > 1:
            continue
        d = inverse(e, phi)
        return e, d, n

e, d, n = genkey()
cc = pow(mm, e, n)
f = str(pow(cc, d, n) % 2)

for i in range(1, l):
    e, d, n = genkey()
    cc = pow(mm, e, n)
    ss = inverse(2**i, n)
    cs = (cc * pow(ss, e, n)) % n
    lb = pow(cs, d, n) % 2
    bb = (lb - (int(f, 2) * ss % n)) % 2
    f = str(bb) + f
    assert(((mm >> i) % 2) == bb)
print(long_to_bytes(int(f, 2)))
```

## References

- https://crypto.stackexchange.com/questions/11053/rsa-least-significant-bit-oracle-attack
- https://pastebin.com/KnEUSMxp
- https://github.com/ashutosh1206/Crypton
