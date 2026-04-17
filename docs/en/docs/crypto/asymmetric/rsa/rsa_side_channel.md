# RSA Side-Channel Attack

Power analysis attack (side-channel attack) is a cryptographic attack method that can extract secret information from cryptographic devices. Unlike other attack methods, this attack exploits the power consumption characteristics of cryptographic devices rather than the mathematical properties of cryptographic algorithms. Power analysis attacks are non-invasive attacks, and attackers can conveniently purchase the equipment needed to carry out the attack, which poses a serious threat to the security of cryptographic devices such as smart cards.

Power analysis attacks are a very important part of the security field; we will only briefly discuss them here.

Power analysis attacks are divided into:
- Simple Power Analysis (SPA), which involves direct visual analysis of power traces.
- Differential Power Analysis (DPA), which is based on the correlation coefficients between power traces.

## Attack Conditions

The attacker can obtain side-channel information related to encryption and decryption, such as power consumption, computation time, electromagnetic radiation, etc.

## Example
Here we use HITB 2017's Hack in the card I as an example.

The challenge provides a public key file `publickey.pem`, ciphertext, a circuit diagram for measuring smart card power, and the power consumption changes of the smart card during the **decryption** process (provided through an online website [trace](http://47.74.147.53:20015/index.html)).

![Circuit diagram](./figure/circuitdiagram.png)

Ciphertext:
```
014b05e1a09668c83e13fda8be28d148568a2342aed833e0ad646bd45461da2decf9d538c2d3ab245b272873beb112586bb7b17dc4b30f0c5408d8b03cfbc8388b2bd579fb419a1cac38798da1c3da75dc9a74a90d98c8f986fd8ab8b2dc539768beb339cadc13383c62b5223a50e050cb9c6b759072962c2b2cf21b4421ca73394d9e12cfbc958fc5f6b596da368923121e55a3c6a7b12fdca127ecc0e8470463f6e04f27cd4bb3de30555b6c701f524c8c032fa51d719901e7c75cc72764ac00976ac6427a1f483779f61cee455ed319ee9071abefae4473e7c637760b4b3131f25e5eb9950dd9d37666e129640c82a4b01b8bdc1a78b007f8ec71e7bad48046
```

### Analysis
Since the website only provides one power trace, we can determine that this is a Simple Power Analysis (SPA) attack. We can directly obtain the private key d used in the RSA decryption process by observing the high and low levels in the power trace.
The theoretical basis for SPA attacks on RSA comes from the modular exponentiation algorithm used in RSA.


The fast modular exponentiation algorithm is as follows:

1. When b is even, $a^b \bmod c = ({a^2}^{b/2}) \bmod c$.
2. When b is odd, $a^b \bmod c = ({a^2}^{b/2} \times a) \bmod c$.

The corresponding C code implementation is:
```c
int PowerMod(int a, int b, int c)
{
    int ans = 1;
    a = a % c;
    while(b>0) {
        if(b % 2 == 1) // 当b为奇数时会多执行下面的指令
	        ans = (ans * a) % c;
        b = b/2;
        a = (a * a) % c;
    }
    return ans;
}
```

Since the fast exponentiation process checks each bit of the exponent and takes different operations accordingly, the value of d can be recovered from the power trace (as can be seen from above, the directly obtained value is the **reverse** of the binary representation of d).

**Note**:

> Sometimes modular multiplication may also proceed from high bits to low bits. Here it proceeds from low bits to high bits.

![](./figure/trace.png)

The script for recovering d is as follows:

```python
f = open('./data.txt')
data = f.read().split(",")
print('point number:', len(data))

start_point = 225   # 开始分析的点
mid = 50            # 采样点间隔
fence = 228         # 高低电平分界线

bin_array = []

for point_index in range(start_point, len(data), mid):
    if float(data[point_index]) > fence:
        bin_array.append(1)
    else:
        bin_array.append(0)

bin_array2 = []
flag1 = 0
flag2 = 0
for x in bin_array:
    if x:
        if flag1:
            flag2 = 1
        else:
            flag1 = 1
    else:
        if flag2:
            bin_array2.append(1)
        else:
            bin_array2.append(0)
        flag1 = 0
        flag2 = 0

# d_bin = bin_array2[::-1]
d_bin = bin_array2
d = "".join(str(x) for x in d_bin)[::-1]
print(d)
d_int = int(d,2)
print(d_int)
```
## References
1. Mangard, S., Oswald, E., Popp, T., 冯登国, 周永彬, & 刘继业. (2010). 能量分析攻击.
