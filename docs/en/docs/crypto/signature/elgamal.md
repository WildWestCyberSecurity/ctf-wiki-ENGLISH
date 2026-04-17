# ElGamal

The RSA digital signature scheme is almost entirely consistent with its encryption scheme, only using the private key for signing. However, for ElGamal, the signature scheme differs significantly from its corresponding encryption scheme.

## Basic Principles

### Key Generation

The basic steps are as follows:

1. Select a sufficiently large prime p (with no fewer than 160 decimal digits), so that solving the discrete logarithm problem over $Z_p$ is difficult.
2. Select a generator g of $Z_p^*$.
3. Randomly select an integer d, $0\leq d \leq p-2$, and compute $g^d \equiv y \bmod p$.

The private key is {d}, and the public key is {p,g,y}.

### Signing

A selects a random number $k \in Z_{p-1}$, where $gcd(k,p-1)=1$, and signs the message:


$$
sig_d(m,k)=(r,s)
$$


where $r \equiv g^k \bmod p$ and $s \equiv (m-dr)k^{-1} \bmod p-1$.

### Verification

If $g^m \equiv y^rr^s \bmod p$, then the verification succeeds; otherwise it fails. The principle behind successful verification is as follows. First, we have

$$
y^rr^s \equiv g^{dr}g^{ks} \equiv g^{dr+ks}
$$

Since

$$
s \equiv (m-dr)k^{-1} \bmod p-1
$$

Therefore

$$
ks \equiv m-dr \bmod p-1
$$

Furthermore

$$
ks+dr=a*(p-1)+m
$$

So

$$
g^{ks+dr}=g^{a*(p-1)+m}=(g^{p-1})^a*g^m
$$

Therefore, by Fermat's theorem, we get

$$
g^{ks+dr} \equiv g^m \bmod p
$$

## Common Attacks

### Complete Decryption Attack

#### Attack Conditions

- p is too small or has no large prime factors

If $p$ is too small, we can directly use the baby-step giant-step algorithm to factorize it, or if it has no large prime factors, we can use the $Pohlig\: Hellman$ algorithm to compute the discrete logarithm and thus recover the private key.

- Random number k reuse

If the signer reuses the random number k, the attacker can easily compute the private key. The specific principle is as follows:

Assume there are currently two signatures both generated using the same random number. Then we have

$$
r \equiv g^k \bmod p \\\\ s _1\equiv (m_1-dr)k^{-1} \bmod p-1\\\\ r \equiv g^k \bmod p \\\\ s_2 \equiv (m_2-dr)k^{-1} \bmod p-1
$$

Furthermore,

$$
s_1k \equiv m_1-dr \bmod p-1 \\\\ s_2k \equiv m_2-dr \bmod p-1
$$

Subtracting the two equations:

$$
k(s_1-s_2) \equiv m_1-m_2 \bmod p-1
$$

Here, $s_1,s_2,m_1,m_2,p-1$ are all known, so we can easily compute k. Of course, if $gcd(s_1-s_2,p-1)!=1$, there may be multiple solutions, in which case we just need to try a few more. Then, we can derive the private key d from the computation method of s, as follows:

$$
d \equiv \frac{m-ks}{r}
$$

#### Challenge

2016 LCTF Crypto 450

### Universal Signature Forgery

#### Attack Conditions

The attack is valid when the message $m$ is not hashed, or when there is no specified message format.

#### Principle

After the attacker learns someone's (Alice's) public key, they can forge Alice's signature. The specific principle is as follows:

Here we assume Alice's public key is {p,g,y}. The attacker can forge as follows:

1. Choose integers $i$, $j$, where $gcd(j,p-1)=1$

2. Compute the signature: $r \equiv g^iy^j \bmod p$, $s\equiv -rj^{-1} \bmod p-1$

3. Compute the message: $m\equiv si \bmod p-1$

The generated signature and message will pass verification normally. The specific derivation is as follows:

$y^rr^s \equiv g^{dr}g^{is}y^{js} \equiv g^{dr}g^{djs}g^{is} \equiv g^{dr+s(i+dj)} \equiv g^{dr} g^{-rj^{-1}(i+dj)} \equiv g^{dr-dr-rij^{-1}} \equiv g^{si} \bmod p$

And due to the construction of message m, we have

$$
g^{si} \equiv g^m \bmod p-1
$$

Note that while the attacker can forge messages that pass signature verification, they cannot forge messages in a specified format. Moreover, once the message is hashed, this attack is no longer feasible.

### Known Signature Forgery

#### Attack Conditions

Assume the attacker knows that $(r, s)$ is the signature of message $M$. The attacker can use it to forge signatures for other messages.

#### Principle

1. Choose integers $h, i, j \in[0, p-2]$ satisfying $\operatorname{gcd}(h r-j s, \varphi(p))=1$
2. Compute the following:
   $\begin{array}{l}
   r^{\prime}=r^{h} \alpha^{i} y_{A}^{j} \bmod p \\
   s^{\prime}=\operatorname{sr}(h r-j s)^{-1} \bmod \varphi(p) \\
   m^{\prime}=r^{\prime}(h m+i s)(h r-j s)^{-1} \bmod \varphi(p)
   \end{array}$

We can obtain $(r',s')$ as a valid signature for $m'$.

The proof is as follows:

Given that Alice's signature $(\gamma,\delta)$ for message $x$ satisfies $\beta^{\gamma} \gamma^{\delta} \equiv \alpha^{x}(\bmod p)$, our goal is to construct $\left(x^{\prime}, \lambda, \mu\right)$ satisfying


$$
\beta^{\lambda} \lambda^{\mu} \equiv \alpha^{x'}(\bmod p)
$$




First, we express $\lambda$ in terms of three known bases $\alpha, \beta, \gamma$: $\lambda=\alpha^{i} \beta^{j} \gamma^{h} \bmod p$. From the conditions, we get


$$
\beta^{\gamma} \gamma^{\delta} \equiv \alpha^{x}(\bmod p) \Leftrightarrow \gamma=\left(\beta^{-\gamma} \alpha^{x}\right)^{\delta-1} \bmod p
$$

Then we can get


$$
\lambda=\alpha^{i+x \delta^{-1} h} \beta^{j-\gamma \delta^{-1} h} \bmod p
$$


We substitute the expression for $\lambda$ into the first equation:


$$
\begin{aligned}& \beta^{\lambda}\left(\alpha^{i+x \delta^{-1} h} \beta^{j-\gamma \delta^{-1} h}\right)^{\mu} \equiv \alpha^{x^{\prime}}(\bmod p) \\\Leftrightarrow & \beta^{\lambda+\left(j-\gamma \delta^{-1} h\right) \mu} \equiv \alpha^{x^{\prime}-\left(i+x \delta^{-1} h\right) \mu}(\bmod p)\end{aligned}
$$


We set both exponents to $0$, i.e.,


$$
\left\{\begin{matrix}\lambda+\left(j-\gamma \delta^{-1} h\right) \mu \equiv 0 \bmod p-1 \\ x^{\prime}-\left(i+x \delta^{-1} h\right) \mu \equiv 0 \bmod p-1 \end{matrix}\right.
$$


We can get


$$
\mu=\delta \lambda(h \gamma-j \delta)^{-1} \quad(\bmod p-1)  \\
x^{\prime}=\lambda(h x+i \delta)(h \gamma-j \delta)^{-1}(\bmod p-1)
$$


where


$$
\lambda=\alpha^{i} \beta^{j} \gamma^{h} \bmod p
$$


Therefore we obtain $(\lambda, \mu)$ as a valid signature for $x'$.

Additionally, we can also construct $m'$ using CRT. The principle is as follows:

1. $u=m^{\prime} m^{-1} \bmod \varphi(p), \quad s^{\prime}=s u \bmod \varphi(p)$
2. Then compute $r^{\prime}, \quad r^{\prime} \equiv r u \bmod \varphi(p), r^{\prime} \equiv r \bmod p$

Obviously, CRT can be used to solve for $r'$. Note that $y_{A}^{r'} r'^{s^{\prime}}=y_{A}^{ru} r^{s u}=\left(y_{A}^{r} r^{s}\right)^{u}=\alpha^{m u} \equiv \alpha^{m} \bmod p$ 

Therefore $(r',s')$ is a valid signature for message $m'$.

Countermeasure: When verifying signatures, check that $r < p$.

### Chosen Signature Forgery

#### Attack Conditions

If we can choose messages to be signed and obtain the signatures, then we can forge signatures for new messages that we cannot choose to sign.

#### Principle

We know that the final verification process is as follows:

 $g^m \equiv y^rr^s \bmod p$ 

So as long as we choose a message m that is congruent modulo p-1 to the message $m'$ we want to forge, and then use the signature of message m, we can bypass the verification.

#### Challenge

Here we use the 2017 National Competition (CISCN) mailbox challenge as an example. **It is available for replay on i春秋 (iChunQiu)**.

First, let's analyze the program. We first need to perform a proof of work:


```python
	proof = b64.b64encode(os.urandom(12))
	req.sendall(
        "Please provide your proof of work, a sha1 sum ending in 16 bit's set to 0, it must be of length %d bytes, starting with %s\n" % (
        len(proof) + 5, proof))

    test = req.recv(21)
    ha = hashlib.sha1()
    ha.update(test)

    if (test[0:16] != proof or ord(ha.digest()[-1]) != 0 or ord(ha.digest()[-2]) != 0): # or ord(ha.digest()[-3]) != 0 or ord(ha.digest()[-4]) != 0):
        req.sendall("Check failed")
        req.close()
        return 
```
We need to generate a string starting with `proof` whose length equals the length of proof plus 5, and whose SHA1 value ends with 16 bits of zeros.

Here we directly use the following method to bypass this:

```python
def f(x):
    return sha1(prefix + x).digest()[-2:] == '\0\0'


sh = remote('106.75.66.195', 40001)
# bypass proof
sh.recvuntil('starting with ')
prefix = sh.recvuntil('\n', drop=True)
print string.ascii_letters
s = util.iters.mbruteforce(f, string.ascii_letters + string.digits, 5, 'fixed')
test = prefix + s
sh.sendline(test)
```

Here we use `util.iters.mbruteforce` from pwntools, which is a multi-threaded brute-force function using a given character set and specified length. The first parameter is the brute-force function (here SHA1), the second parameter is the character set, the third parameter is the number of bytes, and the fourth parameter means we only try permutations of exactly the specified byte count (i.e., the length is fixed). For more details, please refer to the pwntools documentation.

After bypassing, we continue analyzing the program. A quick look at the `generate_keys` function shows that it is the ElGamal public key generation process. Then looking at the `verify` function, it is the signature verification process.

Continuing the analysis:

```python
            if len(msg) > MSGLENGTH:
                req.sendall("what r u do'in?")
                req.close()
                return
            if msg[:4] == "test":
                r, s = sign(digitalize(msg), sk, pk, p, g)
                req.sendall("Your signature is" + repr((hex(r), hex(s))) + "\n")
            else:
                if msg == "Th3_bery_un1que1i_ChArmIng_G3nji" + test:
                    req.sendall("Signature:")
                    sig = self.rfile.readline().strip()
                    if len(sig) > MSGLENGTH:
                        req.sendall("what r u do'in?")
                        req.close()
                        return
                    sig_rs = sig.split(",")
                    if len(sig_rs) < 2:
                        req.sendall("yo what?")
                        req.close()
                        return
                    # print "Got sig", sig_rs
                    if verify(digitalize(msg), int(sig_rs[0]), int(sig_rs[1]), pk, p, g):
                        req.sendall("Login Success.\nDr. Ziegler has a message for you: " + FLAG)
                        print "shipped flag"
                        req.close()
                        return
                    else:
                        req.sendall("You are not the Genji I knew!\n")
```

From these three if conditions, we can determine:

- Our message length cannot exceed MSGLENGTH, which is 40000.
- We can sign messages that start with "test".
- We need to make a message starting with `Th3_bery_un1que1i_ChArmIng_G3nji` and ending with the test string from our proof-of-work bypass pass the signature verification, where we can provide the signature values ourselves.

At this point, the analysis reveals that we are performing a chosen signature forgery. Here we naturally want to make full use of the second if condition. As long as the message we input starts with 'test' and is congruent modulo p-1 to the fixed message starting with `Th3_bery_un1que1i_ChArmIng_G3nji`, we can pass the verification.

So how do we construct it? Since the message length can be sufficiently long, we can left-shift the hexadecimal value corresponding to 'test' to get a number a larger than p-1, then take a modulo p-1, and subtract the remainder from a, so that a is now congruent to 0 modulo p-1. Then we add the value of the fixed message starting with `Th3_bery_un1que1i_ChArmIng_G3nji`, achieving congruence modulo p-1.

The specific construction is as follows:

```python
# construct the message begins with 'test'
target = "Th3_bery_un1que1i_ChArmIng_G3nji" + test
part1 = (digitalize('test' + os.urandom(51)) << 512) // (p - 1) * (p - 1)
victim = part1 + digitalize(target)
while 1:
    tmp = hex(victim)[2:].decode('hex')
    if tmp.startswith('test') and '\n' not in tmp:
        break
    else:
        part1 = (digitalize('test' + os.urandom(51)) << 512) // (p - 1) * (
            p - 1)
        victim = part1 + digitalize(target)
```

The final script is as follows:

```python
from pwn import *
from hashlib import sha1
import string
import ast
import os
import binascii
context.log_level = 'debug'


def f(x):
    return sha1(prefix + x).digest()[-2:] == '\0\0'


def digitalize(m):
    return int(m.encode('hex'), 16)


sh = remote('106.75.66.195', 40001)
# bypass proof
sh.recvuntil('starting with ')
prefix = sh.recvuntil('\n', drop=True)
print string.ascii_letters
s = util.iters.mbruteforce(f, string.ascii_letters + string.digits, 5, 'fixed')
test = prefix + s
sh.sendline(test)

sh.recvuntil('Current PK we are using: ')
pubkey = ast.literal_eval(sh.recvuntil('\n', drop=True))
p = pubkey[0]
g = pubkey[1]
pk = pubkey[2]

# construct the message begins with 'test'
target = "Th3_bery_un1que1i_ChArmIng_G3nji" + test
part1 = (digitalize('test' + os.urandom(51)) << 512) // (p - 1) * (p - 1)
victim = part1 + digitalize(target)
while 1:
    tmp = hex(victim)[2:].decode('hex')
    if tmp.startswith('test') and '\n' not in tmp:
        break
    else:
        part1 = (digitalize('test' + os.urandom(51)) << 512) // (p - 1) * (
            p - 1)
        victim = part1 + digitalize(target)

assert (victim % (p - 1) == digitalize(target) % (p - 1))

# get victim signature
sh.sendline(hex(victim)[2:].decode('hex'))
sh.recvuntil('Your signature is')
sig = ast.literal_eval(sh.recvuntil('\n', drop=True))
sig = [int(sig[0], 0), int(sig[1], 0)]

# get flag
sh.sendline(target)
sh.sendline(str(sig[0]) + "," + str(sig[1]))
sh.interactive()
```

Here are a few interesting points worth mentioning:

- `int(x,0)` converts x according to the literal base indicated by its prefix. For example, `int('0x12',0)=18`. The literal must have the corresponding prefix, e.g., `0x` for hexadecimal, `0` for octal, `0b` for binary, because without it, the base cannot be determined.
- In Python (Python2), how large does a number need to be before the computed result includes an `L` suffix? Normally, any number larger than int will have `L`. But the `victim` variable here does not have one... **This is an open question to be resolved.**
