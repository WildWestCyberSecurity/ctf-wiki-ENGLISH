# PCBC

PCBC stands for Plaintext Cipher-Block Chaining. It is also known as Propagating Cipher-Block Chaining.

## Encryption

![](./figure/pcbc_encryption.png)

## Decryption

![](./figure/pcbc_decryption.png)

## Characteristics

- The decryption process is difficult to parallelize
- Swapping adjacent ciphertext blocks does not affect subsequent ciphertext blocks
