# Computer-Related Encodings

This section introduces some computer-related encodings.

## Alphabet Encoding

- A-Z/a-z corresponds to 1-26 or 0-25

## ASCII Encoding

![ascii](./figure/ascii.jpg)

### Characteristics

When we typically use ASCII encoding, we work with visible characters, mainly the following:

- 0-9, 48-57
- A-Z, 65-90
- a-z, 97-122

### Variations

#### Binary Encoding

Convert the ASCII code values to their binary representation.

- Contains only 0 and 1
- No more than 8 bits; 7 bits is also common since visible characters go up to 127.
- This is essentially another form of ASCII encoding.

#### Hexadecimal Encoding

Convert the ASCII code values to their hexadecimal representation.

- A-Z-->0x41~0x5A
- a-z-->0x61~0x7A

### Tools

- jpk, ascii to number, number to ascii
- http://www.ab126.com/goju/1711.html

### Examples

![ascii](./figure/ascii-example.png)



### 2018 DEFCON Quals ghettohackers: Throwback

The challenge description is as follows:

```
Anyo!e!howouldsacrificepo!icyforexecu!!onspeedthink!securityisacomm!ditytop!urintoasy!tem!
```

The first instinct is to fill in the content corresponding to the exclamation marks to get the flag, but after filling them in it doesn't work. So we can split the source string by `!`, where a string length of 1 corresponds to the letter a, length 2 corresponds to letter b, and so on:

```shell
ori = 'Anyo!e!howouldsacrificepo!icyforexecu!!onspeedthink!securityisacomm!ditytop!urintoasy!tem!'
sp = ori.split('!')
print repr(''.join(chr(97 + len(s) - 1) for s in sp))
```

This gives us the result, where we also need to assume that 0 characters represents a space, since this makes the original text readable:

```shell
dark logic
```

### Challenges

- Jarvis-basic - German Military Cipher

## Base Encoding


The "xx" in base xx represents how many characters are used for encoding. For example, base64 uses the following 64 characters for encoding. Since 2 to the power of 6 equals 64, every 6 bits form a unit corresponding to a printable character. 3 bytes have 24 bits, corresponding to 4 Base64 units, meaning 3 bytes need to be represented by 4 printable characters. It can be used as a transfer encoding for email. The printable characters in Base64 include letters A-Z, a-z, and digits 0-9, making 62 characters in total, plus two additional printable symbols that vary across different systems.


![base64](./figure/base64.png)

For more details, see [Base64 - Wikipedia](https://zh.wikipedia.org/wiki/Base64).


**Encoding "man"**

![base64 encoding MAN](./figure/base64_man.png)

If the number of bytes to be encoded is not divisible by 3, there will be 1 or 2 extra bytes remaining. These can be handled as follows: first pad the end with zero values to make it divisible by 3, then perform base64 encoding. One or two `=` signs are appended after the encoded base64 text to represent the number of padded bytes. That is, when there is one remaining byte, the last 6-bit base64 block has four zero-value bits, and two equals signs are appended; when there are two remaining bytes, the last 6-bit base64 block has two zero-value bits, and one equals sign is appended. Refer to the table below:

![base64 padding with 0](./figure/base64_0.png)

Since the padded zeros do not participate in the computation during decoding, information can be hidden at these positions.

Similar to base64, base32 uses 32 visible characters for encoding. Since 2 to the power of 5 equals 32, every 5 bits form one group. 5 bytes equal 40 bits, corresponding to 8 base32 groups, meaning 5 bytes are represented by 8 base32 characters. If there are fewer than 5 bytes, the first group that doesn't have a full 5 bits is padded with zeros to complete 5 bits, and all remaining groups are filled with "=" until a full 5 bytes are reached. Therefore, base32 can have at most 6 equals signs. For example:

![base32](./figure/base32.png)

### Characteristics

- base64 may end with `=` signs, but at most 2
- base32 may end with `=` signs, but at most 6
- The character set varies depending on the base type
- **You may need to add equals signs yourself**
- **= is also 3D**
- For more details, see [base rfc](https://tools.ietf.org/html/rfc4648)

### Tools

- http://www1.tc711.com/tool/BASE64.htm
- Python library functions
- [Script for reading steganographic information](https://github.com/cjcslhp/wheels/tree/master/b64stego)


### Examples

For the challenge description, see the data.txt file in the [misc category base64-stego directory](https://github.com/ctf-wiki/ctf-challenges/tree/master/misc/encode/computer/base64-stego) of `ctf-challenge`.

Use the script to read the steganographic information.

``` python
import base64

def deStego(stegoFile):
    b64table = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    with open(stegoFile,'r') as stegoText:
        message = ""
        for line in stegoText:
            try:
                text = line[line.index("=") - 1:-1]
                message += "".join([ bin( 0 if i == '=' else b64table.find(i))[2:].zfill(6) for i in text])[2 if text.count('=') ==2 else 4:6]  
            except:
                pass
    return "".join([chr(int(message[i:i+8],2)) for i in range(0,len(message),8)])

print(deStego("text.txt"))
```

Output:

```
     flag{BASE64_i5_amaz1ng}
```

<!--
下面是原编辑者的代码，代码的小毛病在于查找隐写字符用`last = line[-3]`写死了，这种写法默认每行尾有一个'\n',而最后一行并非如此，因此左后一个字符显示错误。

一大串 Base64 密文，试试补 0 位的数据。

```python
# coding=utf-8
import base64
import re

result = []
with open('text.txt', 'r') as f:
    for line in f.readlines():
        if len(re.findall(r'=', line)) == 2:
            last = line[-4]
            if last.isupper():
                num = ord(last) - ord('A')
            elif last.islower():
                num = ord(last) - ord('a') + 26
            elif last.isdigit():
                num = int(last) + 52
            elif last == '+':
                num = 62
            elif last == '/':
                num = 63
            elem = '{0:06b}'.format(num)
            result.append(elem[2:])

        elif len(re.findall(r'=', line)) == 1:
            last = line[-3]
            if last.isupper():
                num = ord(last) - ord('A')
            elif last.islower():
                num = ord(last) - ord('a') + 26
            elif last.isdigit():
                num = int(last) + 52
            elif last == '+':
                num = 62
            elif last == '/':
                num = 63
            elem = '{0:06b}'.format(num)
            result.append(elem[4:])

flag_b = ''.join(result)
split = re.findall(r'.{8}', flag_b)
for i in split:
    print chr(int(i, 2)),
```

感觉像是程序有点毛病，不过还是能看出来 flag。

```
flag{BASE64_i5_amaz1ng~
```
-->

### Challenges


## Huffman Coding

See [Huffman Coding](https://zh.wikipedia.org/wiki/%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81).

## XXencoding

XXencode encodes input text in units of three bytes. If the remaining data at the end is less than three bytes, the missing part is padded with zeros. These three bytes have 24 bits in total, which are divided into 4 groups of 6 bits each. The decimal value of each group falls between 0 and 63. Each value is replaced by the character at the corresponding position.

```text
           1         2         3         4         5         6
 0123456789012345678901234567890123456789012345678901234567890123
 |         |         |         |         |         |         |
 +-0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

For more information, see [Wikipedia](https://en.wikipedia.org/wiki/Xxencoding)

### Characteristics

- Contains only digits, uppercase and lowercase letters
- Plus sign and minus sign.

### Tools

- http://web.chacuo.net/charsetxxencode

### Challenges

## URL Encoding

See [URL Encoding - Wikipedia](https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81).

### Characteristics

- A large number of percent signs

### Tools

### Challenges

## Unicode Encoding

See [Unicode - Wikipedia](https://zh.wikipedia.org/wiki/Unicode).

Note that it has four representation forms.

### Examples

Source text: `The`

&#x [Hex]:  `&#x0054;&#x0068;&#x0065;`

&# [Decimal]:  `&#00084;&#00104;&#00101;`

\U [Hex]:  `\U0054\U0068\U0065`

\U+ [Hex]:  `\U+0054\U+0068\U+0065`

### Tools

### Challenges

## HTML Entity Encoding
