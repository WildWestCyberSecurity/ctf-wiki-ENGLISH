# CTR

CTR stands for Counter mode, designed by Diffie and Hellman.

## Encryption

![](./figure/ctr_encryption.png)

## Decryption

![](./figure/ctr_decryption.png)

## Characteristics

| Property                          | Description     |
| ----------------------------- | -------- |
| Encryption parallelizable | Yes |
| Decryption parallelizable | Yes |
| Random read access   | Yes |

CTR mode has some advantages over OFB mode. One advantage is that it allows parallel encryption and decryption of multiple blocks, because the key stream can be independently computed for each block. This can improve the performance and efficiency of the encryption and decryption process.

## 2023 CTF Challenge

Challenge description:

```python
from Crypto.Util.number import long_to_bytes, bytes_to_long
from Crypto.Cipher import AES
from Crypto.Util import Counter
from hashlib import sha256
import os
from secret import flag

def padding(msg):
    return msg + os.urandom(16 - len(msg) % 16)

msg = b"where is the flag? Key in my Heart/Counter!!!!"
key = b"I w0nder how????"

assert len(msg) == 46
assert len(key) == 16

enc_key = os.urandom(16)
initial_value = bytes_to_long(enc_key)
hash = sha256(str(initial_value).encode()).hexdigest()

aes = AES.new(enc_key,AES.MODE_ECB)
enc_flag = aes.encrypt(padding(flag))

ctr = Counter.new(AES.block_size * 8, initial_value = initial_value) 
aes = AES.new(key, counter = ctr, mode = AES.MODE_CTR)
enc = aes.encrypt(msg)

print("enc = {}".format(enc[-16:]))
print("enc_flag = {}".format(enc_flag))
print("hash = {}".format(hash))

"""
enc_last16 = b'\xbe\x9bd\xc6\xd4=\x8c\xe4\x95bi\xbc\xe01\x0e\xb8'
enc_flag = b'\xb2\x97\x83\x1dB\x13\x9b\xc2\x97\x9a\xa6+M\x19\xd74\xd2-\xc0\xb6\xba\xe8ZE\x0b:\x14\xed\xec!\xa1\x92\xdfZ\xb0\xbd\xb4M\xb1\x14\xea\xd8\xee\xbf\x83\x16g\xfa'
hash = efb07225b3f1993113e104757210261083c79de50f577b3f0564368ee7b25eeb
"""
```

We can see that: the `flag` is first encrypted using ECB mode, where `key1` is unknown, and the SHA256 hash result of converting `key1` bytes to an integer is also provided.

The bytes-to-integer conversion of `key1` is set as the initial value of the `Counter` in CTR mode, and then this `Counter` is used as a parameter to encrypt the plaintext with `key2`, with the last 16 bytes of ciphertext given. Among these: the plaintext, `key2`, and the last block of ciphertext (which actually needs further processing) are all known to us.

Our goal now is to reverse-derive the `Counter` initial value based on the known conditions.

Let's review the CTR mode encryption process:

![](./figure/ctr_encryption.png)

To obtain the `Counter`, we first need to get the result produced by the encryptor. During encryption: $plaintext \oplus E(Counter) = ciphertext$, and based on XOR properties, $E(Counter)$ is $plaintext \oplus ciphertext$.

Now we only need to convert $E(Counter)$ to $Counter$.

Here we should not let the CTR mode encryption/decryption diagrams limit our thinking. In fact, looking at just this last part, it can be understood as a single block in ECB mode, so decryption is: $D(E(Counter)) = Counter$.

Then we need to subtract from $Counter$ the amount the counter was incremented during the encryption process to get the final result.

There are also some pitfalls which can be seen in the comments of the following exploit:

```python
from Crypto.Util.number import long_to_bytes, bytes_to_long
from Crypto.Cipher import AES
from Crypto.Util import Counter
from hashlib import sha256
import os
# from secret import flag
flag = b'flag{test}'

def padding(msg):
    return msg + os.urandom(16 - len(msg) % 16)  # 随机值填充

msg = b"where is the flag? Key in my Heart/Counter!!!!"
key = b"I w0nder how????"

assert len(msg) == 46
assert len(key) == 16

enc_key = os.urandom(16)  # 随机key
initial_value = bytes_to_long(enc_key) # key转为整数
hash = sha256(str(initial_value).encode()).hexdigest()  # 字符串(key) 的 sha256

aes = AES.new(enc_key,AES.MODE_ECB) 
enc_flag = aes.encrypt(padding(flag))

                # 16 * 8 = 128,
# {'counter_len': 16, 'prefix': b'', 'suffix': b'', 'initial_value': 1, 'little_endian': False}
ctr = Counter.new(AES.block_size * 8, initial_value = initial_value) 
print(ctr)
aes = AES.new(key, counter = ctr, mode = AES.MODE_CTR)  # key 已知, 推 counter, CTR mode 不需要 padding
enc = aes.encrypt(msg)  # msg 已知


# print("enc = {}".format(len(enc)))  # 46
print("enc = {}".format(enc[-16:]))  # 密文的最后16位, 但并不是最后一个 block
print("enc_flag = {}".format(enc_flag))
print("hash = {}".format(hash))
print('题目数据输出结束' + ' *' * 16)
# Data
enc_last16 = b'\xbe\x9bd\xc6\xd4=\x8c\xe4\x95bi\xbc\xe01\x0e\xb8'
enc_flag = b'\xb2\x97\x83\x1dB\x13\x9b\xc2\x97\x9a\xa6+M\x19\xd74\xd2-\xc0\xb6\xba\xe8ZE\x0b:\x14\xed\xec!\xa1\x92\xdfZ\xb0\xbd\xb4M\xb1\x14\xea\xd8\xee\xbf\x83\x16g\xfa'
hash = 'efb07225b3f1993113e104757210261083c79de50f577b3f0564368ee7b25eeb'

# Solution
# a = msg[32:]  # 从明文index 32 开始
a = msg[16 * (len(msg) // 16):]  # 取最后一个 block
b = enc_last16[16 - (len(enc) % 16):]  # 从密文index 2 开始 | 选最后一个 block
# 加密最后步骤 明文 xor enc_{key}(counter) = 密文
# 解密最后步骤 enc_{key}(counter) xor 密文 = 明文 | enc_{key}(counter) = 密文 xor 明文
enc_Counter1 = bytes(a[i] ^ b[i] for i in range(14))  
for i in range(0xff):
    for j in range(0xff):
        # ECB mode 要求数据长度与块长对齐, 而加密后的数据的最后 2 bytes 我们并不清楚, 所以我们需要尝试所有的可能
        enc_Counter2 = enc_Counter1 + bytes([i]) + bytes([j])
        aes = AES.new(key,AES.MODE_ECB)
        Counter = aes.decrypt(enc_Counter2)  # E_{key}(Counter) = Counter_enc | Counter = D_{key}(Counter_enc)
        initial_value = bytes_to_long(Counter) - (len(msg) // 16)  # 经历两个 block, 最后一个 block 的 Counter - block 数 = 初始值
        if hash == sha256(str(initial_value).encode()).hexdigest():  # type: str
            print(f'found {initial_value = }')
            enc_key = long_to_bytes(initial_value)
            aes = AES.new(enc_key,AES.MODE_ECB)
            flag = aes.decrypt(enc_flag)
            print(flag)
            break
# flag{9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d}
```

## Challenges

- 2017 star ctf ssss
- 2017 star ctf ssss2
