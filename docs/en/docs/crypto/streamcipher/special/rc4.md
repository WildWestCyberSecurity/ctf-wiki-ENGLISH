# RC4

## Basic Introduction

RC4 was designed by Ron Rivest and originally belonged to RSA Security as a proprietary cryptographic product. It is a byte-oriented stream cipher with a variable key length that is very simple yet very effective. The RC4 algorithm is widely used in SSL/TLS protocols and WEP/WPA protocols.

## Basic Process

RC4 mainly consists of three processes:

- Initialize S and T arrays.
- Initialize permutation S.
- Generate stream key.

### Initialize S and T Arrays 

The code for initializing S and T is as follows:

```c
for i = 0 to 255 do
	S[i] = i
	T[i] = K[i mod keylen]
```

 ![image-20180714192918699](figure/rc4_s_t.png)

### Initialize Permutation S

```c
j = 0
for i = 0 to 255 do 
	j = (j + S[i] + T[i]) (mod 256) 
	swap (S[i], S[j])
```

![image-20180714193448454](figure/rc4_s.png)

### Generate Stream Key

```c
i = j = 0 
for each message byte b
	i = (i + 1) (mod 256)
	j = (j + S[i]) (mod 256)
	swap(S[i], S[j])
	t = (S[i] + S[j]) (mod 256) 
	print S[t]
```

![image-20180714193537976](figure/rc4_key.png)

The first two parts are generally called KSA (Key Scheduling Algorithm), and the last part is PRGA (Pseudo-Random Generation Algorithm).

## Attack Methods

To be supplemented.
