# Basic Introduction

## Lattice Definition

A lattice is the set of all integer linear combinations of n ($m\geq n$) linearly independent vectors $b_i(1\leq i \leq n)$ in the m-dimensional Euclidean space $R^m$, i.e.,
$L(B)=\{\sum\limits_{i=1}^{n}x_ib_i:x_i \in Z,1\leq i \leq n\}$

Here B is the set of n vectors. We say:

- These n vectors are a basis of the lattice L.
- The rank of lattice L is n.
- The dimension of lattice L is m.

If m=n, then we say the lattice is full-rank.

Of course, it can also be over other groups, not just $R^m$.

## Several Basic Definitions in Lattices

### successive minima

A lattice in the m-dimensional Euclidean space $R^m$ with rank n has successive minima $\lambda_1,...,\lambda_n \in R$, satisfying that for any $1\leq i\leq n$, $\lambda_i$ is the minimum value such that there exist i linearly independent vectors $v_i$ in the lattice with $||v_j||\leq \lambda_i,1\leq j\leq i$.

Naturally, $\lambda_i \leq \lambda_j ,\forall i <j$.

## Computationally Hard Problems in Lattices

**Shortest Vector Problem (SVP)**: Given a lattice L and its basis vectors B, find a non-zero vector v in lattice L such that for any other non-zero vector u in the lattice, $||v|| \leq ||u||$.

**$\gamma$-Approximate Shortest Vector Problem (SVP-$\gamma$)**: Given a lattice L, find a non-zero vector v in lattice L such that for any other non-zero vector u in the lattice, $||v|| \leq \gamma||u||$.

**Successive Minima Problem (SMP)**: Given a lattice L of rank n, find n linearly independent vectors $s_i$ in lattice L satisfying $\lambda_i(L)=||s_i||, 1\leq i \leq n$.

**Shortest Independent Vector Problem (SIVP)**: Given a lattice L of rank n, find n linearly independent vectors $s_i$ in lattice L satisfying $||s_i|| \leq \lambda_n(L), 1\leq i \leq n$.

**Unique Shortest Vector Problem (uSVP-$\gamma$)**: Given a lattice L satisfying $ \lambda_2(L) > \gamma \lambda_1(L)$, find the shortest vector of the lattice.

**Closest Vector Problem (CVP)**: Given a lattice L and a target vector $t\in R^m$, find a non-zero vector v in the lattice such that for any non-zero vector u in the lattice, $||v-t|| \leq ||u-t||$.


