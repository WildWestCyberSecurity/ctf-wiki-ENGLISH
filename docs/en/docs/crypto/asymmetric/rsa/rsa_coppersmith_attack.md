# Coppersmith Related Attacks

## Basic Principles

Coppersmith related attacks are closely associated with [Don Coppersmith](https://en.wikipedia.org/wiki/Don_Coppersmith), who proposed a polynomial-time method for finding all small integer roots of modular polynomials (univariate, bivariate, and even multivariate).

Here we mainly introduce the univariate case. Assume that

- The modulus is N, and N has a factor $b\geq N^{\beta},0< \beta \leq 1$
- The polynomial F has degree $\delta$

Then this method can find all roots $x_0$ of the polynomial within $O(c\delta^5log^9(N))$ complexity, where we require $|x_0|<cN^{\frac{\beta^2}{\delta}}$.

In this problem, our goal is to find all roots of a polynomial modulo N, which is considered to be a difficult problem. The **Coppersmith method** mainly uses the [Lenstra–Lenstra–Lovász lattice basis reduction algorithm](https://en.wikipedia.org/wiki/Lenstra%E2%80%93Lenstra%E2%80%93Lov%C3%A1sz_lattice_basis_reduction_algorithm) (LLL) to find a polynomial g that

- Has the same root $x_0$ as the original polynomial
- Has smaller coefficients
- Is defined over the integers

Since finding roots of a polynomial over the integers is easy (Berlekamp–Zassenhaus), we can thus obtain the integer roots of the original polynomial in the modular sense.

The key question then is how to transform f into g. Howgrave-Graham provided an approach:

![image-20180717210921382](figure/coppersmith-howgrave-graham.png)

That is, we need to find a polynomial g with "smaller coefficients", using the following transformation:

![image-20180717211351350](figure/coppersmith-f2g.png)

In the LLL algorithm, two properties are very useful:

- It only performs integer linear transformations on the original basis vectors, which ensures that when we obtain g, it still has the original $x_0$ as a root.
- The norms of the newly generated basis vectors are bounded, which allows us to apply the Howgrave-Graham theorem.

With this foundation, we just need to construct the polynomial family g.

For more detailed content, please search on your own. This section will also be continuously updated.

It should be noted that due to the constraints on Coppersmith's roots, when applied to RSA, it is generally only applicable when e is small.

## Basic Broadcast Attack

### Attack Conditions

If a user encrypts the same plaintext using the same encryption exponent e and sends it to e other users, then a broadcast attack can be performed. This attack was proposed by Håstad.

### Attack Principle

Here we assume e is 3, and the encryptor used three different moduli $n_1,n_2,n_3$ to send the encrypted message m to three different users, as follows:

$$
\begin{align*}
c_1&=m^3\bmod n_1 \\
c_2&=m^3\bmod n_2 \\
c_3&=m^3\bmod n_3
\end{align*}
$$

Here we assume $n_1,n_2,n_3$ are pairwise coprime; otherwise, we could directly factorize them, obtain d, and then decrypt directly.

Additionally, we assume $m<n_i, 1\leq i \leq 3$. If this condition is not satisfied, the situation becomes more complicated, which we will not discuss here for now.

Since they are pairwise coprime, by the Chinese Remainder Theorem, we can obtain $m^3 \equiv C \bmod n_1n_2n_3$.

Furthermore, since $m<n_i, 1\leq i \leq 3$, we know that $m^3 < n_1n_2n_3$ and $C<m^3 < n_1n_2n_3$, so $m^3 = C$. We can obtain the value of m by taking the cube root of C.

For larger values of e, we simply need more plaintext-ciphertext pairs.

### SCTF RSA3 LEVEL4

Reference: http://ohroot.com/2016/07/11/rsa-in-ctf.

Here we use level4 from SCTF RSA3 as an example. First, write code to extract data from the pcap file, as follows:

```shell
#!/usr/bin/env python

from scapy.all import *
import zlib
import struct

PA = 24
packets = rdpcap('./syc_security_system_traffic3.pcap')
client = '192.168.1.180'
list_n = []
list_m = []
list_id = []
data = []
for packet in packets:
    # TCP Flag PA 24 means carry data
    if packet[TCP].flags == PA or packet[TCP].flags == PA + 1:
        src = packet[IP].src
        raw_data = packet[TCP].load
        head = raw_data.strip()[:7]
        if head == "We have":
            n, e = raw_data.strip().replace("We have got N is ",
                                            "").split('\ne is ')
            data.append(n.strip())
        if head == "encrypt":
            m = raw_data.replace('encrypted messages is 0x', '').strip()
            data.append(str(int(m, 16)))

with open('./data.txt', 'w') as f:
    for i in range(0, len(data), 2):
        tmp = ','.join(s for s in data[i:i + 2])
        f.write(tmp + '\n')

```

Next, use the obtained data to solve directly using the Chinese Remainder Theorem.

```python
from functools import reduce
import gmpy
import json, binascii


def modinv(a, m):
    return int(gmpy.invert(gmpy.mpz(a), gmpy.mpz(m)))


def chinese_remainder(n, a):
    sum = 0
    prod = reduce(lambda a, b: a * b, n)
    # parallel computation
    for n_i, a_i in zip(n, a):
        p = prod // n_i
        sum += a_i * modinv(p, n_i) * p
    return int(sum % prod)


nset = []
cset = []
with open("data.txt") as f:
    now = f.read().strip('\n').split('\n')
    for item in now:
        item = item.split(',')
        nset.append(int(item[0]))
        cset.append(int(item[1]))

m = chinese_remainder(nset, cset)
m = int(gmpy.mpz(m).root(19)[0])
print binascii.unhexlify(hex(m)[2:-1])

```

Obtain the ciphertext, then decrypt again to get the flag.

```shell
H1sTaDs_B40aDcadt_attaCk_e_are_same_and_smA9l
```

### Challenges

- 2017 WHCTF OldDriver
- 2018 N1CTF easy_fs

## Broadcast Attack with Linear Padding

For the case with linear padding, the attack is still possible. In this case, the **Coppersmith method** is used. We will not introduce it here for now. You can refer to:

- https://en.wikipedia.org/wiki/Coppersmith%27s_attack#Generalizations

## Related Message Attack

### Attack Conditions

When Alice uses the same public key to encrypt two messages M1 and M2 that have a certain linear relationship, and sends the encrypted messages C1 and C2 to Bob, we may be able to obtain the corresponding messages M1 and M2. Here we assume the modulus is N, and the linear relationship between the two is as follows:

$$
M_1 \equiv f(M_2) \bmod N
$$

where f is a linear function, for example $f=ax+b$.

Under conditions with a small error probability, its complexity is $O(elog^2N)$.

This attack was proposed by Franklin and Reiter.

### Attack Principle

First, we know that $C_1 \equiv M_1 ^e \bmod N$, and $M_1 \equiv f(M_2) \bmod N$, so we can determine that $M_2$ is a solution to $f(x)^e \equiv C_1 \bmod N$, i.e., it is a root of the equation $f(x)^e-C_1$ modulo N. Similarly, $M_2$ is a root of $x^e - C_2$ modulo N. Therefore $x-M_2$ simultaneously divides both polynomials. Thus, we can find the greatest common divisor of the two polynomials. If the greatest common divisor happens to be linear, then we have found $M_2$. Note that when $e=3$, the greatest common divisor is always linear.

Here we focus on the case where $e=3$ and $f(x)=ax+b$. First, we have:

$$
C_1 \equiv M_1 ^3 \bmod N,M_1 \equiv aM_2+b \bmod N
$$

Then we have:

$$
C_1 \equiv (aM_2+b)^3 \bmod N,C_2 \equiv M_2^3 \bmod N
$$

We need to clarify that what we want to obtain is the message m, so we need to construct it separately.

First, we have equation 1:

$$
(aM_2+b)^3=a^3M_2^3+3a^2M^2b+3aM_2b^2+b^3
$$

Then we construct equation 2 as follows:

$$
(aM_2)^3-b^3 \equiv (aM_2-b)(a^2M_2^2+aM_2b+b^2) \bmod N
$$

From equation 1 we have:

$$
a^3M_2^3-2b^3+3b(a^2M_2^2+aM_2b+b^2) \equiv C_1 \bmod N
$$

Thus we obtain equation 3:

$$
3b(a^2M_2^2+aM_2b+b^2) \equiv C_1-a^3C_2+2b^3 \bmod N
$$

From equations 2 and 3, we get:

$$
(a^3C_2-b^3)*3b \equiv (aM_2-b)( C_1-a^3C_2+2b^3 ) \bmod N
$$

Therefore we have:

$$
aM_2-b=\frac{3a^3bC_2-3b^4}{C_1-a^3C_2+2b^3}
$$

And thus:

$$
aM_2\equiv  \frac{2a^3bC_2-b^4+C_1b}{C_1-a^3C_2+2b^3}
$$

And finally:

$$
M_2 \equiv\frac{2a^3bC_2-b^4+C_1b}{aC_1-a^4C_2+2ab^3}=\frac{b}{a}\frac{C_1+2a^3C_2-b^3}{C_1-a^3C_2+2b^3}
$$

In the above equation, everything on the right side is known, so we can directly obtain the corresponding message.

For those interested, you can further read [A New Related Message Attack on RSA](https://www.iacr.org/archive/pkc2005/33860001/33860001.pdf) and the original [paper](https://www.cs.unc.edu/~reiter/papers/1996/Eurocrypt.pdf). We will not go into more detail here.

### SCTF RSA3

Here we use level3 from SCTF RSA3 as an example. First, by tracking the TCP stream, we can determine that the encryption method adds the user's user id to the plaintext before encrypting, and there are multiple groups. Here we select group 0 and group 9, which have the same modulus. The decryption script is as follows:

```python
import gmpy2
id1 = 1002
id2 = 2614

c1 = 0x547995f4e2f4c007e6bb2a6913a3d685974a72b05bec02e8c03ba64278c9347d8aaaff672ad8460a8cf5bffa5d787c5bb724d1cee07e221e028d9b8bc24360208840fbdfd4794733adcac45c38ad0225fde19a6a4c38e4207368f5902c871efdf1bdf4760b1a98ec1417893c8fce8389b6434c0fee73b13c284e8c9fb5c77e420a2b5b1a1c10b2a7a3545e95c1d47835c2718L
c2 = 0x547995f4e2f4c007e6bb2a6913a3d685974a72b05bec02e8c03ba64278c9347d8aaaff672ad8460a8cf5bffa5d787c72722fe4fe5a901e2531b3dbcb87e5aa19bbceecbf9f32eacefe81777d9bdca781b1ec8f8b68799b4aa4c6ad120506222c7f0c3e11b37dd0ce08381fabf9c14bc74929bf524645989ae2df77c8608d0512c1cc4150765ab8350843b57a2464f848d8e08L
n = 25357901189172733149625332391537064578265003249917817682864120663898336510922113258397441378239342349767317285221295832462413300376704507936359046120943334215078540903962128719706077067557948218308700143138420408053500628616299338204718213283481833513373696170774425619886049408103217179262264003765695390547355624867951379789924247597370496546249898924648274419164899831191925127182066301237673243423539604219274397539786859420866329885285232179983055763704201023213087119895321260046617760702320473069743688778438854899409292527695993045482549594428191729963645157765855337481923730481041849389812984896044723939553
a = 1
b = id1 - id2


def getmessage(a, b, c1, c2, n):
    b3 = gmpy2.powmod(b, 3, n)
    part1 = b * (c1 + 2 * c2 - b3) % n
    part2 = a * (c1 - c2 + 2 * b3) % n
    part2 = gmpy2.invert(part2, n)
    return part1 * part2 % n


message = getmessage(a, b, c1, c2, n) - id2
message = hex(message)[2:]
if len(message) % 2 != 0:
    message = '0' + message

print message.decode('hex')

```

The plaintext is obtained:

```shell
➜  sctf-rsa3-level3 git:(master) ✗ python exp.py
F4An8LIn_rElT3r_rELa53d_Me33Age_aTtaCk_e_I2_s7aLL
```

Of course, we can also use sage directly, which is even simpler.

```python
import binascii

def attack(c1, c2, b, e, n):
    PR.<x>=PolynomialRing(Zmod(n))
    g1 = x^e - c1
    g2 = (x+b)^e - c2

    def gcd(g1, g2):
        while g2:
            g1, g2 = g2, g1 % g2
        return g1.monic()
    return -gcd(g1, g2)[0]

c1 = 0x547995f4e2f4c007e6bb2a6913a3d685974a72b05bec02e8c03ba64278c9347d8aaaff672ad8460a8cf5bffa5d787c5bb724d1cee07e221e028d9b8bc24360208840fbdfd4794733adcac45c38ad0225fde19a6a4c38e4207368f5902c871efdf1bdf4760b1a98ec1417893c8fce8389b6434c0fee73b13c284e8c9fb5c77e420a2b5b1a1c10b2a7a3545e95c1d47835c2718L
c2 = 0x547995f4e2f4c007e6bb2a6913a3d685974a72b05bec02e8c03ba64278c9347d8aaaff672ad8460a8cf5bffa5d787c72722fe4fe5a901e2531b3dbcb87e5aa19bbceecbf9f32eacefe81777d9bdca781b1ec8f8b68799b4aa4c6ad120506222c7f0c3e11b37dd0ce08381fabf9c14bc74929bf524645989ae2df77c8608d0512c1cc4150765ab8350843b57a2464f848d8e08L
n = 25357901189172733149625332391537064578265003249917817682864120663898336510922113258397441378239342349767317285221295832462413300376704507936359046120943334215078540903962128719706077067557948218308700143138420408053500628616299338204718213283481833513373696170774425619886049408103217179262264003765695390547355624867951379789924247597370496546249898924648274419164899831191925127182066301237673243423539604219274397539786859420866329885285232179983055763704201023213087119895321260046617760702320473069743688778438854899409292527695993045482549594428191729963645157765855337481923730481041849389812984896044723939553
e=3
a = 1
id1 = 1002
id2 = 2614
b = id2 - id1
m1 = attack(c1,c2, b,e,n)
print binascii.unhexlify("%x" % int(m1 - id1))
```

The result is as follows:

```shell
➜  sctf-rsa3-level3 git:(master) ✗ sage exp.sage
sys:1: RuntimeWarning: not adding directory '' to sys.path since everybody can write to it.
Untrusted users could put files in this directory which might then be imported by your Python code. As a general precaution from similar exploits, you should not execute Python code from this directory
F4An8LIn_rElT3r_rELa53d_Me33Age_aTtaCk_e_I2_s7aLL
```

### Challenges

- hitcon 2014 rsaha
- N1CTF 2018 rsa_padding

## Coppersmith's short-pad attack

### Attack Conditions

Currently, most messages are padded before encryption, but if the padding length is too short, it **may** be easily attacked.

The so-called short padding here essentially means that the corresponding polynomial root is too small.

### Attack Principle

Suppose Alice wants to send a message to Bob. First, Alice randomly pads the message M to be encrypted, then encrypts it to obtain ciphertext C1, and sends it to Bob. At this point, the man-in-the-middle Pete intercepts the ciphertext. After some time, Alice does not receive a reply from Bob, so she randomly pads the message M again, encrypts it to obtain ciphertext C2, and sends it to Bob. Pete intercepts it once again. At this point, Pete **may** be able to decrypt using the following principle.

Here we assume the length of the modulus N is k, and the padding length is $m=\lfloor \frac{k}{e^2} \rfloor$. Furthermore, assume the length of the message to be encrypted is at most k-m bits, and the padding method is as follows:

$$
M_1=2^mM+r_1, 0\leq r_1\leq 2^m
$$

The padding method for message M2 is similar.

Then we can decrypt using the following approach.

First, define:

$$
g_1(x,y)=x^e-C_1
g_2(x,y)=(x+y)^e-C_2
$$

where $y=r_2-r_1$. Obviously these two equations share the same root M1. Then there is a series of derivations.

## Known High Bits Message Attack

### Attack Conditions

Here we assume that we first encrypted a message m, as follows:

$$
C\equiv m^d \bmod N
$$

And we assume that we know a large portion $m_0$ of the message m, i.e., $m=m_0+x$, but we don't know $x$. Then we may be able to recover the message using this method. Here the unknown x is essentially the root of the polynomial, which needs to satisfy Coppersmith's constraints.

You can refer to https://github.com/mimoo/RSA-and-LLL-attacks.

## Factoring with High Bits Known

### Attack Conditions

When we know the higher bits of one factor of the modulus N in a public key, we have a certain probability of factoring N.

### Attack Tools

Please refer to https://github.com/mimoo/RSA-and-LLL-attacks, which includes usage tutorials. Pay attention to the following code:

```python
beta = 0.5
dd = f.degree()
epsilon = beta / 7
mm = ceil(beta**2 / (dd * epsilon))
tt = floor(dd * mm * ((1/beta) - 1))
XX = ceil(N**((beta**2/dd) - epsilon)) + 1000000000000000000000000000000000
roots = coppersmith_howgrave_univariate(f, N, beta, mm, tt, XX)
```

Where:

- It must satisfy $q\geq N^{beta}$, so here $beta=0.5$ is given. Obviously, one of the two factors must be greater.
- XX is the upper bound of the root of $f(x)=q'+x$ modulo q. Naturally, we can choose to adjust it, which also indicates the possible gap between the known $q'$ and the factor q.

### 2016 HCTF RSA2

Here we use RSA2 from 2016 HCTF as an example.

First, the beginning of the program is a verification bypass. Just bypass it. The code is as follows:

```python
from pwn import *
from hashlib import sha512
sh = remote('127.0.0.1', 9999)
context.log_level = 'debug'
def sha512_proof(prefix, verify):
    i = 0
    pading = ""
    while True:
        try:
            i = randint(0, 1000)
            pading += str(i)
            if len(pading) > 200:
                pading = pading[200:]
            #print pading
        except StopIteration:
            break
        r = sha512(prefix + pading).hexdigest()
        if verify in r:
            return pading


def verify():
    sh.recvuntil("Prefix: ")
    prefix = sh.recvline()
    print len(prefix)
    prefix = prefix[:-1]
    prefix = prefix.decode('base64')
    proof = sha512_proof(prefix, "fffffff")
    sh.send(proof.encode('base64'))
if __name__ == '__main__':
    verify()
    print 'verify success'
    sh.recvuntil("token: ")
    token = "5c9597f3c8245907ea71a89d9d39d08e"
    sh.sendline(token)

    sh.recvuntil("n: ")
    n = sh.readline().strip()
    n = int(n[2:], 16)

    sh.recvuntil("e: ")
    e = sh.readline().strip()
    e = int(e[2:], 16)

    sh.recvuntil("e2: ")
    e2 = sh.readline().strip()
    e2 = int(e2[2:], 16)

    sh.recvuntil("is: ")
    enc_flag = sh.readline().strip()
    enc_flag = int(enc_flag[2:-1], 16)
    print "n: ", hex(n)
    print "e: ", hex(e)
    print "e2: ", hex(e2)
    print "flag: ", hex(enc_flag)
```

Here we have also obtained n, e, e2, and the encrypted flag, as follows:

```python
n:  0x724d41149e1bd9d2aa9b333d467f2dfa399049a5d0b4ee770c9d4883123be11a52ff1bd382ad37d0ff8d58c8224529ca21c86e8a97799a31ddebd246aeeaf0788099b9c9c718713561329a8e529dfeae993036921f036caa4bdba94843e0a2e1254c626abe54dc3129e2f6e6e73bbbd05e7c6c6e9f44fcd0a496f38218ab9d52bf1f266004180b6f5b9bee7988c4fe5ab85b664280c3cfe6b80ae67ed8ba37825758b24feb689ff247ee699ebcc4232b4495782596cd3f29a8ca9e0c2d86ea69372944d027a0f485cea42b74dfd74ec06f93b997a111c7e18017523baf0f57ae28126c8824bd962052623eb565cee0ceee97a35fd8815d2c5c97ab9653c4553f
e:  0x10001
e2:  0xf93b
flag:  0xf11e932fa420790ca3976468dc4df1e6b20519ebfdc427c09e06940e1ef0ca566d41714dc1545ddbdcae626eb51c7fa52608384a36a2a021960d71023b5d0f63e6b38b46ac945ddafea42f01d24cc33ce16825df7aa61395d13617ae619dca2df15b5963c77d6ededf2fe06fd36ae8c5ce0e3c21d72f2d7f20cd9a8696fbb628df29299a6b836c418cbfe91e2b5be74bdfdb4efdd1b33f57ebb72c5246d5dce635529f1f69634d565a631e950d4a34a02281cbed177b5a624932c2bc02f0c8fd9afd332ccf93af5048f02b8bd72213d6a52930b0faa0926973883136d8530b8acf732aede8bb71cb187691ebd93a0ea8aeec7f82d0b8b74bcf010c8a38a1fa8
```

Next, let's analyze the main program. We can see that:

```python
	p, q, e = gen_key()
	n = p * q
	phi_n = (p-1)*(q-1)
	d = invmod(e, phi_n)
	while True:
		e2 = random.randint(0x1000, 0x10000)
		if gcd(e2, phi_n) == 1:
			break
```

We obtained $n=p \times q$. And p, q, and the already known e are all generated in the `gen_key` function. Let's look at the `gen_key` function:

```python
def gen_key():
	while True:
		p = getPrime(k/2)
		if gcd(e, p-1) == 1:
			break
	q_t = getPrime(k/2)
	n_t = p * q_t
	t = get_bit(n_t, k/16, 1)
	y = get_bit(n_t, 5*k/8, 0)
	p4 = get_bit(p, 5*k/16, 1)
	u = pi_b(p4, 1)
	n = bytes_to_long(long_to_bytes(t) + long_to_bytes(u) + long_to_bytes(y))
	q = n / p
	if q % 2 == 0:
		q += 1
	while True:
		if isPrime(q) and gcd(e, q-1) == 1:
			break
		m = getPrime(k/16) + 1
		q ^= m
	return (p, q, e)
```

Where the known parameters are:

$$
k=2048
e=0x10001
$$

First, the program obtains a 1024-bit prime p, with `gcd(2,p-1)=1`.

Then, the program obtains another 1024-bit prime $q_t$, and computes $n_t=p \times q_t$.

The `get_bit` function is called multiple times below. Let's briefly analyze it:

```python
def get_bit(number, n_bit, dire):
	'''
	dire:
		1: left
		0: right
	'''

	if dire:
		sn = size(number)
		if sn % 8 != 0:
			sn += (8 - sn % 8)
		return number >> (sn-n_bit)
	else:
		return number & (pow(2, n_bit) - 1)
```

We can see that depending on the `dire(ction)`, different numbers are obtained:

- When `dire=1`, the program first calculates the number of binary digits `sn` of `number`. If it's not a multiple of 8, `sn` is increased to the next multiple of 8, then it returns `number` right-shifted by `sn-n_bit`. This essentially keeps at most the top `n_bit` bits of `number`.
- When `dire=0`, the program directly gets the lower `n_bit` bits of `number`.

Then let's look at the program:

```python
	t = get_bit(n_t, k/16, 1)
	y = get_bit(n_t, 5*k/8, 0)
	p4 = get_bit(p, 5*k/16, 1)
```

These three operations respectively do the following:

- `t` is at most the top k/16 bits of `n_t`, i.e., 128 bits, with a variable number of bits.
- `y` is the lower 5*k/8 bits of `n_t`, i.e., 1280 bits, with a fixed number of bits.
- `p4` is at most the top 5*k/16 bits of p, i.e., 640 bits, with a variable number of bits.

After that, the program performs the following operation:

```python
	u = pi_b(p4, 1)
```

Using `pi_b` to encrypt `p4`:

```python
def pi_b(x, m):
	'''
	m:
		1: encrypt
		0: decrypt
	'''
	enc = DES.new(key)
	if m:
		method = enc.encrypt
	else:
		method = enc.decrypt
	s = long_to_bytes(x)
	sp = [s[a:a+8] for a in xrange(0, len(s), 8)]
	r = ""
	for a in sp:
		r += method(a)
	return bytes_to_long(r)
```

We already know the key, so as long as we have the ciphertext, we can decrypt it. Furthermore, we can see that the program groups the input message into 8-byte blocks and encrypts using ECB mode, so the ciphertext blocks are independent of each other.

Below:

```python
	n = bytes_to_long(long_to_bytes(t) + long_to_bytes(u) + long_to_bytes(y))
	q = n / p
	if q % 2 == 0:
		q += 1
	while True:
		if isPrime(q) and gcd(e, q-1) == 1:
			break
		m = getPrime(k/16) + 1
		q ^= m
	return (p, q, e)
```

The program concatenates t, u, and y together to get n, then obtains q, XORs the lower k/16 bits of q, and returns `q'`.

In the main program, `n'=p*q'` is obtained again. Let's carefully analyze this:

```
n'=p * ( q + random(2^{k/16}))
```

Since p is k/2 bits, the random part can affect at most the lowest $k/2+k/16=9k/16$ bits of the original n.

Moreover, we also know that the lowest 5k/8=10k/16 bits of n are actually y, so it does not affect u; even if it does, it affects at most one bit.

So we can first use the n we obtained to extract u, as follows:

```
u=hex(n)[2:-1][-480:-320]
```

Although this might get extra bits of u, it doesn't matter. When we decrypt u, each block is independent of the others, so we can only affect the highest-order bits of p4. And the top 8 bits of p4 may also be padding. But this doesn't matter either — we have already obtained a large portion of the factor p, and we can try to decrypt. As follows:

```python
if __name__=="__main__":
	n = 0x724d41149e1bd9d2aa9b333d467f2dfa399049a5d0b4ee770c9d4883123be11a52ff1bd382ad37d0ff8d58c8224529ca21c86e8a97799a31ddebd246aeeaf0788099b9c9c718713561329a8e529dfeae993036921f036caa4bdba94843e0a2e1254c626abe54dc3129e2f6e6e73bbbd05e7c6c6e9f44fcd0a496f38218ab9d52bf1f266004180b6f5b9bee7988c4fe5ab85b664280c3cfe6b80ae67ed8ba37825758b24feb689ff247ee699ebcc4232b4495782596cd3f29a8ca9e0c2d86ea69372944d027a0f485cea42b74dfd74ec06f93b997a111c7e18017523baf0f57ae28126c8824bd962052623eb565cee0ceee97a35fd8815d2c5c97ab9653c4553f
	u = hex(n)[2:-1][-480:-320]
	u = int(u,16)
	p4 = pi_b(u,0)
	print hex(p4)
```

The decryption result is as follows:

```python
➜  2016-HCTF-RSA2 git:(master) ✗ python exp_p4.py
0xa37302107c17fb4ef5c3443f4ef9e220ac659670077b9aa9ff7381d11073affe9183e88acae0ab61fb75a3c7815ffcb1b756b27c4d90b2e0ada753fa17cc108c1d0de82c747db81b9e6f49bde1362693L
```

Next, we directly use sage to decrypt. Sage has already implemented this attack, so we can use it directly:

```python
from sage.all import *
import binascii
n = 0x724d41149e1bd9d2aa9b333d467f2dfa399049a5d0b4ee770c9d4883123be11a52ff1bd382ad37d0ff8d58c8224529ca21c86e8a97799a31ddebd246aeeaf0788099b9c9c718713561329a8e529dfeae993036921f036caa4bdba94843e0a2e1254c626abe54dc3129e2f6e6e73bbbd05e7c6c6e9f44fcd0a496f38218ab9d52bf1f266004180b6f5b9bee7988c4fe5ab85b664280c3cfe6b80ae67ed8ba37825758b24feb689ff247ee699ebcc4232b4495782596cd3f29a8ca9e0c2d86ea69372944d027a0f485cea42b74dfd74ec06f93b997a111c7e18017523baf0f57ae28126c8824bd962052623eb565cee0ceee97a35fd8815d2c5c97ab9653c4553f
p4 =0xa37302107c17fb4ef5c3443f4ef9e220ac659670077b9aa9ff7381d11073affe9183e88acae0ab61fb75a3c7815ffcb1b756b27c4d90b2e0ada753fa17cc108c1d0de82c747db81b9e6f49bde1362693
cipher = 0xf11e932fa420790ca3976468dc4df1e6b20519ebfdc427c09e06940e1ef0ca566d41714dc1545ddbdcae626eb51c7fa52608384a36a2a021960d71023b5d0f63e6b38b46ac945ddafea42f01d24cc33ce16825df7aa61395d13617ae619dca2df15b5963c77d6ededf2fe06fd36ae8c5ce0e3c21d72f2d7f20cd9a8696fbb628df29299a6b836c418cbfe91e2b5be74bdfdb4efdd1b33f57ebb72c5246d5dce635529f1f69634d565a631e950d4a34a02281cbed177b5a624932c2bc02f0c8fd9afd332ccf93af5048f02b8bd72213d6a52930b0faa0926973883136d8530b8acf732aede8bb71cb187691ebd93a0ea8aeec7f82d0b8b74bcf010c8a38a1fa8
e2 = 0xf93b
pbits = 1024
kbits = pbits - p4.nbits()
print p4.nbits()
p4 = p4 << kbits
PR.<x> = PolynomialRing(Zmod(n))
f = x + p4
roots = f.small_roots(X=2^kbits, beta=0.4)
if roots:
    p = p4+int(roots[0])
    print "p: ", hex(int(p))
    assert n % p == 0
    q = n/int(p)
    print "q: ", hex(int(q))
    print gcd(p,q)
    phin = (p-1)*(q-1)
    print gcd(e2,phin)
    d = inverse_mod(e2,phin)
    flag = pow(cipher,d,n)
    flag = hex(int(flag))[2:-1]
    print binascii.unhexlify(flag)
```

For usage of `small_roots`, please refer to the [SAGE documentation](http://doc.sagemath.org/html/en/reference/polynomial_rings/sage/rings/polynomial/polynomial_modn_dense_ntl.html#sage.rings.polynomial.polynomial_modn_dense_ntl.small_roots).

The result is as follows:

```shell
➜  2016-HCTF-RSA2 git:(master) ✗ sage payload.sage
sys:1: RuntimeWarning: not adding directory '' to sys.path since everybody can write to it.
Untrusted users could put files in this directory which might then be imported by your Python code. As a general precaution from similar exploits, you should not execute Python code from this directory
640
p:  0xa37302107c17fb4ef5c3443f4ef9e220ac659670077b9aa9ff7381d11073affe9183e88acae0ab61fb75a3c7815ffcb1b756b27c4d90b2e0ada753fa17cc108c1d0de82c747db81b9e6f49bde13626933aa6762057e1df53d27356ee6a09b17ef4f4986d862e3bb24f99446a0ab2385228295f4b776c1f391ab2a0d8c0dec1e5L
q:  0xb306030a7c6ace771db8adb45fae597f3c1be739d79fd39dfa6fd7f8c177e99eb29f0462c3f023e0530b545df6e656dadb984953c265b26f860b68aa6d304fa403b0b0e37183008592ec2a333c431e2906c9859d7cbc4386ef4c4407ead946d855ecd6a8b2067ad8a99b21111b26905fcf0d53a1b893547b46c3142b06061853L
1
1
hctf{d8e8fca2dc0f896fd7cb4cb0031ba249}
```

### Challenges

- 2016 Huxiang Cup Simple RSA
- 2017 WHCTF Untitled

## Boneh and Durfee attack

### Attack Conditions

When d is small, satisfying $d < N^{0.292}$, we can use this attack, which is stronger than Wiener's Attack.

### Attack Principle

Here we briefly explain the principle.

First:

$$
ed \equiv 1 \bmod  \varphi(N)/2
$$

Therefore:

$$
ed +k\varphi(N)/2=1
$$

That is:

$$
k \varphi(N)/2 \equiv 1 \bmod e
$$

Also:

$$
\varphi(N)=(p-1)(q-1)=qp-p-q+1=N-p-q+1
$$

So:

$$
k(N-p-q+1)/2 \equiv 1 \bmod e
$$

Assume $A=\frac{N+1}{2}$, $y=\frac{-p-q}{2}$, the original formula can be transformed to:

$$
f(k,y)=k(A+y) \equiv 1 \bmod e
$$

Where:

$|k|<\frac{2ed}{\varphi(N)}<\frac{3ed}{N}=3*\frac{e}{N}*d<3*\frac{e}{N}*N^{delta}$

$|y|<2*N^{0.5}$

The estimate of y uses the assumption that p and q are roughly equal in size. Here delta is an estimated value less than 0.292.

If we can find the roots of this bivariate equation, we can naturally solve the quadratic equation $N=pq,p+q=-2y$ to obtain p and q.

For more detailed derivation, refer to New Results on the Cryptanalysis of Low Exponent RSA.

### Attack Tools

Please refer to https://github.com/mimoo/RSA-and-LLL-attacks, which includes usage tutorials.

### 2015 PlaidCTF Curious

Here we use Curious from 2015 PlaidCTF as an example.

First, the problem provides a bunch of N, e, c values. A quick look reveals that e is relatively large. In this case, we can consider using Wiener's Attack, but here we use the stronger attack introduced above.

The core code is as follows:

```python
    nlist = list()
    elist = list()
    clist = list()
    with open('captured') as f:
        # read the line {N : e : c} and do nothing with it
        f.readline()
        for i in f.readlines():
            (N, e, c) = i[1:-2].split(" : ")
            nlist.append(long(N,16))
            elist.append(long(e,16))
            clist.append(long(c,16))

    for i in range(len(nlist)):
        print 'index i'
        n = nlist[i]
        e = elist[i]
        c = clist[i]
        d = solve(n,e)
        if d==0:
            continue
        else:
            m = power_mod(c, d, n)
            hex_string = "%x" % m
            import binascii
            print "the plaintext:", binascii.unhexlify(hex_string)
            return
```

The result is as follows:

```shell
=== solution found ===
private key found: 23974584842546960047080386914966001070087596246662608796022581200084145416583
the plaintext: flag_S0Y0UKN0WW13N3R$4TT4CK!
```

### 2019 Defcon Quals ASRybaB

The general idea of the problem is that we receive three pairs of RSA parameters, then need to find d and encrypt a given number v[i], and send it to the server. As long as the time is within a certain range (940s), we pass. The difficulty naturally lies in the `create_key` function.

```python
def send_challenges():

    code = marshal.loads("63000000000d000000070000004300000073df010000740000721d0064010064020015000000000100640200157d00006e00007401007d01007c0100640300157d02006402007d0300786f007c03006a02008300007c01006b030072a400784c007403007296007404006a05007c02008301007d04007404006a05007c02008301007d05007406007c04007c0500188301006a02008300007c0100640400146b0400724b0050714b00714b00577c04007c0500147d0300713600577c0400640500187c050064050018147d06006406007d07006407007d080078090174030072ce017404006a07007408006403007409007c01007c0700148301008302007408006403007409007c01007c070014830100640500178302008302007d09007871007c09006a02008300007c01007c0800146b0000727b016402007d0a007844007404006a0a007c0a00830100736d017404006a0700740800640300640800830200740800640300640800830200740800640300640900830200178302007d0a00712a01577c09007c0a00397d0900710b01577404006a0b007c09007c06008302006405006b0300729a0171c6006e00007404006a0c007c09007c06008302007d0b007404006a0b007c0b007c06008302006405006b030072ca0171c6006e00005071c60057640a007d0c007c03007c0b0066020053280b0000004e690700000069000000006902000000675839b4c876bedf3f6901000000674e62105839b4d03f678d976e1283c0d23f692d000000690c0000006903000000280d000000740500000046616c736574050000004e53495a45740a0000006269745f6c656e67746874040000005472756574060000006e756d626572740e0000006765745374726f6e675072696d657403000000616273740e00000067657452616e646f6d52616e67657403000000706f777403000000696e74740700000069735072696d6574030000004743447407000000696e7665727365280d00000074010000007874050000004e73697a657406000000707173697a6574010000004e740100000070740100000071740300000070686974060000006c696d69743174060000006c696d697432740100000064740300000070707074010000006574030000007a7a7a2800000000280000000073150000002f6f726967696e616c6368616c6c656e67652e7079740a0000006372656174655f6b657917000000733e000000000106010a010d0206010a010601150109010f010f04200108010e0112020601060109013c0119010601120135020e011801060112011801060105020604".decode("hex"))
    create_key = types.FunctionType(code, globals(), "create_key")
    
    ck = create_key
```

We can take a simple look at what this is actually doing:

```python
>>> import marshal
>>> data="63000000000d000000070000004300000073df010000740000721d0064010064020015000000000100640200157d00006e00007401007d01007c0100640300157d02006402007d0300786f007c03006a02008300007c01006b030072a400784c007403007296007404006a05007c02008301007d04007404006a05007c02008301007d05007406007c04007c0500188301006a02008300007c0100640400146b0400724b0050714b00714b00577c04007c0500147d0300713600577c0400640500187c050064050018147d06006406007d07006407007d080078090174030072ce017404006a07007408006403007409007c01007c0700148301008302007408006403007409007c01007c070014830100640500178302008302007d09007871007c09006a02008300007c01007c0800146b0000727b016402007d0a007844007404006a0a007c0a00830100736d017404006a0700740800640300640800830200740800640300640800830200740800640300640900830200178302007d0a00712a01577c09007c0a00397d0900710b01577404006a0b007c09007c06008302006405006b0300729a0171c6006e00007404006a0c007c09007c06008302007d0b007404006a0b007c0b007c06008302006405006b030072ca0171c6006e00005071c60057640a007d0c007c03007c0b0066020053280b0000004e690700000069000000006902000000675839b4c876bedf3f6901000000674e62105839b4d03f678d976e1283c0d23f692d000000690c0000006903000000280d000000740500000046616c736574050000004e53495a45740a0000006269745f6c656e67746874040000005472756574060000006e756d626572740e0000006765745374726f6e675072696d657403000000616273740e00000067657452616e646f6d52616e67657403000000706f777403000000696e74740700000069735072696d6574030000004743447407000000696e7665727365280d00000074010000007874050000004e73697a657406000000707173697a6574010000004e740100000070740100000071740300000070686974060000006c696d69743174060000006c696d697432740100000064740300000070707074010000006574030000007a7a7a2800000000280000000073150000002f6f726967696e616c6368616c6c656e67652e7079740a0000006372656174655f6b657917000000733e000000000106010a010d0206010a010601150109010f010f04200108010e0112020601060109013c0119010601120135020e011801060112011801060105020604"
>>> code=marshal.loads(data)
>>> code=marshal.loads(data.decode('hex'))
>>> import dis
>>> dis.dis(code)
 24           0 LOAD_GLOBAL              0 (False)
              3 POP_JUMP_IF_FALSE       29

 25           6 LOAD_CONST               1 (7)
              9 LOAD_CONST               2 (0)
             12 BINARY_DIVIDE
             13 STOP_CODE
             14 STOP_CODE
             15 STOP_CODE
...
 56         428 LOAD_GLOBAL              4 (number)
            431 LOAD_ATTR               11 (GCD)
            434 LOAD_FAST               11 (e)
            437 LOAD_FAST                6 (phi)
            440 CALL_FUNCTION            2
            443 LOAD_CONST               5 (1)
            446 COMPARE_OP               3 (!=)
            449 POP_JUMP_IF_FALSE      458
...
```

We can basically guess that this is generating n, e, d, which is also consistent with our initial expectation. Let's directly decompile it:

```python
>>> from uncompyle6 import code_deparse
>>> code_deparse(code)
Instruction context:

  25       6  LOAD_CONST            1  7
              9  LOAD_CONST            2  0
             12  BINARY_DIVIDE
->           13  STOP_CODE
             14  STOP_CODE
             15  STOP_CODE
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python2.7/site-packages/uncompyle6/semantics/pysource.py", line 2310, in code_deparse
    deparsed.ast = deparsed.build_ast(tokens, customize, isTopLevel=isTopLevel)
  File "/usr/local/lib/python2.7/site-packages/uncompyle6/semantics/pysource.py", line 2244, in build_ast
    raise ParserError(e, tokens)
uncompyle6.semantics.parser_error.ParserError: --- This code section failed: ---
...
 64     469  LOAD_FAST             3  'N'
         472  LOAD_FAST            11  'e'
         475  BUILD_TUPLE_2         2  None
         478  RETURN_VALUE
          -1  RETURN_LAST

Parse error at or near `STOP_CODE' instruction at offset 13
```

We can see the STOP_CODE, which is suspicious. If we look carefully at the initial disassembly, we can see that the code at the beginning is obfuscation:

```python
>>> dis.dis(code)
 24           0 LOAD_GLOBAL              0 (False)
              3 POP_JUMP_IF_FALSE       29

 25           6 LOAD_CONST               1 (7)
              9 LOAD_CONST               2 (0)
             12 BINARY_DIVIDE
             13 STOP_CODE
             14 STOP_CODE
             15 STOP_CODE

 26          16 STOP_CODE
             17 POP_TOP
             18 STOP_CODE
             19 LOAD_CONST               2 (0)
             22 BINARY_DIVIDE
             23 STORE_FAST               0 (x)
             26 JUMP_FORWARD             0 (to 29)

 28     >>   29 LOAD_GLOBAL              1 (NSIZE)
             32 STORE_FAST               1 (Nsize)

 29          35 LOAD_FAST                1 (Nsize)
             38 LOAD_CONST               3 (2)
             41 BINARY_DIVIDE
             42 STORE_FAST               2 (pqsize)
```

Up to:

```python
 29          35 LOAD_FAST                1 (Nsize)
```

Everything before this has no effect. It seems like the challenge author intentionally modified the code. Analyzing this part of the code carefully, it appears to be two parts:

```python
# part 1
 25           6 LOAD_CONST               1 (7)
              9 LOAD_CONST               2 (0)
             12 BINARY_DIVIDE
             13 STOP_CODE
             14 STOP_CODE
             15 STOP_CODE
# part 2
 26          16 STOP_CODE
             17 POP_TOP
             18 STOP_CODE
             19 LOAD_CONST               2 (0)
             22 BINARY_DIVIDE
             23 STORE_FAST               0 (x)
             26 JUMP_FORWARD             0 (to 29)
```

These are exactly lines 25 and 26. By guessing, both seem to be `x=7/0`, so we try to fix this part of the code. Next, we need to locate this part. According to the manual, STOP_CODE is 0, so we can locate lines 25-26 as t[6:26], each being 10 bytes (6-15, 16-25).

```python
>>> t=code.co_code
>>> t
't\x00\x00r\x1d\x00d\x01\x00d\x02\x00\x15\x00\x00\x00\x00\x01\x00d\x02\x00\x15}\x00\x00n\x00\x00t\x01\x00}\x01\x00|\x01\x00d\x03\x00\x15}\x02\x00d\x02\x00}\x03\x00xo\x00|\x03\x00j\x02\x00\x83\x00\x00|\x01\x00k\x03\x00r\xa4\x00xL\x00t\x03\x00r\x96\x00t\x04\x00j\x05\x00|\x02\x00\x83\x01\x00}\x04\x00t\x04\x00j\x05\x00|\x02\x00\x83\x01\x00}\x05\x00t\x06\x00|\x04\x00|\x05\x00\x18\x83\x01\x00j\x02\x00\x83\x00\x00|\x01\x00d\x04\x00\x14k\x04\x00rK\x00PqK\x00qK\x00W|\x04\x00|\x05\x00\x14}\x03\x00q6\x00W|\x04\x00d\x05\x00\x18|\x05\x00d\x05\x00\x18\x14}\x06\x00d\x06\x00}\x07\x00d\x07\x00}\x08\x00x\t\x01t\x03\x00r\xce\x01t\x04\x00j\x07\x00t\x08\x00d\x03\x00t\t\x00|\x01\x00|\x07\x00\x14\x83\x01\x00\x83\x02\x00t\x08\x00d\x03\x00t\t\x00|\x01\x00|\x07\x00\x14\x83\x01\x00d\x05\x00\x17\x83\x02\x00\x83\x02\x00}\t\x00xq\x00|\t\x00j\x02\x00\x83\x00\x00|\x01\x00|\x08\x00\x14k\x00\x00r{\x01d\x02\x00}\n\x00xD\x00t\x04\x00j\n\x00|\n\x00\x83\x01\x00sm\x01t\x04\x00j\x07\x00t\x08\x00d\x03\x00d\x08\x00\x83\x02\x00t\x08\x00d\x03\x00d\x08\x00\x83\x02\x00t\x08\x00d\x03\x00d\t\x00\x83\x02\x00\x17\x83\x02\x00}\n\x00q*\x01W|\t\x00|\n\x009}\t\x00q\x0b\x01Wt\x04\x00j\x0b\x00|\t\x00|\x06\x00\x83\x02\x00d\x05\x00k\x03\x00r\x9a\x01q\xc6\x00n\x00\x00t\x04\x00j\x0c\x00|\t\x00|\x06\x00\x83\x02\x00}\x0b\x00t\x04\x00j\x0b\x00|\x0b\x00|\x06\x00\x83\x02\x00d\x05\x00k\x03\x00r\xca\x01q\xc6\x00n\x00\x00Pq\xc6\x00Wd\n\x00}\x0c\x00|\x03\x00|\x0b\x00f\x02\x00S'
>>> t[6:26]
'd\x01\x00d\x02\x00\x15\x00\x00\x00\x00\x01\x00d\x02\x00\x15}\x00\x00'
>>> t[-3:]
'\x02\x00S'
>>> t='d\x01\x00d\x02\x00\x15\x00\x00\x00\x00\x01\x00d\x02\x00\x15}\x00\x00'
>>> t[-3:]
'}\x00\x00'
>>> t[:7]+t[-3:]
'd\x01\x00d\x02\x00\x15}\x00\x00'
>>> _.encode('hex')
'640100640200157d0000'
```

Thus we can fix the original code:

```python
>>> data.find('640100')
56
>>> data1=data[:56]+'640100640200157d0000640100640200157d0000'+data[56+40:]
>>> code1=marshal.loads(data1.decode('hex'))
>>> code_deparse(code1)
if False:
    x = 7 / 0
    x = 7 / 0
Nsize = NSIZE
pqsize = Nsize / 2
N = 0
while N.bit_length() != Nsize:
    while True:
        p = number.getStrongPrime(pqsize)
        q = number.getStrongPrime(pqsize)
        if abs(p - q).bit_length() > Nsize * 0.496:
            break

    N = p * q

phi = (p - 1) * (q - 1)
limit1 = 0.261
limit2 = 0.293
while True:
    d = number.getRandomRange(pow(2, int(Nsize * limit1)), pow(2, int(Nsize * limit1) + 1))
    while d.bit_length() < Nsize * limit2:
        ppp = 0
        while not number.isPrime(ppp):
            ppp = number.getRandomRange(pow(2, 45), pow(2, 45) + pow(2, 12))

        d *= ppp

    if number.GCD(d, phi) != 1:
        continue
    e = number.inverse(d, phi)
    if number.GCD(e, phi) != 1:
        continue
    break

zzz = 3
return (
 N, e)<uncompyle6.semantics.pysource.SourceWalker object at 0x10a0ea110>
```

We can see that the generated d intentionally exceeds 0.292. However, we can observe that ppp has a very small range — in fact, through testing we can determine that there are 125 primes in this range. And:

```python
1280*0.261+45=379.08000000000004>375.03999999999996=1280*0.293
```

So actually only one number is multiplied here. We can enumerate what was multiplied and modify e1=e*ppp, which essentially reduces to the standard Boneh and Durfee attack.

However, if we directly use the script from https://github.com/mimoo/RSA-and-LLL-attacks, it won't work either. We need to increase m, basically to 8, and even then it's not very stable.

If you experiment carefully, you'll find that e1>N. This doesn't seem like a big deal, but the original script assumes e<N, so we need to appropriately modify the estimated bounds:

```python
    X = 2*floor(N^delta)  # this _might_ be too much
    Y = floor(N^(1/2))    # correct if p, q are ~ same size
```

Based on the derivation above, the bounds should be:

$|k|<\frac{2ed}{\varphi(N)}<\frac{3ed}{N}=3*\frac{e}{N}*d<3*\frac{e}{N}*N^{delta}$

$|y|<2*N^{0.5}$

In the end, the main modifications were to m and the upper bound of X:

```python
    delta = .262 # this means that d < N^delta

    #
    # Lattice (tweak those values)
    #

    # you should tweak this (after a first run), (e.g. increment it until a solution is found)
    m = 8 # size of the lattice (bigger the better/slower)

    # you need to be a lattice master to tweak these
    t = int((1-2*delta) * m)  # optimization from Herrmann and May
    X = floor(3*e/N*N^delta) #4*floor(N^delta)  # this _might_ be too much
    Y = floor(2*N^(1/2))    # correct if p, q are ~ same size
```

Finally, the result is obtained:

```shell
[DEBUG] Received 0x1f bytes:
    'Succcess!\n'
    'OOO{Br3akingL!mits?}\n'
OOO{Br3akingL!mits?}
```

It must be said that this challenge really requires a **multi**-core server.


## References

- Survey: Lattice Reduction Attacks on RSA
- An Introduction to Coppersmith's method and Applications in Cryptology
