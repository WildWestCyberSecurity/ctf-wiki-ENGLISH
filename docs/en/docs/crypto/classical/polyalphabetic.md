# Polyalphabetic Substitution Cipher

For polyalphabetic substitution ciphers, the encrypted letters almost no longer maintain their original frequency, so we can generally only break them by finding weaknesses in the algorithm implementation.

## Playfair

### Principle

The Playfair cipher (Playfair cipher or Playfair square) is a substitution cipher invented by the Englishman Charles Wheatstone in 1854. The basic algorithm is as follows:

1.  Select a string of English letters, remove duplicate letters, and fill the remaining letters one by one into a 5 × 5 matrix. The remaining spaces are filled with unused English letters in a-z order. Note: either remove q, or treat i and j as the same letter.
2.  Divide the plaintext to be encrypted into pairs. If both letters in a pair are the same, insert X (or Q) after the first letter and re-group. If there is a remaining single letter, also add X.
3.  For each pair, find the positions of the two letters in the matrix.
    - If the two letters are in different rows and different columns, find the other two letters in the matrix (prioritizing the row of the first letter) such that the four letters form the corners of a rectangle.
    - If the two letters are in the same row, take the letter to the right of each (wrapping to the leftmost if the letter is at the rightmost position).
    - If the two letters are in the same column, take the letter below each (wrapping to the topmost if the letter is at the bottommost position).

The two newly found letters are the encrypted result of the original two letters.

Using "playfair example" as the key, we get:

```
P L A Y F
I R E X M
B C D G H
K N O Q S
T U V W Z
```

The message to encrypt is: Hide the gold in the tree stump

```
HI DE TH EG OL DI NT HE TR EX ES TU MP
```

The result is:

```
BM OD ZB XD NA BE KU DM UI XM MO UV IF
```

### Tools

- CAP4

## Polybius

### Principle

The Polybius cipher, also known as the checkerboard cipher, generally encrypts a given plaintext into pairs of numbers. Its commonly used cipher table is:

|      | 1   | 2   | 3   | 4   | 5    |
| :--- | --- | --- | --- | --- | :--- |
| 1    | A   | B   | C   | D   | E    |
| 2    | F   | G   | H   | I/J | K    |
| 3    | L   | M   | N   | O   | P    |
| 4    | Q   | R   | S   | T   | U    |
| 5    | V   | W   | X   | Y   | Z    |

For example, the plaintext HELLO encrypts to 23 15 31 31 34.

Another cipher table:

|     | A   | D   | F   | G   | X   |
| --- | --- | --- | --- | --- | --- |
| A   | b   | t   | a   | l   | p   |
| D   | d   | h   | o   | z   | k   |
| F   | q   | f   | v   | s   | n   |
| G   | g   | j   | c   | u   | x   |
| X   | m   | r   | e   | w   | y   |

Note that the letter order here has been scrambled.

Origin of A D F G X:

> In 1918, near the end of World War I, the French intercepted a German telegram in which all words were composed of only five letters: A, D, F, G, and X — hence the name ADFGX cipher. The ADFGX cipher was invented in March 1918 by German Colonel Fritz Nebel and is a dual encryption scheme combining the Polybius cipher with a transposition cipher.

For example, HELLO encrypted using this table becomes DD XF AG AG DF.

### Tools

- CrypTool

### Example

Here we use the Anheng Cup September Crypto challenge "Go" as an example. The challenge is:

> Ciphertext: ilnllliiikkninlekile

> The compressed file contains a line of hexadecimal: 546865206c656e677468206f66207468697320706c61696e746578743a203130

> Please decrypt the ciphertext.

First, hex-decode the hexadecimal to get the string: "The length of this plaintext: 10"

The ciphertext length is 20, while the plaintext length is 10. The ciphertext only contains the five characters "l", "i", "n", "k", "e", which suggests a checkerboard cipher.

First, try arranging the five characters in alphabetical order:

|      | e   | i   | k   | l   | n    |
| :--- | --- | --- | --- | --- | :--- |
| e    | A   | B   | C   | D   | E    |
| i    | F   | G   | H   | I/J | K    |
| k    | L   | M   | N   | O   | P    |
| l    | Q   | R   | S   | T   | U    |
| n    | V   | W   | X   | Y   | Z    |

Decrypting the ciphertext gives: iytghpkqmq.

This is probably not the flag we're looking for.

It seems the five characters are not arranged this way. There are 5! possible arrangements, so we write a script to brute-force it:

```python
import itertools

key = []
cipher = "ilnllliiikkninlekile"

for i in itertools.permutations('ilnke', 5):
    key.append(''.join(i))

for now_key in key:
    solve_c = ""
    res = ""
    for now_c in cipher:
        solve_c += str(now_key.index(now_c))
    for i in range(0,len(solve_c),2):
        now_ascii = int(solve_c[i])*5+int(solve_c[i+1])+97
        if now_ascii>ord('i'):
            now_ascii+=1
        res += chr(now_ascii)
    if "flag" in res:
        print now_key,res
```
The script essentially implements the checkerboard cipher algorithm, except the order of the five characters is unknown.

Running it produces the following two results:

> linke flagishere

> linek flagkxhdwd

Obviously, the first one is the answer we're looking for.

Here is the correct cipher table:

|      | l   | i   | n   | k   | e    |
| :--- | --- | --- | --- | --- | :--- |
| l    | A   | B   | C   | D   | E    |
| i    | F   | G   | H   | I/J | K    |
| n    | L   | M   | N   | O   | P    |
| k    | Q   | R   | S   | T   | U    |
| e    | V   | W   | X   | Y   | Z    |

## Vigenère Cipher

### Principle

The Vigenère cipher is an encryption algorithm that uses a series of Caesar ciphers to form cipher alphabets. It is a simple form of polyalphabetic cipher.

![Vigenère table](./figure/vigenere1.jpg)

Here is an example:

```
Plaintext: come greatwall
Key:       crypto
```

First, pad the key so that its length matches the plaintext length.

| Plaintext | c   | o   | m   | e   | g   | r   | e   | a   | t   | w   | a   | l   | l   |
| --------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Key       | c   | r   | y   | p   | t   | o   | c   | r   | y   | p   | t   | o   | c   |

Then, look up the ciphertext in the table:

![Vigenère encryption](./figure/vigenere2.jpg)

```
Plaintext:  come greatwall
Key:        crypto
Ciphertext: efkt zfgrrltzn
```
### Cracking

Breaking all polyalphabetic ciphers, including the Vigenère cipher, is based on letter frequency, but direct frequency analysis is not applicable because in the Vigenère cipher, a single letter can be encrypted into different ciphertext characters, making simple frequency analysis useless here.

**The key to breaking the Vigenère cipher is that the key repeats cyclically.** If we know the length of the key, the ciphertext can be viewed as interleaved Caesar ciphers, each of which can be broken individually. To determine the key length, we can use the Kasiski test and the Friedman test.

The Kasiski test is based on the observation that common words like "the" may be encrypted by the same key letters, causing them to repeat in the ciphertext. For example, different occurrences of CRYPTO in the plaintext might be encrypted with key ABCDEF into different ciphertext:

```
Key:        ABCDEF AB CDEFA BCD EFABCDEFABCD
Plaintext:  CRYPTO IS SHORT FOR CRYPTOGRAPHY
Ciphertext: CSASXT IT UKSWT GQU GWYQVRKWAQJB
```

In this case, the repeated elements in the plaintext do not repeat in the ciphertext. However, if the key is the same (using key ABCD):

```
Key:        ABCDAB CD ABCDA BCD ABCDABCDABCD
Plaintext:  CRYPTO IS SHORT FOR CRYPTOGRAPHY
Ciphertext: CSASTP KV SIQUT GQU CSASTPIUAQJB
```

In this case, the Kasiski test becomes effective. This method is more effective for longer passages, as there are usually more repeated fragments in the ciphertext. For example, the key length can be deduced from the following ciphertext:

```
Ciphertext: DYDUXRMHTVDVNQDQNWDYDUXRMHARTJGWNQD
```

The two occurrences of DYDUXRMH are separated by 18 letters. Therefore, the key length can be assumed to be a divisor of 18, i.e., 18, 9, 6, 3, or 2. The two occurrences of NQD are separated by 20 letters, meaning the key length should be 20, 10, 5, 4, or 2. Taking the intersection, we can basically determine the key length is 2. The next step is to perform further operations.

For more detailed cracking principles, we will not go into too much detail here. You can refer to http://www.practicalcryptography.com/cryptanalysis/stochastic-searching/cryptanalysis-vigenere-cipher/.

### Tools

-   Known key
    - Python's pycipher library
    - [Online Vigenère cipher decryption](http://planetcalc.com/2468/)
    - CAP4
-   Unknown key
    - [Vigenère Cipher Codebreaker](http://www.mygeocachingprofile.com/codebreaker.vigenerecipher.aspx)
    - [Vigenere Solver](https://www.guballa.de/vigenere-solver), not fully comprehensive.

## Nihilist

### Principle

The Nihilist cipher, also known as the keyword cipher, works as: plaintext + keyword = ciphertext. Using the keyword "helloworld" as an example:

First, construct a checkerboard matrix using the key (similar to the Polybius cipher):
- Create a new 5 × 5 matrix
- Fill in the characters without repetition in order
- Fill the remaining spaces in alphabetical order
- Letters i and j are equivalent

|     | 1   | 2   | 3     | 4   | 5   |
| --- | --- | --- | ----- | --- | --- |
| 1   | h   | e   | l     | o   | w   |
| 2   | r   | d   | a     | b   | c   |
| 3   | f   | g   | i / j | k   | m   |
| 4   | n   | p   | q     | s   | t   |
| 5   | u   | v   | x     | y   | z   |

For the encryption process, encrypt according to matrix M:

```
a -> M[2,3] -> 23
t -> M[4,5] -> 45
```
For the decryption process, decrypt according to matrix M:

```
23 -> M[2,3] -> a
45 -> M[4,5] -> t
```
The characteristics of the ciphertext are as follows:

- Pure numbers
- Contains only digits 1 to 5
- Ciphertext length is even.

## Hill

### Principle

The Hill cipher uses each letter's position in the alphabet as its corresponding number (i.e., A=0, B=1, C=2, etc.), then converts the plaintext into n-dimensional vectors, multiplies them by an n × n matrix, and takes the result modulo 26. Note that the matrix used for encryption (i.e., the key) must be invertible in $\mathbb{Z}_{26}^{n}$, otherwise decryption is impossible. A matrix is invertible only if its determinant is coprime with 26. Here is an example:

```
Plaintext: ACT
```

Convert the plaintext into a matrix:

$$
\begin{bmatrix}
0\\
2\\
19
\end{bmatrix}
$$

Assume the key is:

$$
\begin{bmatrix}
6 & 24 & 1\\
13 & 16 & 10\\
20 & 17 & 15
\end{bmatrix}
$$

The encryption process is:

$$
\begin{bmatrix}
6 & 24 & 1\\
13 & 16 & 10\\
20 & 17 & 15
\end{bmatrix}
\begin{bmatrix}
0\\
2\\
19
\end{bmatrix}
\equiv
\begin{bmatrix}
67\\
222\\
319
\end{bmatrix}
\equiv
\begin{bmatrix}
15\\
14\\
7
\end{bmatrix}
\bmod 26
$$

The ciphertext is:

```
Ciphertext: POH
```

### Tools

- http://www.practicalcryptography.com/ciphers/hill-cipher/
- CAP4
- Cryptool

### Example

Here we use ISCC 2015 base decrypt 150 as an example. The challenge is:

> Ciphertext: 22,09,00,12,03,01,10,03,04,08,01,17 (wjamdbkdeibr)
>
> The matrix used is 1 2 3 4 5 6 7 8 10
>
> Please decrypt the ciphertext.

First, the matrix is 3 × 3, meaning 3 characters are encrypted at a time. We directly use Cryptool. Note that this matrix is arranged by columns, i.e.:

```
1 4 7
2 5 8
3 6 10
```

The final result is `overthehillx`.

## AutokeyCipher

### Principle

The Autokey Cipher is also a polyalphabetic substitution cipher, similar to the Vigenère cipher, but uses a different method to generate the key. It is generally more secure than the Vigenère cipher. There are two main types of Autokey ciphers: keyword autokey cipher and plaintext autokey cipher. Below we use the keyword autokey cipher as an example:

```
Plaintext: THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG
Keyword:   CULTURE
```

Auto-generated key:

```
CULTURE THE QUICK BROWN FOX JUMPS OVER THE
```

The subsequent encryption process is similar to the Vigenère cipher. From the corresponding table we get:

Ciphertext:

```
VBP JOZGD IVEQV HYY AIICX CSNL FWW ZVDP WVK
```

### Tools

-   Known keyword
    - Python's pycipher library
-   Unknown keyword
    - http://www.practicalcryptography.com/cryptanalysis/stochastic-searching/cryptanalysis-autokey-cipher/
    - **break_autokey.py in the tools folder — to be completed.**
