# Block Cipher

## Overview

A block cipher encrypts one block of plaintext at a time. Common encryption algorithms include:

- IDEA encryption
- DES encryption
- AES encryption

Block ciphers are also symmetric encryption.

In fact, we can also understand block ciphers as a special type of substitution cipher, but each substitution operates on a large block. And precisely because of the large block, the plaintext space is enormous, and for different keys, we cannot create a lookup table to map to the corresponding ciphertext. Therefore, **complex** encryption and decryption algorithms are needed to encrypt and decrypt the plaintext and ciphertext.

At the same time, plaintext can often be very long or very short, so block encryption usually requires two auxiliary mechanisms:

- Padding, i.e., padding to the specified block length
- Block encryption mode, i.e., the method of encrypting plaintext blocks.

## Basic Strategies

In block cipher design, Shannon's two major strategies are fully utilized: confusion and diffusion.

### Confusion

Confusion makes the statistical relationship between the ciphertext and the key as complex as possible, so that even if an attacker obtains some statistical properties of the ciphertext, they cannot infer the key. Generally, complex nonlinear transformations can achieve good confusion effects. Common methods include:

- S-box
- Multiplication

### Diffusion

Diffusion ensures that each bit in the plaintext affects many bits in the ciphertext. Common methods include:

- Linear transformation
- Permutation
- Shift, circular shift

## Common Encryption/Decryption Structures

Currently, the main structure used in block ciphers is:

- Iterative structure, because iterative structures are easy to design and implement, and also convenient for security evaluation.

### Iterative Structure

#### Overview

The basic iterative structure generally includes three parts:

- Key scheduling
- Round encryption function
- Round decryption function

![image-20180714222206782](./figure/iterated_cipher.png)

#### Round Function

Currently, the main design methods for round functions are:

- Feistel Network, invented by Horst Feistel, one of the DES designers.
    - DES
- Substitution-Permutation Network (SPN)
    - AES
- Other schemes

#### Key Expansion

Currently, there are many methods for key expansion. No perfect key expansion method has been found. The basic principle is to make each bit of the key affect the round keys of as many rounds as possible.
