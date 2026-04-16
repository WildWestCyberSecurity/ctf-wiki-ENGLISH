# Cryptographically Secure Pseudorandom Number Generator

## Introduction

A cryptographically secure pseudo-random number generator (CSPRNG), also known as a cryptographic pseudo-random number generator (CPRNG), is a special type of pseudorandom number generator. It needs to satisfy certain necessary properties to be suitable for cryptographic applications.

Many aspects of cryptography require random numbers:

-   Key generation
-   Generating initialization vectors (IVs) for block cipher modes such as CBC, CFB, and OFB
-   Nonces, used to prevent replay attacks and for block cipher CTR mode, etc.
-   [one-time pads](https://en.wikipedia.org/wiki/One-time_pad)
-   Salts in certain signature schemes, such as [ECDSA](https://en.wikipedia.org/wiki/ECDSA), [RSASSA-PSS](https://en.wikipedia.org/w/index.php?title=RSASSA-PSS&action=edit&redlink=1)

## Requirements

Undoubtedly, the requirements for a cryptographically secure pseudorandom number generator are higher than those for a general pseudorandom number generator. Generally speaking, the requirements for a CSPRNG can be divided into two categories:

-   Pass statistical randomness tests. A CSPRNG must pass the [next-bit test](https://en.wikipedia.org/wiki/Next-bit_test), meaning that given the first k bits of a sequence, an attacker cannot predict the next bit in polynomial time with a probability greater than 50%. It is worth mentioning that Andrew Yao proved in 1982 that if a generator can pass the [next-bit test](https://en.wikipedia.org/wiki/Next-bit_test), then it can also pass all other polynomial-time statistical tests.
-   It must be able to withstand sufficiently strong attacks. For example, when part of the generator's initial state or runtime state is known to the attacker, the attacker should still be unable to obtain the random numbers generated before the state was leaked.

## Classification

Currently, CSPRNG designs can be divided into the following three categories:

-   Based on cryptographic algorithms, such as ciphertexts or hash values.
-   Based on mathematical hard problems
-   Some special-purpose designs

## References

-   https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator
