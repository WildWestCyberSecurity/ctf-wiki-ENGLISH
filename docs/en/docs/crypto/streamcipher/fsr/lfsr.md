# Linear Feedback Shift Register - LFSR

## Introduction

The feedback function of a linear feedback shift register is generally as follows:

$a_{i+n}=\sum\limits_{j=1}^{n}c_ja_{i+n-j}$

where $c_j$ are all in some finite field $F_q$.

Since the linear space is a linear transformation, we can determine this linear transformation to be:

$$ \begin{align*}
&\left[
  a_{i+1},a_{i+2},a_{i+3}, ...,a_{i+n}
\right]\\\\=&\left[
  a_{i},a_{i+1},a_{i+2}, ...,a_{i+n-1}
\right]\left[ \begin{matrix} 0   & 0      & \cdots & 0 & c_n     \\ 1   & 0      & \cdots & 0 & c_{n-1}  \\ 0   & 1      & \cdots & 0 & c_{n-2}\\\vdots & \vdots & \ddots & \vdots \\ 0   & 0      & \cdots & 1 & c_1     \\ \end{matrix} \right]\\\\=&\left[
  a_{0},a_{1},a_{2}, ...,a_{n-1}
\right]\left[ \begin{matrix} 0   & 0      & \cdots & 0 & c_n     \\ 1   & 0      & \cdots & 0 & c_{n-1}  \\ 0   & 1      & \cdots & 0 & c_{n-2}\\\vdots & \vdots & \ddots & \vdots \\ 0   & 0      & \cdots & 1 & c_1     \\ \end{matrix} \right]^{i+1}
\end{align*} $$

Furthermore, we can derive its characteristic polynomial as:

$f(x)=x^n-\sum\limits_{i=1}^{n}c_ix^{n-i}$

At the same time, we define its reciprocal polynomial as:

$\overline f(x)=x^nf(\frac{1}{x})=1-\sum\limits_{i=1}^{n}c_ix^{i}$

We also call the reciprocal polynomial the connection polynomial of the linear feedback shift register.

There are some theorems here that we should note; interested readers can derive them on their own.

## Characteristic Polynomial and Generating Function

Given the characteristic polynomial of an n-stage linear feedback shift register, the generating function of the corresponding sequence is:

$A(x)=\frac{p(x)}{\overline f(x)}$

where $p(x)=\sum\limits_{i=1}^{n}(c_{n-i}x^{n-i}\sum\limits_{j=1}^{i}a_jx^{j-1})$. It can be seen that p(x) is entirely determined by the initial state and the coefficients of the feedback function.

## Sequence Period and Generating Function

The period of a sequence is the period of the denominator of its generating function in irreducible proper fraction form.

For an n-stage linear feedback shift register, the maximum period is $2^{n}-1$ (excluding the all-zero state). A sequence that achieves the maximum period is generally called an m-sequence.

## Special Properties

- Adding two sequences together yields a new sequence whose period is the sum of the periods of the two sequences.
- A sequence is an n-stage m-sequence if and only if its minimal polynomial is a primitive polynomial of degree n.

## B-M Algorithm

Generally, we can consider LFSR from two perspectives:

- Key generation perspective: We generally want to use LFSRs with the lowest possible degree to generate sequences with large periods and good randomness.
- Cryptanalysis perspective: Given a sequence of length n, how to construct an LFSR with the smallest possible degree to generate it. This is essentially the origin of the B-M algorithm.

Generally, we define the linear complexity of a sequence as follows:

- If s is an all-zero sequence, its linear complexity is 0.
- If no LFSR can generate s, its linear complexity is infinity.
- Otherwise, the linear complexity of s is the minimum degree L(s) of an LFSR that generates it.

The B-M algorithm requires knowledge of a sequence of length 2n. Its complexity:

- Time complexity: O(n^2) bit operations
- Space complexity: O(n) bits.

Details of the B-M algorithm will be added later; it is currently being studied.

However, if we know a sequence of length 2n, we can also use a somewhat brute-force method to recover the original sequence. Suppose the known sequence is $a_1,...,a_{2n}$. We can set:

$S_1=(a_1,...,a_n)$

$S_2=(a_2,...,a_{n+1})$

....

$S_{n+1}=(a_{n+1},...,a_{2n})$

Then we can construct matrix $X=(S_1,...,S_n)$, and:

$S_{n+1}=(c_n,...,c_1)X$

Therefore:

$(c_n,...,c_1)=S_{n+1}X^{-1}$

This gives us the LFSR feedback expression, from which we can then derive the initialization seed.

## 2018 强网杯 (Qiangwang Cup) streamgame1

Let's briefly look at the challenge:

```python
from flag import flag
assert flag.startswith("flag{")
assert flag.endswith("}")
assert len(flag)==25

def lfsr(R,mask):
    output = (R << 1) & 0xffffff
    i=(R&mask)&0xffffff
    lastbit=0
    while i!=0:
        lastbit^=(i&1)
        i=i>>1
    output^=lastbit
    return (output,lastbit)



R=int(flag[5:-1],2)
mask    =   0b1010011000100011100

f=open("key","ab")
for i in range(12):
    tmp=0
    for j in range(8):
        (R,out)=lfsr(R,mask)
        tmp=(tmp << 1)^out
    f.write(chr(tmp))
f.close()
```

We can see that the flag length is 25-5-1=19, so we can brute-force enumerate. The result:

```shell
➜  2018-强网杯-streamgame1 git:(master) ✗ python exp.py
12
0b1110101100001101011
```

Therefore the flag is flag{1110101100001101011}.

## 2018 CISCN Preliminary oldstreamgame

Let's briefly look at the challenge:

```shell
flag = "flag{xxxxxxxxxxxxxxxx}"
assert flag.startswith("flag{")
assert flag.endswith("}")
assert len(flag)==14

def lfsr(R,mask):
    output = (R << 1) & 0xffffffff
    i=(R&mask)&0xffffffff
    lastbit=0
    while i!=0:
        lastbit^=(i&1)
        i=i>>1
    output^=lastbit
    return (output,lastbit)

R=int(flag[5:-1],16)
mask = 0b10100100000010000000100010010100

f=open("key","w")
for i in range(100):
    tmp=0
    for j in range(8):
        (R,out)=lfsr(R,mask)
        tmp=(tmp << 1)^out
    f.write(chr(tmp))
f.close()
```

The program is straightforward — still an LFSR, but the initial state is 32 bits. Of course, we could choose to brute-force, but here we opt not to.

Here we present two approaches.

The first approach: The 32nd bit of the program's output is determined by the first 31 bits of the output and the 1st bit of the initial seed. Therefore, we can determine the first bit of the initial seed, then the 2nd bit, and so on. The code is as follows:

```python
mask = 0b10100100000010000000100010010100
b = ''
N = 32
with open('key', 'rb') as f:
    b = f.read()
key = ''
for i in range(N / 8):
    t = ord(b[i])
    for j in xrange(7, -1, -1):
        key += str(t >> j & 1)
idx = 0
ans = ""
key = key[31] + key[:32]
while idx < 32:
    tmp = 0
    for i in range(32):
        if mask >> i & 1:
            tmp ^= int(key[31 - i])
    ans = str(tmp) + ans
    idx += 1
    key = key[31] + str(tmp) + key[1:31]
num = int(ans, 2)
print hex(num)
```

Running it:

```shell
➜  2018-CISCN-start-oldstreamgame git:(master) ✗ python exp1.py
0x926201d7
```

The second approach: We can consider the matrix transformation process. If 32 linear transformations are performed, the first 32 bits of the output stream can be obtained. In fact, we only need the first 32 bits to recover the initial state.


```python
mask = 0b10100100000010000000100010010100

N = 32
F = GF(2)

b = ''
with open('key', 'rb') as f:
    b = f.read()

R = [vector(F, N) for i in range(N)]
for i in range(N):
    R[i][N - 1] = mask >> (31 - i) & 1
for i in range(N - 1):
    R[i + 1][i] = 1
M = Matrix(F, R)
M = M ^ N

vec = vector(F, N)
row = 0
for i in range(N / 8):
    t = ord(b[i])
    for j in xrange(7, -1, -1):
        vec[row] = t >> j & 1
        row += 1
print rank(M)
num = int(''.join(map(str, list(M.solve_left(vec)))), 2)
print hex(num)
```


Running the script:

```shell
➜  2018-CISCN-start-oldstreamgame git:(master) ✗ sage exp.sage
32
0x926201d7
```

Therefore the flag is flag{926201d7}.

There is also another approach by TokyoWesterns; please refer to the corresponding files in the folder.

## Challenges



## References

- Cryptography Lecture Notes, by Li Chao and Qu Longjiang
