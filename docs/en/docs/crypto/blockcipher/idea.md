# IDEA

## Overview

**International Data Encryption Algorithm** (IDEA), originally called **Improved Proposed Encryption Standard** (IPES), is a symmetric key block cipher in cryptography, designed by James Massey and Xuejia Lai, first proposed in 1991. This algorithm was proposed to replace the old Data Encryption Standard DES. (From Wikipedia)

## Basic Process

### Key Generation

IDEA uses 6 keys in each round of encryption, and then the final output round uses 4 keys. So there are 52 in total.

1. The first 8 keys come from the algorithm's initial key, K1 is taken from the upper 16 bits of the key, and K8 is taken from the lower 16 bits of the key.
2. Circularly left-shift the key by 25 bits to obtain the next round of keys, then divide into 8 groups again.

### Encryption Process

The IDEA encryption data block size is 64 bits, and the key length used is 128 bits. The algorithm performs 8 identical transformations on the input data block, only using different keys each time, and finally performs one output transformation. The operations in each round:

![](figure/IDEA_Round.png)

The input and output are both grouped in 16-bit units. The main operations performed in each round are:

- Bitwise XOR, ⊕
- Modular addition, modulus $2^{16}$, ⊞
- Modular multiplication, modulus $2^{16}+1$, ⊙. Note that an input of 0x0000 will be changed to $2^{16}$, and an output result of $2^{16}$ will be changed to 0x0000.

Here we call the encryption method of the middle box composed of K5 and K6 as MA. This is also an important part of the IDEA algorithm. Additionally, we call MA_L the left result after encryption of this part, which will ultimately operate with the leftmost 16 bits; MA_R is the right half result after encryption of this part, which will ultimately operate with the third 16 bits.

The operations in the final output round are as follows:

![](figure/IDEA_Output_Trans.png)

### Decryption Process

The decryption process is similar to the encryption process, mainly differing in the key selection:

- The first 4 subkeys of the decryption key in round i (1-9) are derived from the first 4 subkeys of round 10-i in the encryption process
  - The 1st and 4th decryption subkeys are the multiplicative inverses of the corresponding subkeys modulo $2^{16}+1$.
  - The 2nd and 3rd subkeys are selected as follows:
    - When the round number is 2, ..., 8, take the additive inverses modulo $2^{16}$ of the corresponding 3rd and 2nd subkeys.
    - When the round number is 1 or 9, take the additive inverses modulo $2^{16}$ of the corresponding 2nd and 3rd subkeys.
- The 5th and 6th keys remain unchanged.

### Overall Process

![](figure/IDEA_All.png)

Let us prove the correctness of the algorithm. Here we focus on the first round of the decryption algorithm. First, let's see how $Y_i$ is obtained:

$Y_1 = W_{81} \odot Z_{49}$

$Y_2=W_{83}\boxplus Z_{50}$

$Y_3=W_{82}\boxplus Z_{51}$

$Y_4=W_{83}\odot Z_{52}$

During decryption, the transformations directly performed in the first round are:

$J_{11}=Y_1 \odot U_1=Y_1 \odot Z_{49}^{-1}=W_{81}$

$J_{12}=Y_2 \boxplus U2=Y_2\boxplus Z_{50}^{-1}=W_{83}$

$J_{13}=Y_3 \boxplus U3=Y_3\boxplus Z_{51}^{-1}=W_{82}$

$J_{14}=Y_4 \odot U_4=Y_4 \odot Z_{52}^{-1}=W_{84}$

We can see that the results obtained only have the two middle 16-bit encrypted results swapped. Let's further look at how $W_{8i}$ is obtained:

$W_{81}=I_{81} \oplus MA_R(I_{81}\oplus I_{83},I_{82}\oplus I_{84})$

$W_{82}=I_{83} \oplus MA_R(I_{81}\oplus I_{83},I_{82}\oplus I_{84})$

$W_{83}=I_{82} \oplus MA_L(I_{81}\oplus I_{83},I_{82}\oplus I_{84})$

$W_{84}=I_{84} \oplus MA_L(I_{81}\oplus I_{83},I_{82}\oplus I_{84})$

Then for V11:

$V_{11}=J_{11} \oplus MA_R(J_{11}\oplus J_{13},J_{12}\oplus J_{14})$

By simple substitution of known values, it is obvious that:

$V_{11}=W_{81} \oplus MA_R(I_{81}\oplus I_{83},I_{82} \oplus I_{84})=I_{81}$

The same applies to other elements. We will find that the result after the first round of decryption is exactly $I_{81},I_{83},I_{82},I_{84}$.

Similarly, this relationship holds all the way until:

$V_{81}=I_{11},V_{82}=I_{13},V_{83}=I_{12},V_{84}=I_{14}$

Then after one final simple output transformation, we obtain exactly the originally encrypted values.

![](figure/IDEA_Round.png)



## Challenges

- 2017 HITCON seccomp
