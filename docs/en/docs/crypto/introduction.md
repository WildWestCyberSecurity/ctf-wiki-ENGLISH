# Introduction to Cryptography

Cryptography can generally be divided into classical cryptography and modern cryptography.

Classical cryptography, existing as a practical art, had its encoding and decryption typically relying on the creativity and skill of the designer and adversary, without clearly defining cryptographic primitives. Classical cryptography mainly includes the following aspects:

- Monoalphabetic Cipher
- Polyalphabetic Cipher
- Various unusual encryption methods

Modern cryptography originated from the large amount of related theory that emerged in the mid-to-late 20th century. The publication of Shannon's (C. E. Shannon) classic paper "Communication Theory of Secrecy Systems" in 1949 marked the beginning of modern cryptography. Modern cryptography mainly includes the following aspects:

- Symmetric Cryptography, represented by DES, AES, RC4.
- Asymmetric Cryptography, represented by RSA, ElGamal, Elliptic Curve Cryptography.
- Hash Function, represented by MD5, SHA-1, SHA-512, etc.
- Digital Signature, represented by RSA signature, ElGamal signature, DSA signature.

Among these, symmetric cryptographic systems are mainly divided into two types:

- Block Cipher
- Stream Cipher

Generally speaking, the fundamental goal of cryptography designers is to ensure the following properties of information and information systems:

- Confidentiality
- Integrity
- Availability
- Authentication
- Non-repudiation

The first three are known as the CIA triad of information security.

For cryptanalysts, the general approach is to try to identify the cryptographic algorithm, then perform brute force cracking or exploit vulnerabilities in the cryptographic system. Of course, it is also possible to bypass corresponding checks by constructing forged hash values or digital signatures.

Generally, we assume that the attacker knows the cryptographic system to be broken, and attack types are typically divided into the following four categories:

| Attack Type              | Description                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| Ciphertext-only attack   | Only possessing the ciphertext                                                           |
| Known-plaintext attack   | Possessing the ciphertext and the corresponding plaintext                                |
| Chosen-plaintext attack  | Having encryption access, able to encrypt plaintext and obtain the corresponding ciphertext |
| Chosen-ciphertext attack | Having decryption access, able to decrypt ciphertext and obtain the corresponding plaintext |

!!! note 
    Note: Previously, common scenarios for these attacks were written here. As learning progressed, it was gradually realized that these attack types focus on describing the attacker's capabilities and may be applicable to a wide variety of scenarios. Hence the correction.

Here are some recommended resources:

- [Khan Academy Open Course](http://open.163.com/special/Khan/moderncryptography.html)
- [Understanding Cryptography — Principles and Applications of Common Encryption Techniques](https://github.com/yuankeyang/python/blob/master/%E3%80%8A%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BA%E5%AF%86%E7%A0%81%E5%AD%A6%E2%80%94%E2%80%94%E5%B8%B8%E7%94%A8%E5%8A%A0%E5%AF%86%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86%E4%B8%8E%E5%BA%94%E7%94%A8%E3%80%8B.pdf)
- https://cryptopals.com/, a collection of cryptography exercises.

!!! note
    It is recommended to watch the open course first and briefly review the e-book before considering whether to purchase the physical book, as books bought often end up sitting idle.

## References

- [Wikipedia - Cryptography](https://zh.wikipedia.org/wiki/%E5%AF%86%E7%A0%81%E5%AD%A6)

!!! info
    Most definitions and examples in this section are referenced from Wikipedia.
