# Introduction

## Attack Modes

When attacking a cryptographic system, we will inevitably obtain some information about the system. Depending on the amount of information obtained, the methods we can adopt may differ. In modern cryptanalysis, we generally assume that the attacker knows the cryptographic algorithm, which is a reasonable assumption because many historically secret algorithms have eventually become publicly known, such as RC4. They were discovered through various means, such as espionage, reverse engineering, etc.

Here, based on the amount of information the attacker obtains from the cryptographic system, we classify attack modes into the following categories:

- **Ciphertext-Only Attack**: The attacker can only obtain some encrypted ciphertext.
- **Known-Plaintext Attack**: The attacker has some plaintext corresponding to the ciphertext.
- **Chosen-Plaintext Attack**: The attacker can choose some plaintext before starting the attack and obtain the encrypted ciphertext. If the attacker can choose new plaintext and obtain the corresponding ciphertext based on information already obtained during the attack, it is called an Adaptive Chosen-Plaintext Attack.
- **Chosen-Ciphertext Attack**: The attacker can choose some ciphertext before starting the attack and obtain the decrypted plaintext. If the attacker can choose new ciphertext and obtain the corresponding plaintext based on information already obtained during the attack, it is called an Adaptive Chosen-Ciphertext Attack.
- **Related-Key Attack**: The attacker can obtain the ciphertext or plaintext encrypted or decrypted with two or more related keys. However, the attacker does not know these keys.

## Common Attack Methods

Depending on different attack modes, there may be different attack methods. Currently, the common attack methods are:

- Brute-Force Attack
- Meet-in-the-Middle Attack
- Linear Cryptanalysis
- Differential Cryptanalysis
- Impossible Differential Cryptanalysis
- Integral Cryptanalysis
- Algebraic Cryptanalysis
- Related-Key Attack
- Side-Channel Attack

## References

- https://zh.wikipedia.org/wiki/%E5%AF%86%E7%A0%81%E5%88%86%E6%9E%90
