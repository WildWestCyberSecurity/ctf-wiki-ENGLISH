# HTTPS

`HTTPs = HTTP + SSL / TLS`. The information transmitted between the server and client is encrypted through TLS, so the transmitted data is all encrypted.

- [Wireshark Analysis of HTTPS](http://www.freebuf.com/articles/system/37900.html)

## Example

> Challenge: hack-dat-kiwi-ctf-2015: ssl-sniff-2

Opening the traffic capture reveals `SSL` encrypted data. Importing the provided `server.key.insecure` from the challenge allows decryption:

```xml
GET /key.html HTTP/1.1
Host: localhost

HTTP/1.1 200 OK
Date: Fri, 20 Nov 2015 14:16:24 GMT
Server: Apache/2.4.7 (Ubuntu)
Last-Modified: Fri, 20 Nov 2015 14:15:54 GMT
ETag: "1c-524f98378d4e1"
Accept-Ranges: bytes
Content-Length: 28
Content-Type: text/html

The key is 39u7v25n1jxkl123
```
