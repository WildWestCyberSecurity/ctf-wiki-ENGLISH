# Meet-in-the-Middle Attack - MITM

## Overview

The meet-in-the-middle attack is a time-space tradeoff attack method proposed by Diffie and Hellman in 1977. From a personal perspective, it is more of a concept that is not only applicable to cryptographic attacks but also to other areas, and can reduce algorithmic complexity.

The basic principle is as follows:

Assume E and D are the encryption and decryption functions respectively, and k1 and k2 are the keys used for the two encryptions. Then we have:

$C=E_{k_2}(E_{k_1}(P))$

$P=D_{k_1}(D_{k_2}(C))$

From this we can derive:

$E_{k_1}(P)=D_{k_2}(C)$

When the user knows a pair of plaintext and ciphertext:

1. The attacker can enumerate all possible k1 values, store all encrypted results of P, and sort them by ciphertext size.
2. The attacker then enumerates all possible k2 values, decrypts the ciphertext C to get C1, and searches for C1 in the encrypted results from step 1. If found, we can reasonably conclude that we have found the correct k1 and k2.
3. If the result obtained in step 2 is not reliable enough, we can use additional plaintext-ciphertext pairs for verification.

Assuming both k1 and k2 have a key length of n, the original brute-force enumeration requires $O(n^2)$, but now we only need $O(n log_2n)$.

This is similar to the meet-in-the-middle attack on 2DES.

## Challenges

- 2018 National Competition Crackmec, see the Wiki AES section
- 2018 Plaid CTF Transducipher, see the principles in the Bit Attack section
- 2018 National Competition Crackme java, see the Wiki Discrete Logarithm over Integer Fields section
- 2018 WCTF RSA, see the wiki RSA Complex section

## References

- https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%80%94%E7%9B%B8%E9%81%87%E6%94%BB%E6%93%8A
