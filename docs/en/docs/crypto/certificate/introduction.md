# Certificate Formats


## PEM


PEM starts with `-----BEGIN` and ends with `-----END`, with ASN.1 formatted data in between. ASN.1 is base64-encoded binary data. [Wikipedia](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) has examples of complete PEM files.


You can use Python 3 and the PyCryptodome library to interact with PEM files and extract relevant data. For example, if we want to extract the modulus `n`:


```py
#!/usr/bin/env python3
from Crypto.PublicKey import RSA

with open("certificate.pem","r") as f:
	key = RSA.import_key(f.read())
	print(key.n)
```


## DER


DER is a binary encoding of ASN.1 types. Certificates with the `.cer` or `.crt` extension usually contain data in DER format, but Windows may also accept data in PEM format.


We can use `openssl` to convert a DER file to a PEM file:


```bash
openssl x509 -inform DER -in certificate.der > certificate.pem
```


Now the problem is reduced to reading a PEM file, so we can reuse the Python code from the previous section.


## Other Format Conversions


```bash
openssl x509 -outform der -in certificate.pem -out certificate.der
openssl x509 -inform der -in certificate.cer -out certificate.pem
```


## References

1. [Attacking RSA for fun and CTF points – part 1](https://bitsdeep.com/posts/attacking-rsa-for-fun-and-ctf-points-part-1/)
2. [What are the differences between .pem, .cer and .der?](https://stackoverflow.com/questions/22743415/what-are-the-differences-between-pem-cer-and-der)
