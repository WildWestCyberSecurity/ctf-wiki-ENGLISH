# HTTP

`HTTP` (`Hyper Text Transfer Protocol`) is an application layer protocol for distributed, collaborative, and hypermedia information systems. `HTTP` is the foundation of data communication for the World Wide Web.

## Example

> Challenge: Jiangsu Province Linghang Cup 2017: hack

Overall observation reveals:

- Mainly `HTTP` traffic
- Primarily from `192.168.173.134`
- No attachments present

![linghang_hack](./figure/linghang_hack.png)

From this image, we can basically determine that this is a traffic capture generated during `SQL injection - blind injection`.

At this point, we can essentially determine the direction of the flag. After extracting all URLs, we can use `python` to obtain the flag.

- Extract URLs: `tshark -r hack.pcap -T fields  -e http.request.full_uri|tr -s '\n'|grep flag > log`
- Obtain the blind injection results:

```python
import re

with open('log') as f:
    tmp = f.read()
    flag = ''
    data = re.findall(r'=(\d*)%23',tmp)
    data = [int(i) for i in data]
    for i,num in enumerate(data):
        try:
            if num > data[i+1]:
                flag += chr(num)
        except Exception:
            pass
    print flag
```
