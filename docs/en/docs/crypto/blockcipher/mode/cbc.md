# CBC

CBC stands for Cipher-Block Chaining mode, where:

- The IV does not need to be kept secret
- The IV must be unpredictable, and its integrity must be ensured.

## Encryption

![](./figure/cbc_encryption.png)

## Decryption

![](./figure/cbc_decryption.png)

## Advantages and Disadvantages

### Advantages

1. The ciphertext block is related not only to the current plaintext block but also to the previous ciphertext block or IV, hiding the statistical properties of the plaintext.
2. Has limited two-step error propagation property, meaning a single bit change in a ciphertext block only affects the current ciphertext block and the next ciphertext block.
3. Has self-synchronizing property, meaning if the ciphertext is correct from block k onward, then block k+1 can be decrypted normally.

### Disadvantages

1. Encryption cannot be parallelized, but decryption can be parallelized.

## Applications

CBC is very widely used:

- Common data encryption and TLS encryption.
- Integrity authentication and identity authentication.

## Attacks

### Byte Flipping Attack

#### Principle
The principle of byte flipping is quite simple. By observing the **decryption process**, we can find the following properties:

- The IV vector affects the first plaintext block
- The n-th ciphertext block can affect the (n + 1)-th plaintext block

Assume the n-th ciphertext block is $C_n$, and the decrypted n-th plaintext block is $P_n$.

Then $P_{n+1}=C_n~\text{xor}~f(C_{n+1})$.

Where the $f$ function is the $\text{Block Cipher Decryption}$ shown in the diagram.

For a known plaintext and ciphertext, we can modify the n-th ciphertext block $C_n$ to $C_n~\text{xor}~P_{n+1}~\text{xor}~A$. Then when decrypting this ciphertext, the decrypted n-th plaintext block will become $A$.

#### Example

```python
from flag import FLAG
from Crypto.Cipher import AES
from Crypto import Random
import base64

BLOCK_SIZE=16
IV = Random.new().read(BLOCK_SIZE)
passphrase = Random.new().read(BLOCK_SIZE)

pad = lambda s: s + (BLOCK_SIZE - len(s) % BLOCK_SIZE) * chr(BLOCK_SIZE - len(s) % BLOCK_SIZE)
unpad = lambda s: s[:-ord(s[len(s) - 1:])]

prefix = "flag="+FLAG+"&userdata="
suffix = "&user=guest"
def menu():
    print "1. encrypt"
    print "2. decrypt"
    return raw_input("> ")

def encrypt():
    data = raw_input("your data: ")
    plain = prefix+data+suffix
    aes = AES.new(passphrase, AES.MODE_CBC, IV)
    print base64.b64encode(aes.encrypt(pad(plain)))


def decrypt():
    data = raw_input("input data: ")
    aes = AES.new(passphrase, AES.MODE_CBC, IV)
    plain = unpad(aes.decrypt(base64.b64decode(data)))
    print 'DEBUG ====> ' + plain
    if plain[-5:]=="admin":
        print plain
    else:
        print "you are not admin"

def main():
    for _ in range(10):
        cmd = menu()
        if cmd=="1":
            encrypt()
        elif cmd=="2":
            decrypt()
        else:
            exit()

if __name__=="__main__":
    main()
```

As we can see, the challenge expects us to provide an encrypted string. If the decrypted string ends with "admin", the program will output the plaintext. So the workflow is: first provide an arbitrary plaintext, then modify the ciphertext so that the decrypted string ends with "admin". We can enumerate the flag length to determine at which position we need to make modifications.

Below is exp.py:

```python
from pwn import *
import base64

pad = 16
data = 'a' * pad
for x in range(10, 100):
    r = remote('xxx.xxx.xxx.xxx', 10004)
    #r = process('./chall.sh')
    
    r.sendlineafter('> ', '1')
    r.sendlineafter('your data: ', data)
    cipher = list(base64.b64decode(r.recv()))
    #print 'cipher ===>', ''.join(cipher)
    
    BLOCK_SIZE = 16
    prefix = "flag=" + 'a' * x + "&userdata="
    suffix = "&user=guest"
    plain = prefix + data + suffix
    
    idx = (22 + x + pad) % BLOCK_SIZE + ((22 + x + pad) / BLOCK_SIZE - 1) * BLOCK_SIZE
    cipher[idx + 0] = chr(ord(cipher[idx + 0]) ^ ord('g') ^ ord('a'))
    cipher[idx + 1] = chr(ord(cipher[idx + 1]) ^ ord('u') ^ ord('d'))
    cipher[idx + 2] = chr(ord(cipher[idx + 2]) ^ ord('e') ^ ord('m'))
    cipher[idx + 3] = chr(ord(cipher[idx + 3]) ^ ord('s') ^ ord('i'))
    cipher[idx + 4] = chr(ord(cipher[idx + 4]) ^ ord('t') ^ ord('n'))

    r.sendlineafter('> ', '2')
    r.sendlineafter('input data: ', base64.b64encode(''.join(cipher)))

    msg = r.recvline()
    if 'you are not admin' not in msg:
        print msg
        break
    r.close()  

```
### Padding Oracle Attack
See the introduction below for details.
