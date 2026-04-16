# Fundamental Mathematical Knowledge
<!-- https://ctf-wiki.org/crypto/basic/introduction/ -->
This section will introduce "fundamental mathematical knowledge" — note the quotation marks, as the content is not necessarily all that basic.

## Algebraic Systems and Modern Algebra

Within a set, if one or more **algebraic operations** (Algebraic Operations) are defined, we generally refer to it as an **algebraic system** (Algebraic System), also called an **algebraic structure** (Algebraic Structure).

As a continuously advancing and refining branch of mathematics, the scope of algebra has gradually expanded. The sets it studies have grown from the classical number sets — integers, rationals, reals, and complex numbers — to objects such as vectors, matrices, and linear operators, with a focus on the algebraic operations defined on them. These topics collectively form what is now known as **Modern Algebra**, or **Abstract Algebra**.

The algebraic operations mentioned above are rules defined among elements of a set and are closely related to whether a set can **form** an algebraic system. They are extensions of familiar arithmetic operations such as addition, subtraction, multiplication, and division. By defining appropriate algebraic operations, a set can form algebraic systems such as groups, rings, fields, lattices, etc., which are what we will introduce below.

## Groups

Given a non-empty set $G\neq\varnothing$ and a binary algebraic operation "$\circ$" defined on it, if they satisfy the following properties:

1. Closure: $\forall v, u \in G, \quad v \circ u \in G;$
2. Associativity: $\forall v, u, w \in G, \quad (v \circ u) \circ w = v \circ (u \circ w);$
3. Identity: $\exists e \in G, \forall v \in G, \quad e \circ v = v;$
4. Inverse: $\forall v \in G, \exists v^{-1} \in G, \quad v^{-1} \circ v = e;$

then the set $G$ is said to form a **group** (Group) under this algebraic operation, denoted $(G,\circ)$.

A very common example is the integer addition group $(\mathbb{Z},+)$. It is easy to verify that it is closed under addition, satisfies associativity, has the integer $0$ as the identity element, and for every integer $m$ has its additive inverse $-m$ as the inverse element. Similarly, one can verify that the set of positive rationals $(\mathbb{Q}_+,\times)$ forms a group under multiplication (the identity element is $1$, and for every element $a$, the inverse is $\frac{1}a$); the set of all invertible $m \times m$ matrices over the real field $\mathbb{R}$ forms a group under matrix multiplication (the identity element is the $m \times m$ identity matrix $E_m$, and for every element $A$, the inverse is its matrix inverse $A^{-1}$). In modern algebra, this is called the $m$-th order general linear group $GL_m(\mathbb{R})$.

Additionally, here is an even simpler example: the multiplication group defined on the set $\{-1,1\}$, i.e., $(\{-1,1\},\times)$, which is also easy to verify as forming a group.

In modern algebra, the branch that studies groups is called **Group Theory**.

### Semigroups and Monoids

In modern algebra, some algebraic systems possess partial properties of a ring. Although they are not within our main scope of discussion, they also have broad applications and significant research value:

For a non-empty set that is closed under a binary algebraic operation,

* If it only satisfies associativity, then the set is said to form a **semigroup** (Semigroup) under that operation;
* If the set, in addition to being closed, satisfies associativity and has an identity element, then it is said to form a **monoid** (Monoid) under that operation.

Therefore, we can consider:

* A monoid is a semigroup with an identity element;
* A group is a monoid in which every element has an inverse.

For example, the positive integers form a semigroup under integer addition, while the non-negative integers form a monoid under integer addition, since zero can be regarded as the identity element of integer addition.

### Abelian Groups

Given a group $(G,\circ)$, if it satisfies commutativity (Commutativity), i.e. $\forall v, u \in G,$ $ v \circ u = u \circ v, $ then this group is called an **abelian group** or Abel group (Abelian Group).

It is easy to see that among the examples mentioned above, the integer addition group $(\mathbb{Z},+)$ is an abelian group, but the $m$-th order general linear group $GL_m(\mathbb{R})$ is not.

### Rings and Fields

Given a non-empty set $R\neq\varnothing$ and two binary algebraic operations "$+$" and "$\circ$" defined on it, if they satisfy the following properties:

1. $(R,+)$ forms an abelian group;
2. $R$ satisfies associativity under operation "$\circ$": $\forall v, u, w \in R,$ $(v \circ w) \circ u = v \circ (w \circ u);$
3. Distributivity: $\forall v, u, w \in R,$ $w \circ (v + u) = w \circ v + w \circ u$ and $(v + u) \circ w = v \circ w + u \circ w$ both hold;

then the set $R$ is said to form a **ring** (Ring) under these two algebraic operations, denoted $(R,+,\circ)$, where operations "$+$" and "$\circ$" are commonly called addition and multiplication, respectively.

* If the multiplication on ring $R$ has an identity element, i.e. $\exists e \in G, \forall v \in G,$ $e \circ v = v,$ then ring $R$ is called a **ring with identity** (Ring with identity);
* If the multiplication on ring $R$ satisfies commutativity, then it is called a **commutative ring** (Commutative Ring);
* If for every element $a \neq 0$ in ring $R$ (other than the additive identity) there exists a multiplicative inverse $a^{-1}$, then $R$ is called a **division ring** (Division Ring);
* If ring $R$ is both a commutative ring and a division ring, then ring $R$ is a **field** (Field).

> In some textbooks, a ring is assumed to contain a multiplicative identity by default, and a ring without a multiplicative identity is called a **pseudo ring** (Pseudo Ring).

In modern algebra, the branches that study rings and fields are called **Ring Theory** and **Field Theory**, respectively.

> In some Traditional Chinese contexts, "field" and "field theory" are often referred to as "体" (body) and "体论" (body theory) (written as「體」and「體論」in Traditional Chinese).

### Order

Exponentiation: Analogous to powers of numbers, we define the exponentiation of elements in a group. For $v \in G, m$ a positive integer,

* $v^0 = e;$
* $v^m = v \circ v \circ \cdots \circ v,$ where there are $m$ copies of $v$ participating in the operation;
* $v^{-m} = \left(v^{-1}\right)^m;$

Order of an element: For any given element $v \in G,$ if a positive integer $m$ satisfies $v^m = e,$ then the order of element $v$ is said to be $m$. If no such positive integer exists, the element is said to have infinite order.

For example, in the group $\left(\{1,-1,+\mathrm{j},-\mathrm{j}\},\times\right)$, the orders of each element are as follows:

| Element | Order |
|:-:|:-:|
| 1 | 1 |
| -1 | 2 |
| $+\mathrm{j}$ | 4 |
| $-\mathrm{j}$ | 4 |

### Homomorphism

A **homomorphism** between algebraic systems refers to a mapping between different algebraic systems that preserves the algebraic operations.

Specifically, for groups $(G,\circ)$ and $(H,\ast)$, if a mapping $\psi: G \to H$ satisfies $\forall v, u \in G,$

$$ \psi(v \circ u) = \psi(v) \ast \psi(u), $$

then the mapping $\psi$ is called a **group homomorphism** from $G$ to $H$.

## References

* Yang Zixu, *Modern Algebra* (4th Edition), Higher Education Press
* [Introduction to Group Theory - OI-Wiki](https://oi-wiki.org/math/group-theory/)
