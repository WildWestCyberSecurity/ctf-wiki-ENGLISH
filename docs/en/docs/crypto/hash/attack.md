# Hash Attack

The common attack methods against hash functions mainly include:

-  Brute force attack: Does not depend on any algorithm details, only related to the hash value length;
   - Birthday Attack: Does not exploit the structure or any algebraic weaknesses of the hash function, only depends on the length of the message digest, i.e., the length of the hash value.
   - Meet-In-The-Middle Attack: A variant of the birthday attack that does not compare hash values, but instead compares intermediate variables. This attack is mainly applicable to hash schemes with a block chaining structure.
-  Cryptanalysis: Depends on the design flaws of the specific algorithm.

## Brute Force Attack

**HashCat** is arguably the best software for cracking hashes based on CPU and GPU. Related links are as follows:

[HashCat Official Website](http://www.hashcat.net/hashcat/)

[HashCat Basic Usage](http://www.freebuf.com/sectool/112479.html)

## Hash Length Extension Attacks
### Introduction

The basic definition is as follows, sourced from [Wikipedia](https://zh.wikipedia.org/wiki/%E9%95%BF%E5%BA%A6%E6%89%A9%E5%B1%95%E6%94%BB%E5%87%BB).

Hash Length Extension Attacks are attack methods targeting certain cryptographic hash functions that allow inclusion of additional information. This attack is applicable when **the lengths of the message and key are known**, for all hash functions constructed as H(key ∥ message). Algorithms based on the Merkle–Damgård construction, such as MD5 and SHA-1, are vulnerable to this type of attack.

These types of hash functions have the following characteristics:

- The message padding methods are quite similar. First, a bit 1 is appended after the message, then several 0 bits are padded until the total length is congruent to 448 modulo 512, and finally the 64-bit message length (before padding) is appended.
- The chaining variable obtained from each block is used as the initial vector IV for the next execution of the hash function. Only in the last block is the corresponding chaining variable converted to the hash value.

The following conditions should generally be met when performing an attack:

- We know the length of the key. If unknown, it needs to be brute-forced.
- We can control the message content.
- We already know the hash value of a message that contains the key.

This way, we can obtain a pair (message, x) satisfying x = H(key ∥ message), even though we do not know the content of the key.

### Attack Principle

Let's assume we know the hash value of hash(key+s), where s is known. During computation, padding will necessarily be performed. So we can first obtain the extended string of key+s, namely:

now=key|s|padding

If we append additional information extra after now, i.e.:

key|s|padding|extra

Then when computing the hash value:

1. Extra will be padded until the conditions are met.
2. First, the chaining variable IV1 corresponding to now is computed, and we already know the hash value of this part. Since the algorithm that produces the hash value from the chaining variable is reversible, we can obtain the chaining variable.
3. Next, the hash algorithm is applied to the extra part using the obtained chaining variable IV1, and the hash value is returned.

Since we already know the hash value of the first part and we also know the value of extra, we can obtain the final hash value.

As mentioned earlier, we can control the value of message. So in fact, s, padding, and extra are all under our control. Therefore, we can naturally find the corresponding (message, x) satisfying x = hash(key|message).

### Examples

It seems most examples are from web challenges. Since I'm not very familiar with web, I won't provide examples for now.

### Tools

- [hashpump](https://github.com/bwall/HashPump)

For usage instructions, please refer to the readme on GitHub.

## Flawed Hash Algorithm Design
Some custom hash algorithms may be reversible.

### Hashinator
The logic of the challenge is simple: a `password` is selected from the well-known password dictionary "rockyou", and then hashed for 32 rounds using multiple randomly chosen hash algorithms. We need to crack the original `password` from the final hash result.

#### Analysis
The hash algorithms used in the challenge are: `md5`, `sha1`, `blake`, `scrypt`.
The key code is as follows:
```python
    password = self.generate_password()     # from rock_you.txt
    salt = self.generate_salt(password)     # 与password的长度有关
    hash_rounds = self.generate_rounds()    # 生成进行hash算法的顺序
    password_hash = self.calculate_hash(salt + password, hash_rounds)
```
1. The program first randomly selects a `password` from `rockyou.txt` as the plaintext for encryption.
2. Then generates a `salt` of length `128 - len(password)` based on the selected `password`'s length.
3. Selects from the 4 hash algorithms listed above to form 32 rounds of hashing operations.
4. Computes the final `password_hash` given to us based on the previously obtained `password` and `salt`.

Obviously, it is impossible to complete the challenge by reversing the hash algorithms.
We know all possible plaintexts, so first consider whether we can complete the enumeration by constructing a rainbow table. However, note that in the `generate_salt()` function, the combined length of `salt` and `password` exceeds 128 bytes, and it is commented:
```
    msize = 128 # f-you hashcat :D
```
So, we have to reluctantly give up on that approach.

In that case, there is only one possibility: the algorithm is reversible. Looking at the specific implementation of the `calculate_hash()` function, we can find the following suspicious code:
```python
for i in range(len(hash_rounds)):
    interim_salt = xor(interim_salt, hash_rounds[-1-i](interim_hash))
    interim_hash = xor(interim_hash, hash_rounds[i](interim_salt))
final_hash = interim_salt + interim_hash
```
Let's reorganize the information we know:
1. hash_rounds stores 32 rounds, i.e., the hash function handle to be used in each round.
2. final_hash is the hash result given to us.
3. The contents of hash_rounds are also printed to us after generation.
4. We want to obtain the values of `interim_salt` and `interim_hash` in the first round.
5. Both `interim_salt` and `interim_hash` have a length of 64 bytes.

Carefully observing the computation method of `interim_salt` and `interim_hash`, we can see that it is reversible.

$$
interim\_hash_1 = interim\_hash_2 \oplus hash\_rounds[i](interim\_salt_3)
$$

In this line of code, we already know $interim\_hash_1$ and $interim\_salt_3$, from which we can deduce the value of $interim\_hash_2$, which is the `interim_hash` from the previous round.
By reversing this method 32 times, we can obtain the original `password` and `salt`.

The specific decryption script is:
```python
import os
import hashlib
import socket
import threading
import socketserver
import struct
import time
import threading
# import pyscrypt
from base64 import b64encode, b64decode
from pwn import *
def md5(bytestring):
    return hashlib.md5(bytestring).digest()
def sha(bytestring):
    return hashlib.sha1(bytestring).digest()
def blake(bytestring):
    return hashlib.blake2b(bytestring).digest()
def scrypt(bytestring):
    l = int(len(bytestring) / 2)
    salt = bytestring[:l]
    p = bytestring[l:]
    return hashlib.scrypt(p, salt=salt, n=2**16, r=8, p=1, maxmem=67111936)
    # return pyscrypt.hash(p, salt, 2**16, 8, 1, dkLen=64)
def xor(s1, s2):
    return b''.join([bytes([s1[i] ^ s2[i % len(s2)]]) for i in range(len(s1))])
def main():
    # io = socket.socket(family=socket.AF_INET)
    # io.connect(('47.88.216.38', 20013))
    io = remote('47.88.216.38', 20013)
    print(io.recv(1000))
    ans_array = bytearray()
    while True:
        buf = io.recv(1)
        if buf:
            ans_array.extend(buf)
        if buf == b'!':
            break

    password_hash_base64 = ans_array[ans_array.find(b"b'") + 2: ans_array.find(b"'\n")]
    password_hash = b64decode(password_hash_base64)
    print('password:', password_hash)
    method_bytes = ans_array[
        ans_array.find(b'used:\n') + 6 : ans_array.find(b'\nYour')
    ]
    methods = method_bytes.split(b'\n')
    methods = [bytes(x.strip(b'- ')).decode() for x in methods]
    print(methods)
    in_salt = password_hash[:64]
    in_hash = password_hash[64:]
    for pos, neg in zip(methods, methods[::-1]):
        '''
            interim_salt = xor(interim_salt, hash_rounds[-1-i](interim_hash))
            interim_hash = xor(interim_hash, hash_rounds[i](interim_salt))
        '''
        in_hash = xor(in_hash, eval("{}(in_salt)".format(neg)))
        in_salt = xor(in_salt, eval("{}(in_hash)".format(pos)))
    print(in_hash, in_salt)
    print(in_hash[-20:])
    io.interactive()
main()

```

#### Original Hash Algorithm
```python

import os
import hashlib
import socket
import threading
import socketserver
import struct
import time

# import pyscrypt

from base64 import b64encode

def md5(bytestring):
    return hashlib.md5(bytestring).digest()

def sha(bytestring):
    return hashlib.sha1(bytestring).digest()

def blake(bytestring):
    return hashlib.blake2b(bytestring).digest()

def scrypt(bytestring):
    l = int(len(bytestring) / 2)
    salt = bytestring[:l]
    p = bytestring[l:]
    return hashlib.scrypt(p, salt=salt, n=2**16, r=8, p=1, maxmem=67111936)
    # return pyscrypt.hash(p, salt, 2**16, 8, 1)

def xor(s1, s2):
    return b''.join([bytes([s1[i] ^ s2[i % len(s2)]]) for i in range(len(s1))])

class HashHandler(socketserver.BaseRequestHandler):

    welcome_message = """
Welcome, young wanna-be Cracker, to the Hashinator.

To prove your worthiness, you must display the power of your cracking skills.

The test is easy:
1. We send you a password from the rockyou list, hashed using multiple randomly chosen algorithms.
2. You crack the hash and send back the original password.

As you already know the dictionary and won't need any fancy password rules, {} seconds should be plenty, right?

Please wait while we generate your hash...
    """

    hashes = [md5, sha, blake, scrypt]
    timeout = 10
    total_rounds = 32

    def handle(self):
        self.request.sendall(self.welcome_message.format(self.timeout).encode())

        password = self.generate_password()     # from rock_you.txt
        salt = self.generate_salt(password)     # 与password的长度有关
        hash_rounds = self.generate_rounds()    # 生成进行hash算法的顺序
        password_hash = self.calculate_hash(salt + password, hash_rounds)
        self.generate_delay()

        self.request.sendall("Challenge password hash: {}\n".format(b64encode(password_hash)).encode())
        self.request.sendall("Rounds used:\n".encode())
        test_rounds = []
        for r in hash_rounds:
            test_rounds.append(r)

        for r in hash_rounds:
            self.request.sendall("- {}\n".format(r.__name__).encode())
        self.request.sendall("Your time starts now!\n".encode())
        self.request.settimeout(self.timeout)
        try:
            response = self.request.recv(1024)
            if response.strip() == password:
                self.request.sendall("Congratulations! You are a true cracking master!\n".encode())
                self.request.sendall("Welcome to the club: {}\n".format(flag).encode())
                return
        except socket.timeout:
            pass
        self.request.sendall("Your cracking skills are bad, and you should feel bad!".encode())


    def generate_password(self):
        rand = struct.unpack("I", os.urandom(4))[0]
        lines = 14344391 # size of rockyou
        line = rand % lines
        password = ""
        f = open('rockyou.txt', 'rb')
        for i in range(line):
            password = f.readline()
        return password.strip()

    def generate_salt(self, p):
        msize = 128 # f-you hashcat :D
        salt_size = msize - len(p)
        return os.urandom(salt_size)

    def generate_rounds(self):
        rand = struct.unpack("Q", os.urandom(8))[0]
        rounds = []
        for i in range(self.total_rounds):
            rounds.append(self.hashes[rand % len(self.hashes)])
            rand = rand >> 2
        return rounds

    def calculate_hash(self, payload, hash_rounds):
        interim_salt = payload[:64]
        interim_hash = payload[64:]
        for i in range(len(hash_rounds)):
            interim_salt = xor(interim_salt, hash_rounds[-1-i](interim_hash))
            interim_hash = xor(interim_hash, hash_rounds[i](interim_salt))
            '''
            interim_hash = xor(
                interim_hash,
                hash_rounds[i](
                    xor(interim_salt, hash_rounds[-1-i](interim_hash))
                )
            )
            '''
        final_hash = interim_salt + interim_hash
        return final_hash

    def generate_delay(self):
        rand = struct.unpack("I", os.urandom(4))[0]
        time.sleep(rand / 1000000000.0)



class ThreadedTCPServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    allow_reuse_address = True

PORT = 1337
HOST = '0.0.0.0'
flag = ""

with open("flag.txt") as f:
    flag = f.read()

def main():
    server = ThreadedTCPServer((HOST, PORT), HashHandler)
    server_thread = threading.Thread(target=server.serve_forever)
    server_thread.start()
    server_thread.join()

if __name__ == "__main__":
    main()


```
