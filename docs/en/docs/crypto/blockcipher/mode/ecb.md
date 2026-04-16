# ECB

ECB stands for Electronic Codebook mode.

## Encryption

![](./figure/ecb_encryption.png)

## Decryption

![](./figure/ecb_decryption.png)

## Advantages and Disadvantages

### Advantages

1. Simple to implement.
2. Encryption of different plaintext blocks can be computed in parallel, making it very fast.

### Disadvantages

1. Identical plaintext blocks are encrypted into identical ciphertext blocks, which does not hide the statistical patterns of plaintext blocks. As shown in the figure below:

![image-20180716215135907](./figure/ecb_bad_linux.png)

To solve the problem of identical plaintext producing identical ciphertext, other encryption modes were proposed.

## Typical Applications

1. Used for encryption protection of random numbers.
2. Used for encryption of single-block plaintext.

## 2016 ABCTF aes-mess-75

The challenge description is as follows:

```
We encrypted a flag with AES-ECB encryption using a secret key, and got the hash: e220eb994c8fc16388dbd60a969d4953f042fc0bce25dbef573cf522636a1ba3fafa1a7c21ff824a5824c5dc4a376e75 However, we lost our plaintext flag and also lost our key and we can't seem to decrypt the hash back :(. Luckily we encrypted a bunch of other flags with the same key. Can you recover the lost flag using this?

[HINT] There has to be some way to work backwards, right?
```

We can see that this encryption is ECB encryption, and AES uses 16 bytes per group, where each byte can be represented by two hexadecimal characters. Therefore, we group every 32 characters and search in the corresponding txt file.

The corresponding flag:

```
e220eb994c8fc16388dbd60a969d4953 abctf{looks_like
f042fc0bce25dbef573cf522636a1ba3 _you_can_break_a
fafa1a7c21ff824a5824c5dc4a376e75 es}
```

The last one was obviously padded during encryption.

## Challenges

- 2018 PlaidCTF macsh

