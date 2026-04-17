# Knapsack Encryption

## The Knapsack Problem

First, let's introduce the knapsack problem. Suppose a knapsack can hold weight W, and there are n items with weights $a_1, a_2,...,a_n$ respectively. We want to ask which items can be packed to exactly fill the knapsack, and each item can only be used once. This is essentially solving the following problem:

$$
x_1a_1+x_2a_2+,...,+x_na_n=W
$$

where all $x_i$ can only be 0 or 1. Obviously, we must enumerate all combinations of the n items to solve this problem, and the complexity is $2^n$. This is exactly what makes knapsack encryption clever.

During encryption, if the plaintext we want to encrypt is x, we can represent it as an n-bit binary number, then multiply each bit by $a_i$ to obtain the encrypted result.

But what about decryption? We have indeed made it difficult for others to decrypt the ciphertext, but we ourselves also have no way to decrypt it.

However, when $a_i$ is superincreasing, we do have a way to solve it. A superincreasing sequence satisfies the following condition:

$$
a_i>\sum_{k=1}^{i-1}a_k
$$

That is, the i-th number is greater than the sum of all preceding numbers.

Why can we decrypt when this condition is satisfied? Because if the encrypted result is greater than $a_n$, its corresponding coefficient must be 1. Otherwise, the equation cannot hold no matter what. Therefore, we can immediately obtain the corresponding plaintext.

However, this introduces another problem: since $a_i$ is public, if an attacker intercepts the ciphertext, they can easily break such a cipher. To address this issue, the Merkle–Hellman encryption algorithm was developed. We can use the initial knapsack set as the private key, the transformed knapsack set as the public key, and slightly modify the encryption process.

Although we mentioned the superincreasing sequence here, we didn't explain how it is generated.

## Merkle–Hellman

### Public and Private Key Generation

#### Generating the Private Key

The private key is our initial knapsack set. Here we use a superincreasing sequence. How do we generate it? We can assume $a_1=1$, then $a_2$ just needs to be greater than 1, and similarly we can generate subsequent values in order.

#### Generating the Public Key

The process of generating the public key mainly uses modular multiplication.

First, we generate the modulus m for the modular multiplication, ensuring that

$$
m>\sum_{i=1}^{i=n}a_i
$$

Next, we choose the multiplier w for the modular multiplication as a private key and ensure that

$$
gcd(w,m)=1
$$

Then, we can generate the public key using the following formula:

$$
b_i \equiv w a_i \bmod m
$$

The new knapsack set $b_i$ and m serve as the public key.

### Encryption and Decryption

#### Encryption

Suppose the plaintext we want to encrypt is v, and each bit is $v_i$. Then our encrypted result is

$$
\sum_{i=1}^{i=n}b_iv_i \bmod m
$$

#### Decryption

For the decryptor, first compute the modular inverse $w^{-1}$ of w with respect to m.

Then we can multiply the received ciphertext by $w^{-1}$ to obtain the plaintext, because

$$
\sum_{i=1}^{i=n}w^{-1}b_iv_i \bmod m=\sum_{i=1}^{i=n}a_iv_i \bmod m
$$

This is because

$$
b_i \equiv w a_i \bmod m
$$

Since each encrypted message block is less than m, the result obtained is naturally the plaintext.

### Breaking the Cipher

This cryptosystem was broken two years after it was proposed. The basic idea of the attack is that we don't necessarily need to find the correct multiplier w (i.e., the trapdoor information). We only need to find any modulus `m′` and multiplier `w′` such that when we use `w′` to multiply the public knapsack vector B, it produces a superincreasing knapsack vector.

### Example

Here we use the Archaic challenge from 2014 ASIS Cyber Security Contest Quals as an example. [Problem link](https://github.com/ctfs/write-ups-2014/tree/b02bcbb2737907dd0aa39c5d4df1d1e270958f54/asis-ctf-quals-2014/archaic).

First, let's look at the source program:

```python
secret = 'CENSORED'
msg_bit = bin(int(secret.encode('hex'), 16))[2:]
```

First, all binary bits of the secret are obtained.

Then, the keypair is obtained using the following function, containing the public key and private key:

```python
keyPair = makeKey(len(msg_bit))
```

Let's carefully analyze the makekey function:

```python
def makeKey(n):
	privKey = [random.randint(1, 4**n)]
	s = privKey[0]
	for i in range(1, n):
		privKey.append(random.randint(s + 1, 4**(n + i)))
		s += privKey[i]
	q = random.randint(privKey[n-1] + 1, 2*privKey[n-1])
	r = random.randint(1, q)
	while gmpy2.gcd(r, q) != 1:
		r = random.randint(1, q)
	pubKey = [ r*w % q for w in privKey ]
	return privKey, q, r, pubKey
```

We can see that prikey is a superincreasing sequence, and the obtained q is larger than the sum of all numbers in prikey. Furthermore, the obtained r is coprime to q. All of this indicates that the encryption is a knapsack cipher.

Indeed, the encryption function multiplies each bit of the message by the corresponding public key and sums them up:

```python
def encrypt(msg, pubKey):
	msg_bit = msg
	n = len(pubKey)
	cipher = 0
	i = 0
	for bit in msg_bit:
		cipher += int(bit)*pubKey[i]
		i += 1
	return bin(cipher)[2:]
```

For the cracking script, we directly use the script from [GitHub](https://github.com/ctfs/write-ups-2014/tree/b02bcbb2737907dd0aa39c5d4df1d1e270958f54/asis-ctf-quals-2014/archaic) with some simple modifications.

```python
import binascii
# open the public key and strip the spaces so we have a decent array
fileKey = open("pub.Key", 'rb')
pubKey = fileKey.read().replace(' ', '').replace('L', '').strip('[]').split(',')
nbit = len(pubKey)
# open the encoded message
fileEnc = open("enc.txt", 'rb')
encoded = fileEnc.read().replace('L', '')
print "start"
# create a large matrix of 0's (dimensions are public key length +1)
A = Matrix(ZZ, nbit + 1, nbit + 1)
# fill in the identity matrix
for i in xrange(nbit):
    A[i, i] = 1
# replace the bottom row with your public key
for i in xrange(nbit):
    A[i, nbit] = pubKey[i]
# last element is the encoded message
A[nbit, nbit] = -int(encoded)

res = A.LLL()
for i in range(0, nbit + 1):
    # print solution
    M = res.row(i).list()
    flag = True
    for m in M:
        if m != 0 and m != 1:
            flag = False
            break
    if flag:
        print i, M
        M = ''.join(str(j) for j in M)
        # remove the last bit
        M = M[:-1]
        M = hex(int(M, 2))[2:-1]
		print M
```

After the output, decode it:

```python
295 [1, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0, 0, 1, 0, 0, 1, 0, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 0, 0, 1, 0, 1, 1, 0, 0, 0, 1, 0, 0, 1, 1, 0, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 0, 0, 1, 1, 0, 1, 0, 1, 0, 1, 1, 0, 0, 1, 1, 0, 0, 1, 1, 0, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 1, 0, 0, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 1, 0, 0, 0, 1, 1, 0, 0, 1, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 1, 0, 0, 1, 1, 0, 0, 0, 1, 1, 0, 0, 1, 1, 0, 0, 0, 1, 0, 0, 1, 1, 1, 0, 0, 1, 0, 0, 1, 1, 0, 1, 0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0]
415349535f3962643364356664323432323638326331393536383830366130373036316365
>>> import binascii
>>> binascii.unhexlify('415349535f3962643364356664323432323638326331393536383830366130373036316365')
'ASIS_9bd3d5fd2422682c19568806a07061ce'
```

Note that in the matrix res obtained from the LLL attack, only the rows containing exclusively 0 and 1 values are the results we want, because when encrypting the plaintext, we decompose it into a binary bit string. Furthermore, we need to remove the last digit of that row.

The flag is `ASIS_9bd3d5fd2422682c19568806a07061ce`.

### Challenges

- 2017 National Competition classic
