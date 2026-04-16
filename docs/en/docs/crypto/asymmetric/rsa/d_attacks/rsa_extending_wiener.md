# Extending Wiener's Attack

* The `Extending Wiener's Attack` originates from [`"Extending Wiener's Attack in the Presence of Many Decrypting Exponents"`](https://www.sci-hub.ren/https://link.springer.com/chapter/10.1007/3-540-46701-7_14). Related problems have already appeared in CTF competitions, such as "Simple" from the 2020 Yangcheng Cup, but they are all *template problems*. Here we will analyze in detail the method proposed in the original paper and its analysis approach, explain the principle of the Extending Wiener's Attack, and present some **open questions** at the end for discussion.

## Principle Analysis

### Wiener's Approach

* Wiener proposed a method for factoring $N$ when the private key is too small. He proved that when

    $$
    d < \frac{1}{3}N^{\frac{1}{4}}
    $$
    
    is satisfied (it should also satisfy $q < p < 2q$, but since the discussion here and later focuses mainly on the private key, we omit such conditions), $N$ can always be factored.

* The following is a partial description of `Wiener's Approach` from the original paper, with some content omitted. This is essentially the proof of Wiener's attack, so for a more detailed understanding please refer to the principle of Wiener's attack. Here we mainly need `Equation 1` for later use. The method is as follows:

    Given

    $$
    e*d -k*\lambda(N) = 1
    $$

    where $\lambda(N) = lcm(p-1, q-1) = \varphi(N) / g$, let $s = 1-p-q$, then we have

    $$
    edg - kN = g + ks\tag{1}
    $$

    Dividing both sides by $dgN$ gives

    $$
    \frac{e}{N} - \frac{k}{dg} = \frac{g+ks}{dgN} = (\frac{k}{dg})(\frac{s}{N}) + \frac{1}{dN}
    $$

    We know that $e \approx N, s \approx N^{1/2}$, so $k/(dg)\approx 1$. Thus we can see that the right side of the equation is approximately $N^{-1/2}$. We know that when

    $$|x - a/b| < 1/(2b^2)$$

    then $a/b$ is a continued fraction approximation of $x$ (`Continued Fractions Theorem`)

    So when

    $$d < \frac{\sqrt{2}}{2g}N^{\frac{1}{4}}$$

    $k/dg$ is a continued fraction approximation of $e/N$, meaning it can be covered by continued fraction expansion.

* Note that the ranges mentioned earlier and later are not contradictory.

    The approximations of some parameter values here are not strict, so there are discrepancies with the strict range of Wiener's attack. For specific details, please refer to the proof of Wiener's attack.

### Guo's Approach

* Guo studied the case with more than one $e$, but only investigated the cases of two and three $e$ values. As in the previous section, we still use the approach of translating and explaining the original text. For the case of two $e$ values, we can consider

    $$
    e_1d_1g - k_1(p-1)(q-1) = g\\
    e_2d_2g - k_2(p-1)(q-1) = g
    $$

    Simple simplification yields the following equation

    $$
    k_2d_1e_1 - k_1d_2e_2 = k_2 - k_1\tag{2}
    $$

    Dividing both sides by $k_2d_1e_2$

    $$
    \frac{e_1}{e_2} - \frac{k_1d_2}{k_2d_1} = \frac{k_2 - k_1}{k_2d_1e_2}
    $$

    Let $d_i < N^\alpha$, then the right side of the equation is approximately $N^{-(1+\alpha)}$

    So when

    $$2(k_2d_1)^2 < N^{1+\alpha}$$

    $k_1d_2/(k_2d_1)$ is a continued fraction approximation of $e_1/e_2$. When $k_2$ and $d_1$ are at most $N^\alpha$ and $g$ is small, we get

    $$\alpha < 1/3 - \epsilon\ \ \ (\epsilon > 0)$$

* However, even if we obtain $(k_1d_2)/(k_2d_1)$, we still cannot factor $N$. The original paper further discusses Guo's proposal to attempt factoring $k_1d_2$, which we will not elaborate on here.

## Extending Wiener's Attack

* As of now (2021/10), the above content has been explained and analyzed in many blog posts online, but the specific principle of the Extending Wiener's Attack, lattice construction, and generalization to higher dimensions have not been provided. Here I will translate and explain the original paper content in detail.

* To extend the analysis to $n$ encryption exponents $e_i$ (with small decryption exponents $d_i$), we use both Wiener's and Guo's methods simultaneously. We denote the relation

    $$
    d_ige_i - k_iN = g + k_is
    $$

    as the Wiener equation $W_i$, and similarly we can obtain the relation

    $$
    k_id_je_j - k_jd_ie_i = k_i - k_j
    $$

    denoted as the Guo equation $G_{i,j}$.

    We assume that both $d_i$ and $k_i$ are less than $N^{\alpha_n}$, with $g$ being small and $s \approx N^{1/2}$. Note that the right sides of $W_i$ and $G_i$ are very small, in fact at most $N^{1/2 + \alpha}$ and $N^\alpha$ respectively.

    Finally, we consider composite relations such as $W_uG_{v,w}$, which are clearly of size $N^{1/2 + 2\alpha}$.

* The original paper defines two types of relations and indicates their size ranges. These ranges are important and easy to analyze. What we do next is use different composite relations of these two equations to construct a lattice, then find its basis vectors to obtain $d_1g/k_1$, from which we can compute $\varphi(N)$ and further factor $N$.

* At this point the principle analysis is essentially complete. The lattice construction is not particularly complex, but the core lies in the selection of composite relations and the analysis of the final bound on $\alpha$.

## Case of Two Small Decryption Exponents

* We select the relations $W_1, G_{1,2},W_1W_2$, giving us

    $$
    \begin{aligned}
        d_1ge_1 - k_1N &= g+k_1s\\
        k_1d_2e_2 - k_2d_1e_1 &= k_1-k_2\\
        d_1d_2g^2e_1e_2 - d_1gk_2e_1N - d_2gk_1e_2N + k_1k_2N^2 &= (g+k_1s)(g+k_2s)
    \end{aligned}
    $$

    We multiply the first relation by $k_2$, so that the left side is entirely composed of $d_1d_2g^2, d_1gk_2, d_2gk_1$ and $k_1k_2$. This allows us to use known quantities to construct a lattice and convert the above equations into matrix operations

    $$
    \begin{pmatrix}
        k_1k_2&d_1gk_2&d_2gk_1&d_1d_2g^2
    \end{pmatrix} \begin{pmatrix}
        1&-N&0&N^2\\
        &e_1&-e_1&-e_1N\\
        &&e_2&-e_2N\\
        &&&e_1e_2
    \end{pmatrix} = \begin{pmatrix}
        k_1k_2&k_2(g+k_1s)&g(k_1 - k_2)&(g+k_1s)(g+k_2s)
    \end{pmatrix}
    $$

    The sizes of the right-side vector components are $N^{2\alpha_2}, N^{1/2+2\alpha_2}, N^{\alpha_2}, N^{1+2\alpha_2}$. To make the sizes equal, we can construct a D matrix.

    $$
    D = \begin{pmatrix}
        N&&&\\
        &N^{1/2}&&\\
        &&N^{1+\alpha_2}&\\
        &&&1
    \end{pmatrix}
    $$

    The final matrix we construct is

    $$
    L_2 = \begin{pmatrix}
        1&-N&0&N^2\\
        &e_1&-e_1&-e_1N\\
        &&e_2&-e_2N\\
        &&&e_1e_2
    \end{pmatrix} * D
    $$

    Then the vector $b = \begin{pmatrix} k_1k_2&d_1gk_2&d_2gk_1&d_1d_2g^2 \end{pmatrix}$ satisfies

    $$
    \Vert bL_2 	\Vert < 2N^{1+2\alpha_2}
    $$

    This is why we need to construct the $D$ matrix. Given the $D$ matrix, we can obtain an upper bound, and the problem can be transformed into an SVP-like problem.

    The vector b can be obtained using a lattice basis reduction algorithm such as `LLL` to find the basis vector $b$, and then we solve $b_2/b_1$ to get $d_1g/k_1$.

    Then we can obtain

    $$
    \varphi(N) = \frac{edg}{k} - \frac{g}{k} = \lfloor edg/k\rceil
    $$

    We assume the shortest vector length in these lattices is $\Delta^{1/4-\epsilon}$, where $\Delta = det(L_2) = N^{13/2 + \alpha_2}$. If these lattices are random, we can almost certainly say that no lattice point is shorter than Minkowski's bound $2\Delta^{1/4}$, so $bL_2$ is the shortest vector when

    $$
    N^{1+2\alpha_2} < (1/c_2)\left(N^{13/2+\alpha_2}\right)^{1/4}
    $$

    For some small $c_2$, if

    $$
    \alpha_2 < 5/14 - \epsilon^{'}
    $$

    then we can find vector $b$ through lattice basis reduction.

* The above is the attack detail given in the original paper for the case of two small decryption exponents, along with the analysis of the bound on $\alpha$.

## Case of Three Small Decryption Exponents

* For the case of three exponents, we additionally select $G_{1, 3}, W_1G_{2, 3}, W_2G_{1,3}$

    Then our vector b is
    
    $$B = \begin{pmatrix}
        k_1k_2k_3&d_1gk_2k_3&k_1d_2gk_3&d_1d_2g^2k_3&k_1k_2d_3g&k_1d_3g&k_2d_3g&d_1d_2d_3g^3
    \end{pmatrix}$$

    Then we can construct the lattice

    $$
    L_3 = \left(\begin{array}{rrrrrrrr}
            1 & -N & 0 & N^{2} & 0 & 0 & 0 & -N^{3} \\
            0 & e_{1} & -e_{1} & -N e_{1} & -e_{1} & 0 & N e_{1} & N^{2} e_{1} \\
            0 & 0 & e_{2} & -N e_{2} & 0 & N e_{2} & 0 & N^{2} e_{2} \\
            0 & 0 & 0 & e_{1} e_{2} & 0 & -e_{1} e_{2} & -e_{1} e_{2} & -N e_{1} e_{2} \\
            0 & 0 & 0 & 0 & e_{3} & -N e_{3} & -N e_{3} & N^{2} e_{3} \\
            0 & 0 & 0 & 0 & 0 & e_{1} e_{3} & 0 & -N e_{1} e_{3} \\
            0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} & -N e_{2} e_{3} \\
            0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3}
            \end{array}\right)
    $$

    where

    $$
    D = diag(\begin{array}{r}
        N^{\frac{3}{2}}&N&N^{a + \frac{3}{2}}&\sqrt{N}&N^{a + \frac{3}{2}}&N^{a + 1}&N^{a + 1}&1\end{array})
    $$

    Similarly we can obtain

    $$
    \Vert bL_2 	\Vert < \sqrt{8}N^{3/2+2\alpha_3}
    $$

    So when

    $$\alpha_3 < 2/5 - \epsilon^{'}$$

    the vector $b$ can be found through lattice basis reduction.

## Case of Four Small Decryption Exponents

* Additionally select $G_{1, 4}, W_1G_{2, 4}, G_{1, 2}G_{3,4}, G_{1, 3}G_{2, 4}, W_1W_2G_{3, 4}, W_1W_3G_{2, 4}, W_2W_3G_{1, 4}, W_1W_2W_3W_4$ for construction. Details are omitted here.

## Analysis

* The Extending Wiener's Attack, combined with the three examples above, has clearly illustrated the method details. However, it has not explained how to select composite relations. In fact, the appendix of the original paper provides the selection of composite relations and gives the expression for $\alpha_n$.

* In the appendix of the original paper, consider $n$ exponents $e_i$, giving $2^n$ different quantities $h_j$ (the number of $e_i$ in an expression). Before multiplying by $D$, the determinant of matrix $L_n$ is $N^{n2^{n-1}}$.

    The last relation $W_1W_2\dots W_n$ has a maximum of $N^{n/2 + n\alpha_n}$, so we know the maximum bound for any case, and we only need to increase the other values to this amount (i.e., construct the $D$ matrix).
    
    A new relation is introduced

    $$
    R_{u,v} = W_{i_1}\dots W_{i_u}G_{j_1, l_1}\dots G_{j_v, l_v}
    $$

    where $i_1,\dots,i_u,j_1,\dots,j_u,l_1,\dots,l_v$ are all distinct. Then there are at most $u + 2v$ exponents $e_i$, and our relation $R_{u,v}$ is at most $N^{u/2 + (u+v)\alpha_n}$. Also note that we need all coefficients to be roughly the same size, so we multiply certain equations by $k_i$, making the relation $R_{u, v} = N^{u/2 + (n-v)\alpha_n}$.

    Finally, we compute the difference between all sizes and the maximum size $N^{n/2 + n\alpha_n}$, and construct matrix $D$.

    This completes the construction of matrix $D$. Let the product of exponents inside matrix $D$ be $\beta_n = x+y\alpha_n$, then we have

    $$
    det(L_n) \approx N^{n2^{n-1} + x + y\alpha_n}
    $$

    Thus

    $$
    N^{n/2 + n\alpha_n} < (1/c_n)\left(N^{n2^{n-1} + x + y\alpha_n}\right)^{1/2^n}
    $$

    For small $c_n$, we have

    $$
    \alpha_n < \frac{x}{n2^n - y} - \epsilon^{'}
    $$

    So to make $\alpha_n$ larger, we need $x$ and $y$ to be larger, which means we should select more $v$ and smaller $u$. For example, in the $n=2$ case we select $W_1, G_{1, 2}, W_1W_2$ instead of $W_1, W_2, W_1W_2$ because the former gives $\beta_2 = 5/2 + \alpha$ while the latter gives $\beta_2 = 2$.

* At this point, the entire process of the Extending Wiener's Attack has been clearly explained: how to select composite relations, how to construct the lattice, how to construct matrix $D$, and how to solve. The end of the original paper also provides the chosen relations table for $n\le 5$.

    ![Chosen relations table](figure/extendwiener-chosen-relations.png)

    Here I also provide the chosen relations for $n\le8$ and the constructed matrix for $n=6$ for verifying whether you can write the logic code for selecting relations.

    ```txt
    -
    W(1)
    G(1, 2)
    W(1)W(2)
    G(1, 3)
    W(1)G(2, 3)
    W(2)G(1, 3)
    W(1)W(2)W(3)
    G(1, 4)
    W(1)G(2, 4)
    G(1, 2)G(3, 4)
    G(1, 3)G(2, 4)
    W(1)W(2)G(3, 4)
    W(1)W(3)G(2, 4)
    W(2)W(3)G(1, 4)
    W(1)W(2)W(3)W(4)
    G(1, 5)
    W(1)G(2, 5)
    G(1, 2)G(3, 5)
    G(1, 3)G(2, 5)
    G(1, 4)G(2, 5)
    W(1)W(2)G(3, 5)
    W(1)G(2, 3)G(4, 5)
    W(1)G(2, 4)G(3, 5)
    W(2)G(1, 3)G(4, 5)
    W(2)G(1, 4)G(3, 5)
    W(3)G(1, 4)G(2, 5)
    W(1)W(2)W(3)G(4, 5)
    W(1)W(2)W(4)G(3, 5)
    W(1)W(3)W(4)G(2, 5)
    W(2)W(3)W(4)G(1, 5)
    W(1)W(2)W(3)W(4)W(5)
    G(1, 6)
    W(1)G(2, 6)
    G(1, 2)G(3, 6)
    G(1, 3)G(2, 6)
    G(1, 4)G(2, 6)
    G(1, 5)G(2, 6)
    W(1)W(2)G(3, 6)
    W(1)G(2, 3)G(4, 6)
    W(1)G(2, 4)G(3, 6)
    W(1)G(2, 5)G(3, 6)
    G(1, 2)W(3)G(4, 6)
    G(1, 2)G(3, 4)G(5, 6)
    G(1, 2)G(3, 5)G(4, 6)
    G(1, 3)G(2, 4)G(5, 6)
    G(1, 3)G(2, 5)G(4, 6)
    G(1, 4)G(2, 5)G(3, 6)
    W(1)W(2)W(3)G(4, 6)
    W(1)W(2)G(3, 4)G(5, 6)
    W(1)W(2)G(3, 5)G(4, 6)
    W(1)W(3)G(2, 4)G(5, 6)
    W(1)W(3)G(2, 5)G(4, 6)
    W(1)W(4)G(2, 5)G(3, 6)
    W(2)W(3)G(1, 4)G(5, 6)
    W(2)W(3)G(1, 5)G(4, 6)
    W(2)W(4)G(1, 5)G(3, 6)
    W(3)W(4)G(1, 5)G(2, 6)
    W(1)W(2)W(3)W(4)G(5, 6)
    W(1)W(2)W(3)W(5)G(4, 6)
    W(1)W(2)W(4)W(5)G(3, 6)
    W(1)W(3)W(4)W(5)G(2, 6)
    W(2)W(3)W(4)W(5)G(1, 6)
    W(1)W(2)W(3)W(4)W(5)W(6)
    G(1, 7)
    W(1)G(2, 7)
    G(1, 2)G(3, 7)
    G(1, 3)G(2, 7)
    G(1, 4)G(2, 7)
    G(1, 5)G(2, 7)
    G(1, 6)G(2, 7)
    W(1)W(2)G(3, 7)
    W(1)G(2, 3)G(4, 7)
    W(1)G(2, 4)G(3, 7)
    W(1)G(2, 5)G(3, 7)
    W(1)G(2, 6)G(3, 7)
    G(1, 2)W(3)G(4, 7)
    G(1, 2)G(3, 4)G(5, 7)
    G(1, 2)G(3, 5)G(4, 7)
    G(1, 2)G(3, 6)G(4, 7)
    G(1, 3)G(2, 4)G(5, 7)
    G(1, 3)G(2, 5)G(4, 7)
    G(1, 3)G(2, 6)G(4, 7)
    G(1, 4)G(2, 5)G(3, 7)
    G(1, 4)G(2, 6)G(3, 7)
    G(1, 5)G(2, 6)G(3, 7)
    W(1)W(2)W(3)G(4, 7)
    W(1)W(2)G(3, 4)G(5, 7)
    W(1)W(2)G(3, 5)G(4, 7)
    W(1)W(2)G(3, 6)G(4, 7)
    W(1)G(2, 3)W(4)G(5, 7)
    W(1)G(2, 3)G(4, 5)G(6, 7)
    W(1)G(2, 3)G(4, 6)G(5, 7)
    W(1)G(2, 4)G(3, 5)G(6, 7)
    W(1)G(2, 4)G(3, 6)G(5, 7)
    W(1)G(2, 5)G(3, 6)G(4, 7)
    W(2)G(1, 3)W(4)G(5, 7)
    W(2)G(1, 3)G(4, 5)G(6, 7)
    W(2)G(1, 3)G(4, 6)G(5, 7)
    W(2)G(1, 4)G(3, 5)G(6, 7)
    W(2)G(1, 4)G(3, 6)G(5, 7)
    W(2)G(1, 5)G(3, 6)G(4, 7)
    W(3)G(1, 4)G(2, 5)G(6, 7)
    W(3)G(1, 4)G(2, 6)G(5, 7)
    W(3)G(1, 5)G(2, 6)G(4, 7)
    W(4)G(1, 5)G(2, 6)G(3, 7)
    W(1)W(2)W(3)W(4)G(5, 7)
    W(1)W(2)W(3)G(4, 5)G(6, 7)
    W(1)W(2)W(3)G(4, 6)G(5, 7)
    W(1)W(2)W(4)G(3, 5)G(6, 7)
    W(1)W(2)W(4)G(3, 6)G(5, 7)
    W(1)W(2)W(5)G(3, 6)G(4, 7)
    W(1)W(3)W(4)G(2, 5)G(6, 7)
    W(1)W(3)W(4)G(2, 6)G(5, 7)
    W(1)W(3)W(5)G(2, 6)G(4, 7)
    W(1)W(4)W(5)G(2, 6)G(3, 7)
    W(2)W(3)W(4)G(1, 5)G(6, 7)
    W(2)W(3)W(4)G(1, 6)G(5, 7)
    W(2)W(3)W(5)G(1, 6)G(4, 7)
    W(2)W(4)W(5)G(1, 6)G(3, 7)
    W(3)W(4)W(5)G(1, 6)G(2, 7)
    W(1)W(2)W(3)W(4)W(5)G(6, 7)
    W(1)W(2)W(3)W(4)W(6)G(5, 7)
    W(1)W(2)W(3)W(5)W(6)G(4, 7)
    W(1)W(2)W(4)W(5)W(6)G(3, 7)
    W(1)W(3)W(4)W(5)W(6)G(2, 7)
    W(2)W(3)W(4)W(5)W(6)G(1, 7)
    W(1)W(2)W(3)W(4)W(5)W(6)W(7)
    G(1, 8)
    W(1)G(2, 8)
    G(1, 2)G(3, 8)
    G(1, 3)G(2, 8)
    G(1, 4)G(2, 8)
    G(1, 5)G(2, 8)
    G(1, 6)G(2, 8)
    G(1, 7)G(2, 8)
    W(1)W(2)G(3, 8)
    W(1)G(2, 3)G(4, 8)
    W(1)G(2, 4)G(3, 8)
    W(1)G(2, 5)G(3, 8)
    W(1)G(2, 6)G(3, 8)
    W(1)G(2, 7)G(3, 8)
    G(1, 2)W(3)G(4, 8)
    G(1, 2)G(3, 4)G(5, 8)
    G(1, 2)G(3, 5)G(4, 8)
    G(1, 2)G(3, 6)G(4, 8)
    G(1, 2)G(3, 7)G(4, 8)
    G(1, 3)G(2, 4)G(5, 8)
    G(1, 3)G(2, 5)G(4, 8)
    G(1, 3)G(2, 6)G(4, 8)
    G(1, 3)G(2, 7)G(4, 8)
    G(1, 4)G(2, 5)G(3, 8)
    G(1, 4)G(2, 6)G(3, 8)
    G(1, 4)G(2, 7)G(3, 8)
    G(1, 5)G(2, 6)G(3, 8)
    G(1, 5)G(2, 7)G(3, 8)
    G(1, 6)G(2, 7)G(3, 8)
    W(1)W(2)W(3)G(4, 8)
    W(1)W(2)G(3, 4)G(5, 8)
    W(1)W(2)G(3, 5)G(4, 8)
    W(1)W(2)G(3, 6)G(4, 8)
    W(1)W(2)G(3, 7)G(4, 8)
    W(1)G(2, 3)W(4)G(5, 8)
    W(1)G(2, 3)G(4, 5)G(6, 8)
    W(1)G(2, 3)G(4, 6)G(5, 8)
    W(1)G(2, 3)G(4, 7)G(5, 8)
    W(1)G(2, 4)G(3, 5)G(6, 8)
    W(1)G(2, 4)G(3, 6)G(5, 8)
    W(1)G(2, 4)G(3, 7)G(5, 8)
    W(1)G(2, 5)G(3, 6)G(4, 8)
    W(1)G(2, 5)G(3, 7)G(4, 8)
    W(1)G(2, 6)G(3, 7)G(4, 8)
    G(1, 2)W(3)W(4)G(5, 8)
    G(1, 2)W(3)G(4, 5)G(6, 8)
    G(1, 2)W(3)G(4, 6)G(5, 8)
    G(1, 2)W(3)G(4, 7)G(5, 8)
    G(1, 2)G(3, 4)W(5)G(6, 8)
    G(1, 2)G(3, 4)G(5, 6)G(7, 8)
    G(1, 2)G(3, 4)G(5, 7)G(6, 8)
    G(1, 2)G(3, 5)G(4, 6)G(7, 8)
    G(1, 2)G(3, 5)G(4, 7)G(6, 8)
    G(1, 2)G(3, 6)G(4, 7)G(5, 8)
    G(1, 3)G(2, 4)W(5)G(6, 8)
    G(1, 3)G(2, 4)G(5, 6)G(7, 8)
    G(1, 3)G(2, 4)G(5, 7)G(6, 8)
    G(1, 3)G(2, 5)G(4, 6)G(7, 8)
    G(1, 3)G(2, 5)G(4, 7)G(6, 8)
    G(1, 3)G(2, 6)G(4, 7)G(5, 8)
    G(1, 4)G(2, 5)G(3, 6)G(7, 8)
    G(1, 4)G(2, 5)G(3, 7)G(6, 8)
    G(1, 4)G(2, 6)G(3, 7)G(5, 8)
    G(1, 5)G(2, 6)G(3, 7)G(4, 8)
    W(1)W(2)W(3)W(4)G(5, 8)
    W(1)W(2)W(3)G(4, 5)G(6, 8)
    W(1)W(2)W(3)G(4, 6)G(5, 8)
    W(1)W(2)W(3)G(4, 7)G(5, 8)
    W(1)W(2)G(3, 4)W(5)G(6, 8)
    W(1)W(2)G(3, 4)G(5, 6)G(7, 8)
    W(1)W(2)G(3, 4)G(5, 7)G(6, 8)
    W(1)W(2)G(3, 5)G(4, 6)G(7, 8)
    W(1)W(2)G(3, 5)G(4, 7)G(6, 8)
    W(1)W(2)G(3, 6)G(4, 7)G(5, 8)
    W(1)W(3)G(2, 4)W(5)G(6, 8)
    W(1)W(3)G(2, 4)G(5, 6)G(7, 8)
    W(1)W(3)G(2, 4)G(5, 7)G(6, 8)
    W(1)W(3)G(2, 5)G(4, 6)G(7, 8)
    W(1)W(3)G(2, 5)G(4, 7)G(6, 8)
    W(1)W(3)G(2, 6)G(4, 7)G(5, 8)
    W(1)W(4)G(2, 5)G(3, 6)G(7, 8)
    W(1)W(4)G(2, 5)G(3, 7)G(6, 8)
    W(1)W(4)G(2, 6)G(3, 7)G(5, 8)
    W(1)W(5)G(2, 6)G(3, 7)G(4, 8)
    W(2)W(3)G(1, 4)W(5)G(6, 8)
    W(2)W(3)G(1, 4)G(5, 6)G(7, 8)
    W(2)W(3)G(1, 4)G(5, 7)G(6, 8)
    W(2)W(3)G(1, 5)G(4, 6)G(7, 8)
    W(2)W(3)G(1, 5)G(4, 7)G(6, 8)
    W(2)W(3)G(1, 6)G(4, 7)G(5, 8)
    W(2)W(4)G(1, 5)G(3, 6)G(7, 8)
    W(2)W(4)G(1, 5)G(3, 7)G(6, 8)
    W(2)W(4)G(1, 6)G(3, 7)G(5, 8)
    W(2)W(5)G(1, 6)G(3, 7)G(4, 8)
    W(3)W(4)G(1, 5)G(2, 6)G(7, 8)
    W(3)W(4)G(1, 5)G(2, 7)G(6, 8)
    W(3)W(4)G(1, 6)G(2, 7)G(5, 8)
    W(3)W(5)G(1, 6)G(2, 7)G(4, 8)
    W(4)W(5)G(1, 6)G(2, 7)G(3, 8)
    W(1)W(2)W(3)W(4)W(5)G(6, 8)
    W(1)W(2)W(3)W(4)G(5, 6)G(7, 8)
    W(1)W(2)W(3)W(4)G(5, 7)G(6, 8)
    W(1)W(2)W(3)W(5)G(4, 6)G(7, 8)
    W(1)W(2)W(3)W(5)G(4, 7)G(6, 8)
    W(1)W(2)W(3)W(6)G(4, 7)G(5, 8)
    W(1)W(2)W(4)W(5)G(3, 6)G(7, 8)
    W(1)W(2)W(4)W(5)G(3, 7)G(6, 8)
    W(1)W(2)W(4)W(6)G(3, 7)G(5, 8)
    W(1)W(2)W(5)W(6)G(3, 7)G(4, 8)
    W(1)W(3)W(4)W(5)G(2, 6)G(7, 8)
    W(1)W(3)W(4)W(5)G(2, 7)G(6, 8)
    W(1)W(3)W(4)W(6)G(2, 7)G(5, 8)
    W(1)W(3)W(5)W(6)G(2, 7)G(4, 8)
    W(1)W(4)W(5)W(6)G(2, 7)G(3, 8)
    W(2)W(3)W(4)W(5)G(1, 6)G(7, 8)
    W(2)W(3)W(4)W(5)G(1, 7)G(6, 8)
    W(2)W(3)W(4)W(6)G(1, 7)G(5, 8)
    W(2)W(3)W(5)W(6)G(1, 7)G(4, 8)
    W(2)W(4)W(5)W(6)G(1, 7)G(3, 8)
    W(3)W(4)W(5)W(6)G(1, 7)G(2, 8)
    W(1)W(2)W(3)W(4)W(5)W(6)G(7, 8)
    W(1)W(2)W(3)W(4)W(5)W(7)G(6, 8)
    W(1)W(2)W(3)W(4)W(6)W(7)G(5, 8)
    W(1)W(2)W(3)W(5)W(6)W(7)G(4, 8)
    W(1)W(2)W(4)W(5)W(6)W(7)G(3, 8)
    W(1)W(3)W(4)W(5)W(6)W(7)G(2, 8)
    W(2)W(3)W(4)W(5)W(6)W(7)G(1, 8)
    W(1)W(2)W(3)W(4)W(5)W(6)W(7)W(8)
    ```

    $$
    \left(\begin{array}{rrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr}
    1 & -N & 0 & N^{2} & 0 & 0 & 0 & -N^{3} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{6} \\
    0 & e_{1} & -e_{1} & -N e_{1} & -e_{1} & 0 & N e_{1} & N^{2} e_{1} & -e_{1} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{1} & -N^{3} e_{1} & -e_{1} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{1} & N^{4} e_{1} & -e_{1} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{4} e_{1} & -N^{5} e_{1} \\
    0 & 0 & e_{2} & -N e_{2} & 0 & N e_{2} & 0 & N^{2} e_{2} & 0 & N e_{2} & 0 & 0 & 0 & -N^{2} e_{2} & 0 & -N^{3} e_{2} & 0 & N e_{2} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{2} & 0 & N^{4} e_{2} & 0 & N e_{2} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{4} e_{2} & 0 & -N^{5} e_{2} \\
    0 & 0 & 0 & e_{1} e_{2} & 0 & -e_{1} e_{2} & -e_{1} e_{2} & -N e_{1} e_{2} & 0 & -e_{1} e_{2} & 0 & e_{1} e_{2} & 0 & N e_{1} e_{2} & N e_{1} e_{2} & N^{2} e_{1} e_{2} & 0 & -e_{1} e_{2} & 0 & e_{1} e_{2} & e_{1} e_{2} & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{2} & 0 & 0 & -N^{2} e_{1} e_{2} & -N^{2} e_{1} e_{2} & -N^{3} e_{1} e_{2} & 0 & -e_{1} e_{2} & 0 & e_{1} e_{2} & e_{1} e_{2} & e_{1} e_{2} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{2} & 0 & 0 & 0 & N^{3} e_{1} e_{2} & N^{3} e_{1} e_{2} & N^{4} e_{1} e_{2} \\
    0 & 0 & 0 & 0 & e_{3} & -N e_{3} & -N e_{3} & N^{2} e_{3} & 0 & 0 & 0 & 0 & -N^{2} e_{3} & 0 & 0 & -N^{3} e_{3} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{3} & 0 & 0 & N^{4} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{4} e_{3} & 0 & 0 & -N^{5} e_{3} \\
    0 & 0 & 0 & 0 & 0 & e_{1} e_{3} & 0 & -N e_{1} e_{3} & 0 & 0 & e_{1} e_{3} & 0 & N e_{1} e_{3} & 0 & N e_{1} e_{3} & N^{2} e_{1} e_{3} & 0 & 0 & e_{1} e_{3} & 0 & 0 & N e_{1} e_{3} & 0 & 0 & 0 & -N e_{1} e_{3} & 0 & 0 & -N^{2} e_{1} e_{3} & 0 & -N^{2} e_{1} e_{3} & -N^{3} e_{1} e_{3} & 0 & 0 & e_{1} e_{3} & 0 & 0 & 0 & N e_{1} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{3} & 0 & 0 & 0 & N^{3} e_{1} e_{3} & 0 & N^{3} e_{1} e_{3} & N^{4} e_{1} e_{3} \\
    0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} & -N e_{2} e_{3} & 0 & 0 & -e_{2} e_{3} & -e_{2} e_{3} & N e_{2} e_{3} & N e_{2} e_{3} & 0 & N^{2} e_{2} e_{3} & 0 & 0 & -e_{2} e_{3} & -e_{2} e_{3} & 0 & N e_{2} e_{3} & 0 & -N e_{2} e_{3} & 0 & 0 & 0 & 0 & -N^{2} e_{2} e_{3} & -N^{2} e_{2} e_{3} & 0 & -N^{3} e_{2} e_{3} & 0 & 0 & -e_{2} e_{3} & -e_{2} e_{3} & 0 & 0 & N e_{2} e_{3} & 0 & -N e_{2} e_{3} & -N e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{2} e_{3} & N^{3} e_{2} e_{3} & 0 & N^{4} e_{2} e_{3} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} & -e_{1} e_{2} e_{3} & -e_{1} e_{2} e_{3} & -N e_{1} e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} & 0 & e_{1} e_{2} e_{3} & 0 & e_{1} e_{2} e_{3} & e_{1} e_{2} e_{3} & 0 & N e_{1} e_{2} e_{3} & N e_{1} e_{2} e_{3} & N e_{1} e_{2} e_{3} & N^{2} e_{1} e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} & 0 & e_{1} e_{2} e_{3} & e_{1} e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{2} e_{3} & 0 & 0 & -N e_{1} e_{2} e_{3} & -N e_{1} e_{2} e_{3} & 0 & 0 & -N^{2} e_{1} e_{2} e_{3} & -N^{2} e_{1} e_{2} e_{3} & -N^{2} e_{1} e_{2} e_{3} & -N^{3} e_{1} e_{2} e_{3} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{4} & -N e_{4} & 0 & 0 & N^{2} e_{4} & N^{2} e_{4} & N^{2} e_{4} & -N^{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{4} & 0 & 0 & 0 & N^{4} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{4} e_{4} & 0 & 0 & 0 & -N^{5} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{4} & -e_{1} e_{4} & -e_{1} e_{4} & -N e_{1} e_{4} & -N e_{1} e_{4} & 0 & N^{2} e_{1} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{4} & 0 & 0 & -N^{2} e_{1} e_{4} & 0 & 0 & -N^{2} e_{1} e_{4} & -N^{3} e_{1} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{4} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{1} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{4} & 0 & 0 & 0 & N^{3} e_{1} e_{4} & 0 & 0 & N^{3} e_{1} e_{4} & N^{4} e_{1} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{4} & 0 & -N e_{2} e_{4} & 0 & -N e_{2} e_{4} & N^{2} e_{2} e_{4} & 0 & 0 & 0 & 0 & -e_{2} e_{4} & 0 & -N e_{2} e_{4} & 0 & 0 & 0 & N e_{2} e_{4} & -N^{2} e_{2} e_{4} & 0 & -N^{2} e_{2} e_{4} & 0 & -N^{3} e_{2} e_{4} & 0 & 0 & 0 & 0 & -e_{2} e_{4} & 0 & 0 & -N e_{2} e_{4} & 0 & 0 & N e_{2} e_{4} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{2} e_{4} & 0 & 0 & 0 & N^{2} e_{2} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{2} e_{4} & 0 & N^{3} e_{2} e_{4} & 0 & N^{4} e_{2} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{4} & 0 & -N e_{3} e_{4} & -N e_{3} e_{4} & N^{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & N e_{3} e_{4} & N e_{3} e_{4} & N e_{3} e_{4} & N e_{3} e_{4} & 0 & -N^{2} e_{3} e_{4} & -N^{2} e_{3} e_{4} & 0 & 0 & -N^{3} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N e_{3} e_{4} & N e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{3} e_{4} & 0 & N^{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{3} e_{3} e_{4} & N^{3} e_{3} e_{4} & 0 & 0 & N^{4} e_{3} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} & 0 & 0 & -N e_{1} e_{2} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} & 0 & e_{1} e_{2} e_{4} & 0 & 0 & N e_{1} e_{2} e_{4} & 0 & N e_{1} e_{2} e_{4} & N e_{1} e_{2} e_{4} & N^{2} e_{1} e_{2} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{4} & 0 & N e_{1} e_{2} e_{4} & 0 & 0 & 0 & -N e_{1} e_{2} e_{4} & 0 & 0 & -N e_{1} e_{2} e_{4} & 0 & -N e_{1} e_{2} e_{4} & 0 & -N^{2} e_{1} e_{2} e_{4} & 0 & -N^{2} e_{1} e_{2} e_{4} & -N^{2} e_{1} e_{2} e_{4} & -N^{3} e_{1} e_{2} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{4} & 0 & -N e_{1} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{3} e_{4} & -e_{1} e_{3} e_{4} & 0 & 0 & 0 & N e_{1} e_{3} e_{4} & N e_{1} e_{3} e_{4} & 0 & N e_{1} e_{3} e_{4} & N^{2} e_{1} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{3} e_{4} & -e_{1} e_{3} e_{4} & 0 & e_{1} e_{3} e_{4} & 0 & -e_{1} e_{3} e_{4} & 0 & 0 & 0 & N e_{1} e_{3} e_{4} & 0 & -N e_{1} e_{3} e_{4} & 0 & 0 & 0 & 0 & -N e_{1} e_{3} e_{4} & -N e_{1} e_{3} e_{4} & 0 & 0 & -N^{2} e_{1} e_{3} e_{4} & -N^{2} e_{1} e_{3} e_{4} & 0 & -N^{2} e_{1} e_{3} e_{4} & -N^{3} e_{1} e_{3} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{4} & -N e_{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{2} e_{3} e_{4} & -e_{2} e_{3} e_{4} & -e_{2} e_{3} e_{4} & N e_{2} e_{3} e_{4} & N e_{2} e_{3} e_{4} & N e_{2} e_{3} e_{4} & 0 & N^{2} e_{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{2} e_{3} e_{4} & 0 & e_{2} e_{3} e_{4} & 0 & e_{2} e_{3} e_{4} & e_{2} e_{3} e_{4} & N e_{2} e_{3} e_{4} & 0 & -N e_{2} e_{3} e_{4} & 0 & -N e_{2} e_{3} e_{4} & -N e_{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{2} e_{3} e_{4} & -N^{2} e_{2} e_{3} e_{4} & -N^{2} e_{2} e_{3} e_{4} & 0 & -N^{3} e_{2} e_{3} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} e_{4} & -e_{1} e_{2} e_{3} e_{4} & -e_{1} e_{2} e_{3} e_{4} & -e_{1} e_{2} e_{3} e_{4} & -N e_{1} e_{2} e_{3} e_{4} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} e_{4} & 0 & e_{1} e_{2} e_{3} e_{4} & 0 & e_{1} e_{2} e_{3} e_{4} & e_{1} e_{2} e_{3} e_{4} & 0 & e_{1} e_{2} e_{3} e_{4} & e_{1} e_{2} e_{3} e_{4} & e_{1} e_{2} e_{3} e_{4} & 0 & N e_{1} e_{2} e_{3} e_{4} & N e_{1} e_{2} e_{3} e_{4} & N e_{1} e_{2} e_{3} e_{4} & N e_{1} e_{2} e_{3} e_{4} & N^{2} e_{1} e_{2} e_{3} e_{4} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{5} & -N e_{5} & 0 & 0 & 0 & N^{2} e_{5} & 0 & 0 & 0 & 0 & 0 & -N^{3} e_{5} & -N^{3} e_{5} & -N^{3} e_{5} & -N^{3} e_{5} & N^{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{4} e_{5} & 0 & 0 & 0 & 0 & -N^{5} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{5} & -e_{1} e_{5} & -e_{1} e_{5} & -e_{1} e_{5} & -N e_{1} e_{5} & 0 & 0 & N e_{1} e_{5} & N e_{1} e_{5} & N e_{1} e_{5} & N^{2} e_{1} e_{5} & N^{2} e_{1} e_{5} & N^{2} e_{1} e_{5} & 0 & -N^{3} e_{1} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{5} & 0 & 0 & 0 & N^{3} e_{1} e_{5} & 0 & 0 & 0 & N^{3} e_{1} e_{5} & N^{4} e_{1} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{5} & 0 & 0 & -N e_{2} e_{5} & N e_{2} e_{5} & N e_{2} e_{5} & 0 & 0 & 0 & N^{2} e_{2} e_{5} & N^{2} e_{2} e_{5} & 0 & N^{2} e_{2} e_{5} & -N^{3} e_{2} e_{5} & 0 & 0 & 0 & 0 & 0 & -e_{2} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{2} e_{5} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{2} e_{5} & N^{3} e_{2} e_{5} & 0 & 0 & N^{3} e_{2} e_{5} & 0 & N^{4} e_{2} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{5} & 0 & 0 & -N e_{3} e_{5} & 0 & -N e_{3} e_{5} & 0 & 0 & N^{2} e_{3} e_{5} & 0 & N^{2} e_{3} e_{5} & N^{2} e_{3} e_{5} & -N^{3} e_{3} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N e_{3} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{3} e_{5} & 0 & 0 & 0 & -N^{2} e_{3} e_{5} & 0 & 0 & -N^{2} e_{3} e_{5} & 0 & N^{3} e_{3} e_{5} & 0 & N^{3} e_{3} e_{5} & 0 & 0 & N^{4} e_{3} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{4} e_{5} & 0 & 0 & -N e_{4} e_{5} & 0 & -N e_{4} e_{5} & -N e_{4} e_{5} & 0 & N^{2} e_{4} e_{5} & N^{2} e_{4} e_{5} & N^{2} e_{4} e_{5} & -N^{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{4} e_{5} & -N^{2} e_{4} e_{5} & -N^{2} e_{4} e_{5} & -N^{2} e_{4} e_{5} & 0 & -N^{2} e_{4} e_{5} & -N^{2} e_{4} e_{5} & 0 & 0 & N^{3} e_{4} e_{5} & N^{3} e_{4} e_{5} & 0 & 0 & 0 & N^{4} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{5} & -e_{1} e_{2} e_{5} & -e_{1} e_{2} e_{5} & -e_{1} e_{2} e_{5} & -e_{1} e_{2} e_{5} & 0 & -N e_{1} e_{2} e_{5} & -N e_{1} e_{2} e_{5} & 0 & 0 & N^{2} e_{1} e_{2} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{5} & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{2} e_{5} & 0 & 0 & -N e_{1} e_{2} e_{5} & 0 & 0 & 0 & -N^{2} e_{1} e_{2} e_{5} & 0 & 0 & -N^{2} e_{1} e_{2} e_{5} & -N^{2} e_{1} e_{2} e_{5} & -N^{3} e_{1} e_{2} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{5} & 0 & 0 & 0 & -e_{1} e_{3} e_{5} & -N e_{1} e_{3} e_{5} & 0 & -N e_{1} e_{3} e_{5} & 0 & N^{2} e_{1} e_{3} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{3} e_{5} & 0 & -e_{1} e_{3} e_{5} & 0 & 0 & 0 & e_{1} e_{3} e_{5} & 0 & -N e_{1} e_{3} e_{5} & 0 & 0 & 0 & N e_{1} e_{3} e_{5} & -N e_{1} e_{3} e_{5} & 0 & 0 & 0 & -N^{2} e_{1} e_{3} e_{5} & 0 & -N^{2} e_{1} e_{3} e_{5} & 0 & -N^{2} e_{1} e_{3} e_{5} & -N^{3} e_{1} e_{3} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{4} e_{5} & 0 & 0 & 0 & 0 & -N e_{1} e_{4} e_{5} & -N e_{1} e_{4} e_{5} & 0 & N^{2} e_{1} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{4} e_{5} & e_{1} e_{4} e_{5} & e_{1} e_{4} e_{5} & e_{1} e_{4} e_{5} & 0 & 0 & N e_{1} e_{4} e_{5} & N e_{1} e_{4} e_{5} & N e_{1} e_{4} e_{5} & N e_{1} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{1} e_{4} e_{5} & -N^{2} e_{1} e_{4} e_{5} & 0 & 0 & -N^{2} e_{1} e_{4} e_{5} & -N^{3} e_{1} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{5} & 0 & 0 & -N e_{2} e_{3} e_{5} & 0 & 0 & -N e_{2} e_{3} e_{5} & N^{2} e_{2} e_{3} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{5} & 0 & e_{2} e_{3} e_{5} & 0 & 0 & 0 & -N e_{2} e_{3} e_{5} & 0 & -N e_{2} e_{3} e_{5} & 0 & 0 & 0 & 0 & N e_{2} e_{3} e_{5} & N e_{2} e_{3} e_{5} & -N^{2} e_{2} e_{3} e_{5} & 0 & -N^{2} e_{2} e_{3} e_{5} & -N^{2} e_{2} e_{3} e_{5} & 0 & -N^{3} e_{2} e_{3} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{4} e_{5} & 0 & 0 & -N e_{2} e_{4} e_{5} & 0 & -N e_{2} e_{4} e_{5} & N^{2} e_{2} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{2} e_{4} e_{5} & -e_{2} e_{4} e_{5} & 0 & 0 & 0 & 0 & N e_{2} e_{4} e_{5} & N e_{2} e_{4} e_{5} & 0 & 0 & 0 & N e_{2} e_{4} e_{5} & N e_{2} e_{4} e_{5} & 0 & N e_{2} e_{4} e_{5} & -N^{2} e_{2} e_{4} e_{5} & -N^{2} e_{2} e_{4} e_{5} & 0 & -N^{2} e_{2} e_{4} e_{5} & 0 & -N^{3} e_{2} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{4} e_{5} & 0 & 0 & -N e_{3} e_{4} e_{5} & -N e_{3} e_{4} e_{5} & N^{2} e_{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{3} e_{4} e_{5} & -e_{3} e_{4} e_{5} & -e_{3} e_{4} e_{5} & 0 & 0 & 0 & N e_{3} e_{4} e_{5} & N e_{3} e_{4} e_{5} & N e_{3} e_{4} e_{5} & N e_{3} e_{4} e_{5} & N e_{3} e_{4} e_{5} & N e_{3} e_{4} e_{5} & 0 & -N^{2} e_{3} e_{4} e_{5} & -N^{2} e_{3} e_{4} e_{5} & -N^{2} e_{3} e_{4} e_{5} & 0 & 0 & -N^{3} e_{3} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{5} & 0 & 0 & 0 & -N e_{1} e_{2} e_{3} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{5} & 0 & e_{1} e_{2} e_{3} e_{5} & 0 & 0 & e_{1} e_{2} e_{3} e_{5} & 0 & 0 & 0 & N e_{1} e_{2} e_{3} e_{5} & 0 & N e_{1} e_{2} e_{3} e_{5} & N e_{1} e_{2} e_{3} e_{5} & N e_{1} e_{2} e_{3} e_{5} & N^{2} e_{1} e_{2} e_{3} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} e_{5} & 0 & 0 & -N e_{1} e_{2} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{4} e_{5} & -e_{1} e_{2} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N e_{1} e_{2} e_{4} e_{5} & N e_{1} e_{2} e_{4} e_{5} & 0 & N e_{1} e_{2} e_{4} e_{5} & N e_{1} e_{2} e_{4} e_{5} & N^{2} e_{1} e_{2} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{4} e_{5} & 0 & -N e_{1} e_{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{3} e_{4} e_{5} & -e_{1} e_{3} e_{4} e_{5} & -e_{1} e_{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & N e_{1} e_{3} e_{4} e_{5} & N e_{1} e_{3} e_{4} e_{5} & N e_{1} e_{3} e_{4} e_{5} & 0 & N e_{1} e_{3} e_{4} e_{5} & N^{2} e_{1} e_{3} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{4} e_{5} & -N e_{2} e_{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{2} e_{3} e_{4} e_{5} & -e_{2} e_{3} e_{4} e_{5} & -e_{2} e_{3} e_{4} e_{5} & -e_{2} e_{3} e_{4} e_{5} & N e_{2} e_{3} e_{4} e_{5} & N e_{2} e_{3} e_{4} e_{5} & N e_{2} e_{3} e_{4} e_{5} & N e_{2} e_{3} e_{4} e_{5} & 0 & N^{2} e_{2} e_{3} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{4} e_{5} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{2} e_{3} e_{4} e_{5} & -e_{1} e_{2} e_{3} e_{4} e_{5} & -e_{1} e_{2} e_{3} e_{4} e_{5} & -e_{1} e_{2} e_{3} e_{4} e_{5} & -e_{1} e_{2} e_{3} e_{4} e_{5} & -N e_{1} e_{2} e_{3} e_{4} e_{5} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{6} & -N e_{6} & 0 & 0 & 0 & 0 & N^{2} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{3} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{4} e_{6} & N^{4} e_{6} & N^{4} e_{6} & N^{4} e_{6} & N^{4} e_{6} & -N^{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{6} & -e_{1} e_{6} & -e_{1} e_{6} & -e_{1} e_{6} & -e_{1} e_{6} & -N e_{1} e_{6} & 0 & 0 & 0 & N e_{1} e_{6} & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{6} & 0 & 0 & 0 & 0 & 0 & -N^{2} e_{1} e_{6} & -N^{2} e_{1} e_{6} & -N^{2} e_{1} e_{6} & -N^{2} e_{1} e_{6} & -N^{3} e_{1} e_{6} & -N^{3} e_{1} e_{6} & -N^{3} e_{1} e_{6} & -N^{3} e_{1} e_{6} & 0 & N^{4} e_{1} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{6} & 0 & 0 & 0 & -N e_{2} e_{6} & N e_{2} e_{6} & N e_{2} e_{6} & N e_{2} e_{6} & -N e_{2} e_{6} & 0 & 0 & 0 & 0 & 0 & N^{2} e_{2} e_{6} & 0 & 0 & -N^{2} e_{2} e_{6} & -N^{2} e_{2} e_{6} & -N^{2} e_{2} e_{6} & 0 & 0 & 0 & 0 & -N^{3} e_{2} e_{6} & -N^{3} e_{2} e_{6} & -N^{3} e_{2} e_{6} & 0 & -N^{3} e_{2} e_{6} & N^{4} e_{2} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{6} & 0 & 0 & 0 & -N e_{3} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{3} e_{6} & -N^{2} e_{3} e_{6} & -N^{2} e_{3} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N^{3} e_{3} e_{6} & -N^{3} e_{3} e_{6} & 0 & -N^{3} e_{3} e_{6} & -N^{3} e_{3} e_{6} & N^{4} e_{3} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{4} e_{6} & 0 & 0 & 0 & -N e_{4} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{4} e_{6} & 0 & N^{2} e_{4} e_{6} & 0 & 0 & N^{2} e_{4} e_{6} & 0 & 0 & 0 & -N^{3} e_{4} e_{6} & 0 & -N^{3} e_{4} e_{6} & -N^{3} e_{4} e_{6} & -N^{3} e_{4} e_{6} & N^{4} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{5} e_{6} & 0 & 0 & 0 & -N e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & N^{2} e_{5} e_{6} & 0 & N^{2} e_{5} e_{6} & N^{2} e_{5} e_{6} & 0 & N^{2} e_{5} e_{6} & N^{2} e_{5} e_{6} & N^{2} e_{5} e_{6} & 0 & -N^{3} e_{5} e_{6} & -N^{3} e_{5} e_{6} & -N^{3} e_{5} e_{6} & -N^{3} e_{5} e_{6} & N^{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{6} & -e_{1} e_{2} e_{6} & -e_{1} e_{2} e_{6} & -e_{1} e_{2} e_{6} & 0 & 0 & 0 & e_{1} e_{2} e_{6} & e_{1} e_{2} e_{6} & e_{1} e_{2} e_{6} & -N e_{1} e_{2} e_{6} & 0 & 0 & N e_{1} e_{2} e_{6} & N e_{1} e_{2} e_{6} & N e_{1} e_{2} e_{6} & N e_{1} e_{2} e_{6} & N e_{1} e_{2} e_{6} & N e_{1} e_{2} e_{6} & 0 & N^{2} e_{1} e_{2} e_{6} & N^{2} e_{1} e_{2} e_{6} & N^{2} e_{1} e_{2} e_{6} & 0 & 0 & -N^{3} e_{1} e_{2} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{6} & 0 & 0 & -e_{1} e_{3} e_{6} & e_{1} e_{3} e_{6} & e_{1} e_{3} e_{6} & 0 & 0 & 0 & -N e_{1} e_{3} e_{6} & N e_{1} e_{3} e_{6} & N e_{1} e_{3} e_{6} & 0 & 0 & 0 & N e_{1} e_{3} e_{6} & N e_{1} e_{3} e_{6} & 0 & N e_{1} e_{3} e_{6} & N^{2} e_{1} e_{3} e_{6} & N^{2} e_{1} e_{3} e_{6} & 0 & N^{2} e_{1} e_{3} e_{6} & 0 & -N^{3} e_{1} e_{3} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{4} e_{6} & 0 & 0 & -e_{1} e_{4} e_{6} & 0 & -e_{1} e_{4} e_{6} & 0 & 0 & 0 & -N e_{1} e_{4} e_{6} & 0 & -N e_{1} e_{4} e_{6} & 0 & 0 & 0 & 0 & N e_{1} e_{4} e_{6} & N e_{1} e_{4} e_{6} & N^{2} e_{1} e_{4} e_{6} & 0 & N^{2} e_{1} e_{4} e_{6} & N^{2} e_{1} e_{4} e_{6} & 0 & -N^{3} e_{1} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{5} e_{6} & 0 & 0 & -e_{1} e_{5} e_{6} & 0 & -e_{1} e_{5} e_{6} & -e_{1} e_{5} e_{6} & 0 & 0 & -N e_{1} e_{5} e_{6} & 0 & -N e_{1} e_{5} e_{6} & -N e_{1} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & N^{2} e_{1} e_{5} e_{6} & N^{2} e_{1} e_{5} e_{6} & N^{2} e_{1} e_{5} e_{6} & 0 & -N^{3} e_{1} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{6} & -e_{2} e_{3} e_{6} & -e_{2} e_{3} e_{6} & -e_{2} e_{3} e_{6} & -e_{2} e_{3} e_{6} & 0 & -N e_{2} e_{3} e_{6} & N e_{2} e_{3} e_{6} & N e_{2} e_{3} e_{6} & N e_{2} e_{3} e_{6} & N e_{2} e_{3} e_{6} & 0 & 0 & 0 & 0 & 0 & N^{2} e_{2} e_{3} e_{6} & N^{2} e_{2} e_{3} e_{6} & 0 & 0 & N^{2} e_{2} e_{3} e_{6} & -N^{3} e_{2} e_{3} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{4} e_{6} & 0 & 0 & 0 & -e_{2} e_{4} e_{6} & 0 & -N e_{2} e_{4} e_{6} & 0 & 0 & 0 & N e_{2} e_{4} e_{6} & -N e_{2} e_{4} e_{6} & 0 & 0 & 0 & N^{2} e_{2} e_{4} e_{6} & 0 & N^{2} e_{2} e_{4} e_{6} & 0 & N^{2} e_{2} e_{4} e_{6} & -N^{3} e_{2} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & -N e_{2} e_{5} e_{6} & 0 & 0 & 0 & 0 & -N e_{2} e_{5} e_{6} & -N e_{2} e_{5} e_{6} & 0 & 0 & N^{2} e_{2} e_{5} e_{6} & N^{2} e_{2} e_{5} e_{6} & 0 & N^{2} e_{2} e_{5} e_{6} & -N^{3} e_{2} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{4} e_{6} & 0 & 0 & 0 & 0 & 0 & -N e_{3} e_{4} e_{6} & 0 & 0 & -N e_{3} e_{4} e_{6} & 0 & 0 & 0 & N^{2} e_{3} e_{4} e_{6} & 0 & 0 & N^{2} e_{3} e_{4} e_{6} & N^{2} e_{3} e_{4} e_{6} & -N^{3} e_{3} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & -N e_{3} e_{5} e_{6} & 0 & 0 & -N e_{3} e_{5} e_{6} & 0 & -N e_{3} e_{5} e_{6} & 0 & N^{2} e_{3} e_{5} e_{6} & 0 & N^{2} e_{3} e_{5} e_{6} & N^{2} e_{3} e_{5} e_{6} & -N^{3} e_{3} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{4} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & -N e_{4} e_{5} e_{6} & 0 & 0 & -N e_{4} e_{5} e_{6} & -N e_{4} e_{5} e_{6} & 0 & 0 & N^{2} e_{4} e_{5} e_{6} & N^{2} e_{4} e_{5} e_{6} & N^{2} e_{4} e_{5} e_{6} & -N^{3} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{6} & -e_{1} e_{2} e_{3} e_{6} & -e_{1} e_{2} e_{3} e_{6} & -e_{1} e_{2} e_{3} e_{6} & -e_{1} e_{2} e_{3} e_{6} & 0 & -e_{1} e_{2} e_{3} e_{6} & -e_{1} e_{2} e_{3} e_{6} & 0 & 0 & -N e_{1} e_{2} e_{3} e_{6} & -N e_{1} e_{2} e_{3} e_{6} & 0 & 0 & 0 & N^{2} e_{1} e_{2} e_{3} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} e_{6} & 0 & 0 & 0 & -e_{1} e_{2} e_{4} e_{6} & 0 & 0 & -e_{1} e_{2} e_{4} e_{6} & 0 & -N e_{1} e_{2} e_{4} e_{6} & 0 & -N e_{1} e_{2} e_{4} e_{6} & 0 & 0 & N^{2} e_{1} e_{2} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{2} e_{5} e_{6} & -N e_{1} e_{2} e_{5} e_{6} & 0 & 0 & N^{2} e_{1} e_{2} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{4} e_{6} & 0 & 0 & 0 & 0 & 0 & -e_{1} e_{3} e_{4} e_{6} & -N e_{1} e_{3} e_{4} e_{6} & 0 & 0 & -N e_{1} e_{3} e_{4} e_{6} & 0 & N^{2} e_{1} e_{3} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{3} e_{5} e_{6} & 0 & -N e_{1} e_{3} e_{5} e_{6} & 0 & N^{2} e_{1} e_{3} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{4} e_{5} e_{6} & 0 & 0 & 0 & 0 & 0 & 0 & -N e_{1} e_{4} e_{5} e_{6} & -N e_{1} e_{4} e_{5} e_{6} & 0 & N^{2} e_{1} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{4} e_{6} & 0 & 0 & 0 & -N e_{2} e_{3} e_{4} e_{6} & 0 & 0 & 0 & -N e_{2} e_{3} e_{4} e_{6} & N^{2} e_{2} e_{3} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{5} e_{6} & 0 & 0 & 0 & -N e_{2} e_{3} e_{5} e_{6} & 0 & 0 & -N e_{2} e_{3} e_{5} e_{6} & N^{2} e_{2} e_{3} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{4} e_{5} e_{6} & 0 & 0 & 0 & -N e_{2} e_{4} e_{5} e_{6} & 0 & -N e_{2} e_{4} e_{5} e_{6} & N^{2} e_{2} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{3} e_{4} e_{5} e_{6} & 0 & 0 & 0 & -N e_{3} e_{4} e_{5} e_{6} & -N e_{3} e_{4} e_{5} e_{6} & N^{2} e_{3} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{4} e_{6} & 0 & 0 & 0 & 0 & -N e_{1} e_{2} e_{3} e_{4} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{5} e_{6} & 0 & 0 & 0 & -N e_{1} e_{2} e_{3} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{4} e_{5} e_{6} & 0 & 0 & -N e_{1} e_{2} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{3} e_{4} e_{5} e_{6} & 0 & -N e_{1} e_{3} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{2} e_{3} e_{4} e_{5} e_{6} & -N e_{2} e_{3} e_{4} e_{5} e_{6} \\
    0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & 0 & e_{1} e_{2} e_{3} e_{4} e_{5} e_{6}
    \end{array}\right)
    $$

## Open Discussion

* Currently, Extending Wiener's Attack problems are all *template problems* for $n=2$ or $n=3$. For higher dimensions, automated scripts can be written to complete the steps of automatically selecting relations, automatically constructing lattices, etc. For example, the above content was automatically generated. However, for each increment of $n$, the matrix grows exponentially since it is a $2^n * 2^n$ matrix. At this point, directly calling `LLL()` in `sagemath` becomes very slow — around $n=8$ it can no longer finish. I have tried to find parallel LLL algorithms on CUDA or other optimized implementations, but only found papers without source code.

    If you have research in this area or better optimization methods, feel free to contact me ([Xenny](https://github.com/X3NNY)) for further in-depth discussion.

## EXP

* Considering that not everyone needs to deeply study the Extending Wiener's Attack, here is the EXP for $n=2$ for practical use

    ```python
    e1 = ...
    e2 = ...
    N = ...
    a = 5/14
    D = diagonal_matrix(ZZ, [N, int(N^(1/2)), int(N^(1+a)), 1])
    M = matrix(ZZ, [[1, -N, 0, N^2], [0, e1, -e1, -e1*N], [0, 0, e2, -e2*N], [0, 0, 0, e1*e2]])*D
    L = M.LLL()
    t = vector(ZZ, L[0])
    x = t * M^(-1)
    phi = int(x[1]/x[0]*e1)
    ```

## References

* [Extending Wiener's Attack in the Presence of Many Decrypting Exponents](https://www.sci-hub.ren/https://link.springer.com/chapter/10.1007/3-540-46701-7_14)
* [并行LLL算法研究综述](https://arcnl.org/jchen/download/survey_plll.pdf)
* [Factoring Polynomials with Rational Coefficients](https://www.math.leidenuniv.nl/~hwl/PUBLICATIONS/1982f/art.pdf)
* [A PARALLEL JACOBI-TYPE LATTICE BASIS REDUCTION
ALGORITHM](https://www.cas.mcmaster.ca/~qiao/publications/JQ14.pdf)