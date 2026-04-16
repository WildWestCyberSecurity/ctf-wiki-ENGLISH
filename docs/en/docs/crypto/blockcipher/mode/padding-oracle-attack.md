# Padding Oracle Attack

## Introduction

A Padding Oracle Attack generally requires the following conditions to be met:

- Encryption algorithm
    - Uses PKCS5 Padding. Of course, the OAEP padding scheme in asymmetric encryption may also be vulnerable.
    - The block cipher mode is CBC mode.
- Attacker capabilities
    - The attacker can intercept messages encrypted by the above encryption algorithm.
    - The attacker can interact with the padding oracle (i.e., the server): the client sends ciphertext to the server, and the server returns some information indicating whether the padding is correct.

The Padding Oracle Attack can achieve the following:

- Decrypt any given ciphertext without knowing the key or IV.

## Principle

The basic principle of the Padding Oracle Attack is as follows:

- For a long message, decrypt it block by block.
- For each block, first decrypt the last byte of the message, then the second-to-last byte, and so on.

Here, let us recall CBC mode:

- Encryption

$$
C_i=E_K(P_i \oplus C_{i-1})\\
C_0=IV
$$

- Decryption

$$
P_{i}=D_{K}(C_{i})\oplus C_{i-1}\\ C_{0}=IV
$$

We mainly focus on decryption, where we do not know the IV or key. Here we assume the ciphertext block length is n bytes.

Suppose we intercept the last two ciphertext blocks $F$ and $Y$. Let us use obtaining the last byte of the plaintext corresponding to ciphertext block $Y$ as an example for analysis. To obtain the decrypted content of $Y$, we first need to forge a ciphertext block $F'$ so that we can modify the last byte of the plaintext corresponding to $Y$'s decryption. This is because if we construct the ciphertext `F'|Y`, then decrypting $Y$ gives $P'=D_K(Y)\oplus F'$, so modifying the last byte $F'_{n}$ of ciphertext block $F'$ can modify the last byte $P'_n$ of the corresponding decrypted plaintext $P'$ of Y, and thus we can deduce the last byte of the original plaintext $P$. The process for obtaining the last byte of $P$ is as follows:

1. `i=0`, set each byte of $F'$ to a **random byte**.
2. Set $F'_n=i \oplus 0x01$.
3. Send `F'|Y` to the server. If the server does not report an error, then with high probability the last byte of $P'$ is 0x01. Otherwise, only when the last $P'_n \oplus i \oplus 0x01$ bytes of $P'$ are all equal to $P'_n \oplus i \oplus 0x01$ will there be no error. **Moreover, note that padding bytes can only range from 1 to n.** Therefore, under random $F'$ and the constraint on padding byte size, the probability of no error occurring is **very small**. So when the server does not report an error, we can assume that we have indeed obtained the correct byte. At this point, we know that the last byte of $D_k(Y)$, denoted $D_k(Y)_n$, is $P'_n \oplus F'_n = 0x01 \oplus i \oplus 0x01 = i$, which means the last byte of the original plaintext $P$ is $P_n = D_k(Y)_n \oplus F_n = i \oplus F_n$.
4. If an error occurs, set `i=i+1` and go to step 2.

After obtaining the last byte of $P$, we can continue to obtain the second-to-last byte of $P$. At this point, we need to set $F'_n=D_k(Y)_n\oplus 0x02$ and set $F_{n-1}=i \oplus 0x02$ to enumerate `i`. By repeating this process, we can obtain all bytes of the plaintext $P$ corresponding to Y.

In summary, the Padding Oracle Attack is essentially an attack method with a very high probability of success.

However, it should be noted that real-world problems often do not follow the standard Padding Oracle Attack pattern, and we often need to make some modifications.

## 2017 HITCON Secret Server

### Analysis

The encryption used in the program is AES CBC, with padding similar to PKCS5:

```python
def pad(msg):
    pad_length = 16-len(msg)%16
    return msg+chr(pad_length)*pad_length

def unpad(msg):
    return msg[:-ord(msg[-1])]
```

However, no validation is performed during each unpad — it directly performs the unpad operation.

Note that the functions interacting with the user are:

- `send_msg`: accepts the user's plaintext, encrypts it using the fixed IV `2jpmLoSsOlQrqyqE`, and outputs the encrypted result.
- `recv_msg`: accepts the user's IV and ciphertext, decrypts the ciphertext, and returns the result. Different operations are performed based on the returned result:

```python
            msg = recv_msg().strip()
            if msg.startswith('exit-here'):
                exit(0)
            elif msg.startswith('get-flag'):
                send_msg(flag)
            elif msg.startswith('get-md5'):
                send_msg(MD5.new(msg[7:]).digest())
            elif msg.startswith('get-time'):
                send_msg(str(time.time()))
            elif msg.startswith('get-sha1'):
                send_msg(SHA.new(msg[8:]).digest())
            elif msg.startswith('get-sha256'):
                send_msg(SHA256.new(msg[10:]).digest())
            elif msg.startswith('get-hmac'):
                send_msg(HMAC.new(msg[8:]).digest())
            else:
                send_msg('command not found')
```

### Main Vulnerability

Let us briefly summarize what we already have:

- Encryption
  - The encryption IV is fixed and known.
  - The encrypted result of 'Welcome!!'.
- Decryption
  - We can control the IV.

First, since we know the encrypted result of `Welcome!!` and can control the IV in recv_msg, then according to the decryption process:

$$
P_{i}=D_{K}(C_{i})\oplus C_{i-1}\\ C_{0}=IV
$$

If we input the encrypted result of `Welcome!!` into recv_msg, the direct decryption result is `(Welcome!!+'\x07'*7) xor iv`. If we **properly control the IV passed during decryption**, we can control the decrypted result. This means we can execute **any of the commands mentioned above**. Consequently, we can also learn the decrypted result of `flag`.

Furthermore, building on the above, if we append a custom IV and the encrypted result of Welcome after any ciphertext C as input to recv_msg, we can control the last byte of the decrypted message. **Due to the unpad operation, we can thus control the length of the decrypted message to decrease by 0 to 255**.

### Exploitation Approach

The basic exploitation approach is as follows:

1. Bypass proof of work.
2. Obtain the encrypted flag using the arbitrary command execution method.
3. Since the flag starts with `hitcon{`, which is 7 bytes, we can still control the IV to make the first 7 bytes of the decrypted message be specific bytes. This allows us to execute the `get-md5` command on the decrypted message. And due to the unpad operation, we can control the decrypted message to end exactly at a specific byte position. So we can start by controlling the decrypted message to be `hitcon{x`, i.e., keeping only one byte after `hitcon{`. This way we can obtain the encrypted result of the hash of a single byte. Similarly, we can obtain the encrypted result of hashes with a specified number of bytes.
4. With this, we can brute-force byte by byte locally, compute the corresponding `md5`, then use the arbitrary command execution method again to control the decrypted plaintext to be any specified command. If the control fails, the byte is wrong and needs to be brute-forced again; if correct, the corresponding command can be executed directly.

The specific code is as follows:

```python
#coding=utf-8
from pwn import *
import base64, time, random, string
from Crypto.Cipher import AES
from Crypto.Hash import SHA256, MD5
#context.log_level = 'debug'
if args['REMOTE']:
    p = remote('52.193.157.19', 9999)
else:
    p = remote('127.0.0.1', 7777)


def strxor(str1, str2):
    return ''.join([chr(ord(c1) ^ ord(c2)) for c1, c2 in zip(str1, str2)])


def pad(msg):
    pad_length = 16 - len(msg) % 16
    return msg + chr(pad_length) * pad_length


def unpad(msg):
    return msg[:-ord(msg[-1])]  # remove pad


def flipplain(oldplain, newplain, iv):
    """flip oldplain to new plain, return proper iv"""
    return strxor(strxor(oldplain, newplain), iv)


def bypassproof():
    p.recvuntil('SHA256(XXXX+')
    lastdata = p.recvuntil(')', drop=True)
    p.recvuntil(' == ')
    digest = p.recvuntil('\nGive me XXXX:', drop=True)

    def proof(s):
        return SHA256.new(s + lastdata).hexdigest() == digest

    data = pwnlib.util.iters.mbruteforce(
        proof, string.ascii_letters + string.digits, 4, method='fixed')
    p.sendline(data)
    p.recvuntil('Done!\n')


iv_encrypt = '2jpmLoSsOlQrqyqE'


def getmd5enc(i, cipher_flag, cipher_welcome):
    """return encrypt( md5( flag[7:7+i] ) )"""
    ## keep iv[7:] do not change, so decrypt won't change
    new_iv = flipplain("hitcon{".ljust(16, '\x00'), "get-md5".ljust(
        16, '\x00'), iv_encrypt)
    payload = new_iv + cipher_flag
    ## calculate the proper last byte number
    last_byte_iv = flipplain(
        pad("Welcome!!"),
        "a" * 15 + chr(len(cipher_flag) + 16 + 16 - (7 + i + 1)), iv_encrypt)
    payload += last_byte_iv + cipher_welcome
    p.sendline(base64.b64encode(payload))
    return p.recvuntil("\n", drop=True)


def main():
    bypassproof()

    # result of encrypted Welcome!!
    cipher = p.recvuntil('\n', drop=True)
    cipher_welcome = base64.b64decode(cipher)[16:]
    log.info("cipher welcome is : " + cipher_welcome)

    # execute get-flag
    get_flag_iv = flipplain(pad("Welcome!!"), pad("get-flag"), iv_encrypt)
    payload = base64.b64encode(get_flag_iv + cipher_welcome)
    p.sendline(payload)
    cipher = p.recvuntil('\n', drop=True)
    cipher_flag = base64.b64decode(cipher)[16:]
    flaglen = len(cipher_flag)
    log.info("cipher flag is : " + cipher_flag)

    # get command not found cipher
    p.sendline(base64.b64encode(iv_encrypt + cipher_welcome))
    cipher_notfound = p.recvuntil('\n', drop=True)

    flag = ""
    # brute force for every byte of flag
    for i in range(flaglen - 7):
        md5_indexi = getmd5enc(i, cipher_flag, cipher_welcome)
        md5_indexi = base64.b64decode(md5_indexi)[16:]
        log.info("get encrypt(md5(flag[7:7+i])): " + md5_indexi)
        for guess in range(256):
            # locally compute md5 hash
            guess_md5 = MD5.new(flag + chr(guess)).digest()
            # try to null out the md5 plaintext and execute a command
            payload = flipplain(guess_md5, 'get-time'.ljust(16, '\x01'),
                                iv_encrypt)
            payload += md5_indexi
            p.sendline(base64.b64encode(payload))
            res = p.recvuntil("\n", drop=True)
            # if we receive the block for 'command not found', the hash was wrong
            if res == cipher_notfound:
                print 'Guess {} is wrong.'.format(guess)
            # otherwise we correctly guessed the hash and the command was executed
            else:
                print 'Found!'
                flag += chr(guess)
                print 'Flag so far:', flag
                break


if __name__ == "__main__":
    main()

```

The final result is as follows:

```Shell
Flag so far: Paddin9_15_ve3y_h4rd__!!}\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10\x10
```

## 2017 HITCON Secret Server Revenge

### Description

```
The password of zip is the flag of "Secret Server"
```

### Analysis

This program is a continuation of the one above, but with some simple modifications:

- The encryption IV is unknown, but it can be deduced from the encrypted Welcome message.
- The program has an additional 56-byte token.
- The program allows at most 340 operations, so the brute-force approach from above is clearly infeasible.

The general flow of the program is as follows:

1. Pass the proof of work.
2. Send the encrypted "Welcome!!" message.
3. Within 340 operations, guess the token value correctly, and the flag will be automatically output.

### Vulnerability

The vulnerabilities from the previous challenge still exist in this one:

1. Arbitrary command execution
2. Length truncation

### Exploitation Approach

Due to the 340-operation limit, although we can still obtain the encrypted value of `md5(token[:i])` (**note that this encrypted portion is exactly 32 bytes: the first 16 bytes are the encrypted md5 value, and the last 16 bytes are purely the encrypted padding**. Here `md5(token[:i])` specifically refers to the first 16 bytes), we can no longer brute-force 256 times for each character.

Since brute-forcing is not feasible, is it possible to obtain one byte at a time? Let us review what information this program can potentially leak:

1. The encrypted value of certain messages' md5 hashes — here we can obtain the encrypted value of `md5(token[:i])`.
2. The unpad operation removes bytes from the decrypted message each time based on the last byte of the decrypted message. If we can compute the size of this byte, we may be able to determine the value of a byte.

Let us analyze the information leakage from unpad in more depth. If we place the encryption IV and `encrypt(md5(token[:i]))` after some ciphertext C, forming `C|IV|encrypt(md5(token[:i]))`, then the last plaintext block of the decrypted message is `md5(token[:i])`. Consequently, during unpad, the last byte of `md5(token[:i])` (0–255) is used for unpadding, and then the specified command (e.g., md5) is executed on the unpadded string. If we **pre-construct some samples of encrypted message hashes**, then compare the result of the above execution with the samples — if they match, we can basically determine the **last byte** of `md5(token[:i])`. However, if the last byte of `md5(token[:i])` is less than 16, then some bytes from the md5 value itself will be consumed during unpad, and these values are almost certainly different for different lengths of `token[:i]`. So special handling may be needed.

We have now identified the key to this problem: generating encrypted result samples corresponding to each unpad byte size for lookup.

The specific exploitation approach is as follows:

1. Bypass proof of work.
2. Obtain the encrypted result of the token `token_enc` — note that 7 bytes `"token: "` are prepended to the token before encryption. Thus the encrypted length is 64.
3. Sequentially obtain the results of `encrypt(md5(token[:i]))`, a total of 57, including the padding of the last token byte.
4. Construct samples corresponding to each unpad size. Here we construct the ciphertext `token_enc|padding|IV_indexi|welcome_enc`. Since `IV_indexi` is used to modify the last byte of the last plaintext block, this byte is constantly changing. If we want to obtain hash values of some fixed bytes, this part naturally cannot be added. Therefore, the unpad size range when generating samples is 17–255. If the last byte of `md5(token[:i])` is less than 17 during testing, some unknown samples will appear. A natural idea is to directly obtain 255-17+1 samples. However, doing so would exceed the 340-operation limit (255-17+1+57+56>340), so we clearly cannot obtain all bytes of the token. We therefore need to find a way to reuse some content. Here we choose to reuse the results of `encrypt(md5(token[:i]))`. When adding padding, we need to ensure that on one hand the operation count is sufficient, and on the other hand we can reuse previous results. Here we set the unpad loop from 17 to 208, and make it so that when the unpad size exceeds 208, it unpads exactly to a position we can reuse. Note that when the last byte of `md5(token[:i])` is 0, all decrypted plaintext is unpadded away, resulting in a "command not found" ciphertext.
5. Construct the ciphertext `token_enc|padding|IV|encrypt(md5(token[:i]))` again. During decryption, the last byte of `md5(token[:i])` is used for unpadding. If this byte is not less than 17 or is 0, it can be handled. If this byte is less than 17, then clearly the md5 result returned to the user is not within the sample range, so we flip its most significant bit to make the unpad result fall within the sample range. This way, we can guess the last byte of `md5(token[:i])`.
6. After guessing the last byte of `md5(token[:i])`, we can brute-force 256 times locally to find all characters whose hash ends with the last byte of `md5(token[:i])`.
7. However, in step 6, multiple candidate characters may be found for a given `md5(token[:i])`, since we only need the last byte to match.
8. So the question is: how do we eliminate excess candidate strings? Here we use a small trick: when enumerating bytes, we also enumerate the token's padding simultaneously. Since the padding is fixed at 0x01, we only need to filter out all tokens whose last byte is not 0x01.

Here, during testing, the `sleep` in the code was commented out to speed up interaction. The exploitation code is as follows:

```python
from pwn import *
import base64, time, random, string
from Crypto.Cipher import AES
from Crypto.Hash import SHA256, MD5
#context.log_level = 'debug'

p = remote('127.0.0.1', 7777)


def strxor(str1, str2):
    return ''.join([chr(ord(c1) ^ ord(c2)) for c1, c2 in zip(str1, str2)])


def pad(msg):
    pad_length = 16 - len(msg) % 16
    return msg + chr(pad_length) * pad_length


def unpad(msg):
    return msg[:-ord(msg[-1])]  # remove pad


def flipplain(oldplain, newplain, iv):
    """flip oldplain to new plain, return proper iv"""
    return strxor(strxor(oldplain, newplain), iv)


def bypassproof():
    p.recvuntil('SHA256(XXXX+')
    lastdata = p.recvuntil(')', drop=True)
    p.recvuntil(' == ')
    digest = p.recvuntil('\nGive me XXXX:', drop=True)

    def proof(s):
        return SHA256.new(s + lastdata).hexdigest() == digest

    data = pwnlib.util.iters.mbruteforce(
        proof, string.ascii_letters + string.digits, 4, method='fixed')
    p.sendline(data)


def sendmsg(iv, cipher):
    payload = iv + cipher
    payload = base64.b64encode(payload)
    p.sendline(payload)


def recvmsg():
    data = p.recvuntil("\n", drop=True)
    data = base64.b64decode(data)
    return data[:16], data[16:]


def getmd5enc(i, cipher_token, cipher_welcome, iv):
    """return encrypt( md5( token[:i+1] ) )"""
    ## keep iv[7:] do not change, so decrypt msg[7:] won't change
    get_md5_iv = flipplain("token: ".ljust(16, '\x00'), "get-md5".ljust(
        16, '\x00'), iv)
    payload = cipher_token
    ## calculate the proper last byte number
    last_byte_iv = flipplain(
        pad("Welcome!!"),
        "a" * 15 + chr(len(cipher_token) + 16 + 16 - (7 + i + 1)), iv)
    payload += last_byte_iv + cipher_welcome
    sendmsg(get_md5_iv, payload)
    return recvmsg()


def get_md5_token_indexi(iv_encrypt, cipher_welcome, cipher_token):
    md5_token_idxi = []
    for i in range(len(cipher_token) - 7):
        log.info("idx i: {}".format(i))
        _, md5_indexi = getmd5enc(i, cipher_token, cipher_welcome, iv_encrypt)
        assert (len(md5_indexi) == 32)
        # remove the last 16 byte for padding
        md5_token_idxi.append(md5_indexi[:16])
    return md5_token_idxi


def doin(unpadcipher, md5map, candidates, flag):
    if unpadcipher in md5map:
        lastbyte = md5map[unpadcipher]
    else:
        lastbyte = 0
    if flag == 0:
        lastbyte ^= 0x80
    newcandidates = []
    for x in candidates:
        for c in range(256):
            if MD5.new(x + chr(c)).digest()[-1] == chr(lastbyte):
                newcandidates.append(x + chr(c))
    candidates = newcandidates
    print candidates
    return candidates


def main():
    bypassproof()

    # result of encrypted Welcome!!
    iv_encrypt, cipher_welcome = recvmsg()
    log.info("cipher welcome is : " + cipher_welcome)

    # execute get-token
    get_token_iv = flipplain(pad("Welcome!!"), pad("get-token"), iv_encrypt)
    sendmsg(get_token_iv, cipher_welcome)
    _, cipher_token = recvmsg()
    token_len = len(cipher_token)
    log.info("cipher token is : " + cipher_token)

    # get command not found cipher
    sendmsg(iv_encrypt, cipher_welcome)
    _, cipher_notfound = recvmsg()

    # get encrypted(token[:i+1]),57 times
    md5_token_idx_list = get_md5_token_indexi(iv_encrypt, cipher_welcome,
                                              cipher_token)
    # get md5map for each unpadsize, 209-17 times
    # when upadsize>208, it will unpad ciphertoken
    # then we can reuse
    md5map = dict()
    for unpadsize in range(17, 209):
        log.info("get unpad size {} cipher".format(unpadsize))
        get_md5_iv = flipplain("token: ".ljust(16, '\x00'), "get-md5".ljust(
            16, '\x00'), iv_encrypt)
        ## padding 16*11 bytes
        padding = 16 * 11 * "a"
        ## calculate the proper last byte number, only change the last byte
        ## set last_byte_iv = iv_encrypted[:15] | proper byte
        last_byte_iv = flipplain(
            pad("Welcome!!"),
            pad("Welcome!!")[:15] + chr(unpadsize), iv_encrypt)
        cipher = cipher_token + padding + last_byte_iv + cipher_welcome
        sendmsg(get_md5_iv, cipher)
        _, unpadcipher = recvmsg()
        md5map[unpadcipher] = unpadsize

    # reuse encrypted(token[:i+1])
    for i in range(209, 256):
        target = md5_token_idx_list[56 - (i - 209)]
        md5map[target] = i

    candidates = [""]
    # get the byte token[i], only 56 byte
    for i in range(token_len - 7):
        log.info("get token[{}]".format(i))
        get_md5_iv = flipplain("token: ".ljust(16, '\x00'), "get-md5".ljust(
            16, '\x00'), iv_encrypt)
        ## padding 16*11 bytes
        padding = 16 * 11 * "a"
        cipher = cipher_token + padding + iv_encrypt + md5_token_idx_list[i]
        sendmsg(get_md5_iv, cipher)
        _, unpadcipher = recvmsg()
        # already in or md5[token[:i]][-1]='\x00'
        if unpadcipher in md5map or unpadcipher == cipher_notfound:
            candidates = doin(unpadcipher, md5map, candidates, 1)
        else:
            log.info("unpad size 1-16")
            # flip most significant bit of last byte to move it in a good range
            cipher = cipher[:-17] + strxor(cipher[-17], '\x80') + cipher[-16:]
            sendmsg(get_md5_iv, cipher)
            _, unpadcipher = recvmsg()
            if unpadcipher in md5map or unpadcipher == cipher_notfound:
                candidates = doin(unpadcipher, md5map, candidates, 0)
            else:
                log.info('oh my god,,,, it must be in...')
                exit()
    print len(candidates)
    # padding 0x01
    candidates = filter(lambda x: x[-1] == chr(0x01), candidates)
    # only 56 bytes
    candidates = [x[:-1] for x in candidates]
    print len(candidates)
    assert (len(candidates[0]) == 56)

    # check-token
    check_token_iv = flipplain(
        pad("Welcome!!"), pad("check-token"), iv_encrypt)
    sendmsg(check_token_iv, cipher_welcome)
    p.recvuntil("Give me the token!\n")
    p.sendline(base64.b64encode(candidates[0]))
    print p.recv()

    p.interactive()


if __name__ == "__main__":
    main()
```

The result is as follows:

```shell
...
79
1
hitcon{uNp@d_M3th0D_i5_am4Z1n9!}
```

## Teaser Dragon CTF 2018 AES-128-TSB

This challenge is quite interesting. The challenge description is as follows:

```
Haven't you ever thought that GCM mode is overcomplicated and there must be a simpler way to achieve Authenticated Encryption? Here it is!

Server: aes-128-tsb.hackable.software 1337

server.py
```

The attachment and final exploit can be found in the ctf-challenge repository.

The basic flow of the challenge is:

- Continuously receive two strings a and b, where a is plaintext and b is ciphertext. Note:
  - b must have its tail equal to iv after decryption.
- If a and b are equal, then depending on:
  - If a is `gimme_flag`, output the encrypted flag.
  - Otherwise, output a random encrypted string.
- Otherwise, output a plaintext string.

Additionally, we can observe that the unpad in the challenge has a problem — it can truncate a specified length.

```python
def unpad(msg):
    if not msg:
        return ''
    return msg[:-ord(msg[-1])]
```

Initially, the most straightforward approach is to input length 0 for both a and b, which can directly bypass the `a==b` check and obtain a random encrypted string. However, this doesn't seem useful. Let us analyze the encryption process:

```python
def tsb_encrypt(aes, msg):
    msg = pad(msg)
    iv = get_random_bytes(16)
    prev_pt = iv
    prev_ct = iv
    ct = ''
    for block in split_by(msg, 16) + [iv]:
        ct_block = xor(block, prev_pt)
        ct_block = aes.encrypt(ct_block)
        ct_block = xor(ct_block, prev_ct)
        ct += ct_block
        prev_pt = block
        prev_ct = ct_block
    return iv + ct
```

Let us define $P_0=iv,C_0=iv$, then:

 $C_i=C_{i-1}\oplus E(P_{i-1} \oplus P_i)$

Assuming the message length is 16, similar to the padded length of the `gimme_flag` we want to obtain:

 $C_1=IV\oplus E( IV \oplus P_1)$

 $C_2=C_1 \oplus E(P_1 \oplus IV)$

It is easy to observe that $C_2=IV$.

([Image borrowed](https://github.com/pberba/ctf-solutions/tree/master/20180929_teaser_dragon/aes_128_tsb) — the diagram below is clearer:

![](figure/aes-tsb-encryption.png)

Thinking in reverse, if we send `iv+c+iv` to the server, we can always bypass the `tsb_decrypt` MAC check:

```python
def tsb_decrypt(aes, msg):
    iv, msg = msg[:16], msg[16:]
    prev_pt = iv
    prev_ct = iv
    pt = ''
    for block in split_by(msg, 16):
        pt_block = xor(block, prev_ct)
        pt_block = aes.decrypt(pt_block)
        pt_block = xor(pt_block, prev_pt)
        pt += pt_block
        prev_pt = pt_block
        prev_ct = block
    pt, mac = pt[:-16], pt[-16:]
    if mac != iv:
        raise CryptoError()
    return unpad(pt)
```

In this case, the server's decrypted message is:

$unpad(IV \oplus D(C_1 \oplus IV))$

### Obtaining the Last Byte of the Plaintext

We can consider controlling the decrypted value of D to be a constant, such as all zeros, i.e., `C1=IV`. Then we can enumerate the last byte of IV from 0 to 255, obtaining the last byte of $IV \oplus D(C_1 \oplus IV)$ as also 0–255. Only when the value is 1–15 will the message length be non-zero after `unpad`. Therefore, during enumeration, we can track which numbers resulted in a non-zero length and mark them as 1, with the rest marked as 0.

```python
def getlast_byte(iv, block):
    iv_pre = iv[:15]
    iv_last = ord(iv[-1])
    tmp = []
    print('get last byte')
    for i in range(256):
        send_data('')
        iv = iv_pre + chr(i)
        tmpblock = block[:15] + chr(i ^ ord(block[-1]) ^ iv_last)
        payload = iv + tmpblock + iv
        send_data(payload)
        length, data = recv_data()
        if 'Looks' in data:
            tmp.append(1)
        else:
            tmp.append(0)
    last_bytes = []
    for i in range(256):
        if tmp == xor_byte_map[i][0]:
            last_bytes.append(xor_byte_map[i][1])
    print('possible last byte is ' + str(last_bytes))
    return last_bytes
```

Additionally, we can pre-compute a table at the beginning with all possible cases for the last byte, stored in xor_byte_map.

```python
"""
every item is a pair [a,b]
a is the xor list
b is the idx which is zero when xored
"""
xor_byte_map = []
for i in range(256):
    a = []
    b = 0
    for j in range(256):
        tmp = i ^ j
        if tmp > 0 and tmp <= 15:
            a.append(1)
        else:
            a.append(0)
        if tmp == 0:
            b = j
    xor_byte_map.append([a, b])
```

By comparing with this table, we can determine the possible values for the last byte.

### Decrypting an Arbitrary Encrypted Block

After obtaining the last byte of the plaintext, we can exploit the unpad vulnerability by enumerating lengths from 1 to 15 to obtain the corresponding plaintext content.

```python
def dec_block(iv, block):
    last_bytes = getlast_byte(iv, block)

    iv_pre = iv[:15]
    iv_last = ord(iv[-1])
    print('try to get plain')
    plain0 = ''
    for last_byte in last_bytes:
        plain0 = ''
        for i in range(15):
            print 'idx:', i
            tag = False
            for j in range(256):
                send_data(plain0 + chr(j))
                pad_size = 15 - i
                iv = iv_pre + chr(pad_size ^ last_byte)
                tmpblock = block[:15] + chr(
                    pad_size ^ last_byte ^ ord(block[-1]) ^ iv_last
                )
                payload = iv + tmpblock + iv
                send_data(payload)
                length, data = recv_data()
                if 'Looks' not in data:
                    # success
                    plain0 += chr(j)
                    tag = True
                    break
            if not tag:
                break
        # means the last byte is ok
        if plain0 != '':
            break
    plain0 += chr(iv_last ^ last_byte)
    return plain0
```

### Decrypting to a Specified Plaintext

This part is relatively simple — we want to use it to obtain the ciphertext of `gimme_flag`:

```python
    print('get the cipher of flag')
    gemmi_iv1 = xor(pad('gimme_flag'), plain0)
    gemmi_c1 = xor(gemmi_iv1, cipher0)
    payload = gemmi_iv1 + gemmi_c1 + gemmi_iv1
    send_data('gimme_flag')
    send_data(payload)
    flag_len, flag_cipher = recv_data()
```

Here plain0 and cipher0 are the AES encryption plaintext-ciphertext pair we obtained, excluding the two XOR operations before and after.

### Decrypting the Flag

This is essentially implemented using the arbitrary encrypted block decryption capability described above:

```python
    print('the flag cipher is ' + flag_cipher.encode('hex'))
    flag_cipher = split_by(flag_cipher, 16)

    print('decrypt the blocks one by one')
    plain = ''
    for i in range(len(flag_cipher) - 1):
        print('block: ' + str(i))
        if i == 0:
            plain += dec_block(flag_cipher[i], flag_cipher[i + 1])
        else:
            iv = plain[-16:]
            cipher = xor(xor(iv, flag_cipher[i + 1]), flag_cipher[i])
            plain += dec_block(iv, cipher)
            pass
        print('now plain: ' + plain)
    print plain
```

Think about why the ciphertext operations differ starting from the second block.

For the complete code, refer to the ctf-challenge repository.

## References

- [Block cipher modes of operation](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)
- https://en.wikipedia.org/wiki/Padding_oracle_attack
- http://netifera.com/research/poet/PaddingOraclesEverywhereEkoparty2010.pdf
- https://ctftime.org/writeup/7975
- https://ctftime.org/writeup/7974
