# Lattice Overview

Lattice has at least two meanings in mathematics:

- A partially ordered set L defined on a non-empty finite set, such that for any elements a, b in L, there exists a greatest lower bound and a least upper bound of a, b in L. See https://en.wikipedia.org/wiki/Lattice_(order) for details.
- A definition in group theory: a subset of $R^n$ satisfying certain properties. Of course, it can also be over other groups.

Current research on lattices mainly covers the following directions:

1. Hardness of computational problems in lattices, i.e., the computational complexity of these problems, mainly including:
    1. SVP problem
    2. CVP problem
2. How to solve hard problems in lattices. Currently there are both approximation algorithms and some exact algorithms.
3. Lattice-based cryptanalysis, i.e., how to use lattice theory to analyze existing cryptographic algorithms. Current research includes:
    1. Knapsack cryptosystems
    2. DSA nonce biases
    3. Factoring RSA keys with bits known
    4. Small RSA private exponents
    5. Stereotyped messages with small RSA exponents
4. How to design new cryptosystems based on lattice hard problems. This is also one of the important research directions in the post-quantum cryptography era. Current research includes:
    1. Fully homomorphic encryption
    2. The Goldreich–Goldwasser–Halevi (GGH) cryptosystem
    3. The NTRU cryptosystem
    4. The Ajtai–Dwork cryptosystem and the LWE cryptosystem
