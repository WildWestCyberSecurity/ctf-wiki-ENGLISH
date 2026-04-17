# Comprehensive Static Analysis Challenges

## 2017 ISCC Crackone

Using jadx for decompilation, we can obtain the basic logic of the program as follows:

-   The program base64-encodes the user's input, then inserts `\r\n` at specified length positions, which doesn't really seem to serve any useful purpose.
-   Afterwards, the program passes the encoded content to the `check` function in the native library (so). The logic of this function is as follows:

```c
  env = a1;
  len = plen;
  str = pstr;
  v7 = malloc(plen);
  ((*env)->GetByteArrayRegion)(env, str, 0, len, v7);
  v8 = malloc(len + 1);
  memset(v8, 0, len + 1);
  memcpy(v8, v7, len);
  v9 = 0;
  for ( i = 0; ; ++i )
  {
    --v9;
    if ( i >= len / 2 )
      break;
    v11 = v8[i] - 5;
    v8[i] = v8[len + v9];
    v8[len + v9] = v11;
  }
  v8[len] = 0;
  v12 = strcmp(v8, "=0HWYl1SE5UQWFfN?I+PEo.UcshU");
  free(v8);
  free(v7);
  return v12 <= 0;
```

It is not difficult to see that the program simply takes the two halves of the base64-encoded string and performs appropriate operations on them. We can easily write the corresponding Python recovery code as follows:

```python
import base64


def solve():
    ans = '=0HWYl1SE5UQWFfN?I+PEo.UcshU'
    length = len(ans)
    flag = [0] * length

    beg = 0
    end = length
    while beg < length / 2:
        end -= 1
        flag[beg] = chr(ord(ans[end]) + 5)
        flag[end] = ans[beg]
        beg += 1
    flag = ''.join(flag)
    print base64.b64decode(flag)
if __name__ == "__main__":
    solve()
```

The result is as follows:

```shell
➜  2017ISCC python exp.py
flag{ISCCJAVANDKYXX}
```

## 2017 NJCTF easycrack

Through simple reverse engineering, we can discover that the basic logic of the program is as follows:

1.  It monitors the text field in the interface. If the text field content changes, it calls the native `parseText` function.
2.  The main functionality of `parseText` is as follows:
    1.  First, it calls the Java layer function `messageMe` to obtain a string `mestr`. The logic of this function is essentially:
        1.  Sequentially XOR each character of the substring after the last `.` in the package name with 51, and concatenate the results.
    2.  Then, using the length of `mestr` as the period, it XORs the two together. The core logic is `str[i + j] = mestr[j] ^ iinput[i + j];`
    3.  Next, it uses `I_am_the_key` as the key and employs RC4 encryption to encrypt this part, then compares the result with the final `compare` value. The basis for this inference is as follows:
        1.  The `init` function contains the key constant 256, and it essentially matches the RC4 key initialization process.
        2.  The `crypt` function is clearly an RC4 encryption function, with logic that obviously follows RC4 encryption.

The decryption script is as follows:

```python
from Crypto.Cipher import ARC4

def messageme():
    name = 'easycrack'
    init = 51
    ans = ""
    for c in name:
        init = ord(c) ^ init
        ans += chr(init)
    return ans

def decrypt(cipher,key):
    plain =""
    for i in range(0,len(cipher),len(key)):
        tmp = cipher[i:i+len(key)]
        plain +=''.join(chr(ord(tmp[i])^ord(key[i])) for i in range(len(tmp)))
    return plain

def main():
    rc4 = ARC4.new('I_am_the_key')
    cipher = 'C8E4EF0E4DCCA683088134F8635E970EEAD9E277F314869F7EF5198A2AA4'
    cipher = ''.join(chr(int(cipher[i:i+2], 16)) for i in range(0, len(cipher), 2))
    middleplain = rc4.decrypt(cipher)
    mestr = messageme()
    print decrypt(middleplain,mestr)


if __name__ == '__main__':
    main()
```

The result is as follows:

```shell
➜  2017NJCTF-easycrack python exp.py 
It_s_a_easyCrack_for_beginners
➜  2017NJCTF-easycrack 
```

## 2018 Qiangwang Cup picture lock

After a brief analysis, we find that this is an image encryption program: the Java layer passes the first filename under the `image/` directory to the native layer, along with the desired encrypted image filename and the MD5 of the corresponding APK's signature.

Next, we can analyze the native layer code. Since the program is clearly an encryption program, we can use IDA's findcrypto plugin for identification. The result reveals an S-box that essentially conforms to the AES encryption process, so we can basically confirm that the main body of the program is AES encryption. After careful analysis, we can determine that the basic flow of the native layer program is as follows:

1. Split the incoming signature's MD5 string in half to generate two sets of keys.
2. Read `md5sig[i%32]` bytes of content each time.
3. Decide which set of keys to use based on the size of data read:
   1. Odd size uses the second set of keys.
   2. Even size uses the first set of keys.
4. If the read size is less than 16, pad the remaining bytes with the padding size value (e.g., when the size is 12, pad with 4 bytes of 0x4).
5. At this point, the modified content is guaranteed to be at least 16 bytes. Perform AES encryption on the first 16 bytes. For the remaining bytes, XOR them sequentially with `md5sig[i%32]`.

Now that we know the encryption algorithm, it is easy to reverse. First, we can obtain the signature's MD5 as follows:

```shell
➜  picturelock keytool -list -printcert -jarfile picturelock.apk
签名者 #1:

签名:

所有者: CN=a, OU=b, O=c, L=d, ST=e, C=ff
发布者: CN=a, OU=b, O=c, L=d, ST=e, C=ff
序列号: 5f4e6be1
有效期为 Fri Sep 09 14:32:36 CST 2016 至 Tue Sep 03 14:32:36 CST 2041
证书指纹:
	 MD5:  F8:C4:90:56:E4:CC:F9:A1:1E:09:0E:AF:47:1F:41:8D
	 SHA1: 48:E7:04:5E:E6:0D:9D:8A:25:7C:52:75:E3:65:06:09:A5:CC:A1:3E
	 SHA256: BA:12:C1:3F:D6:0E:0D:EF:17:AE:3A:EE:4E:6A:81:67:82:D0:36:7F:F0:2E:37:CC:AD:5D:6E:86:87:0C:8E:38
签名算法名称: SHA256withRSA
主体公共密钥算法: 2048 位 RSA 密钥
版本: 3

扩展:

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 71 A3 2A FB D3 F4 A9 A9   2A 74 3F 29 8E 67 8A EA  q.*.....*t?).g..
0010: 3B DD 30 E3                                        ;.0.
]
]
➜  picturelock md5value=F8:C4:90:56:E4:CC:F9:A1:1E:09:0E:AF:47:1F:41:8D
➜  picturelock echo $md5value | sed 's/://g' | tr '[:upper:]' '[:lower:]'
f8c49056e4ccf9a11e090eaf471f418d
```

Then, we can directly use an existing AES library for decryption:

```python
#!/usr/bin/env python

import itertools

sig = 'f8c49056e4ccf9a11e090eaf471f418d'

from Crypto.Cipher import AES

def decode_sig(payload):
    ans = ""
    for i in range(len(payload)):
        ans +=chr(ord(payload[i]) ^ ord(sig[(16+i)%32]))
    return ans

def dec_aes():
	data = open('flag.jpg.lock', 'rb').read()
	jpg_data = ''
	f = open('flag.jpg', 'wb')
	idx = 0
	i = 0
	cipher1 = AES.new(sig[:0x10])
	cipher2 = AES.new(sig[0x10:])
	while idx < len(data):
		read_len = ord(sig[i % 32])
		payload = data[idx:idx+read_len]
		#print('[+] Read %d bytes' % read_len)
		print('[+] Totally %d / %d bytes, sig index : %d' % (idx, len(data), i))

		if read_len % 2 == 0:
			f.write(cipher1.decrypt(payload[:0x10]))
		else:
			f.write(cipher2.decrypt(payload[:0x10]))
		f.write(decode_sig(payload[16:]))
		f.flush()
		idx += read_len
		i += 1
	print('[+] Decoding done ...')
	f.close()

dec_aes()
```

Finally, we can obtain the decrypted image, which contains the flag.
