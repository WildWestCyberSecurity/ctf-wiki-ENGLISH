# DNS

## Introduction


`DNS` typically uses the `UDP` protocol. The message format is as follows:

```sh
+-------------------------------+
| Header                        |
+-------------------------------+
| Question (query section       |
|   sent to the server)         |
+-------------------------------+
| Answer (resource records      |
|   returned by the server)     |
+-------------------------------+
| Authority (authoritative      |
|   resource records)           |
+-------------------------------+
| Additional (additional        |
|   resource records)           |
+-------------------------------+
```

A query packet only contains the header and question sections. After `DNS` receives a query packet, it appends the answer information, authority section, and additional resource records based on the queried information, modifies the relevant flags in the header, and returns it to the client.

Each `question` section:

```
   0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                                               |
 /                     QNAME                     /
 /                                               /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     QTYPE                     |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     QCLASS                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

- `QNAME`: The domain name being queried. It has a variable length and is encoded as follows: the domain name is split into multiple parts by the `.` delimiter, each part is preceded by a byte indicating the length of that part, and a `0` byte is appended at the end to indicate termination.
- `QTYPE`: Occupies `16` bits and represents the query type. There are `16` types in total. Common values include: `1` (`A` record, requesting the host `IP` address), `2` (`NS`, requesting the authoritative `DNS` server), `5` (`CNAME` alias query).



## Example

> Challenge: `BSides San Francisco CTF 2017`: `dnscap.pcap`

We open it with `wireshark` and find that all traffic is `DNS` protocol, with query names consisting of large strings `([\w\.]+)\.skullseclabs\.org`.

We extract the data using `tshark -r dnscap.pcap -T fields -e dns.qry.name > hex`, then decode it with `python`:

```python
import re


find = ""

with open('hex','rb') as f:
    for i in f:
        text = re.findall(r'([\w\.]+)\.skull',i)
        if text:
            find += text[0].replace('.','')
print find
```

We discover several key pieces of information:

```
Welcome to dnscap! The flag is below, have fun!!
Welcome to dnscap! The flag is below, have fun!!
!command (sirvimes)
...
IHDR
gAMA
bKGD
        pHYs
IHDR
gAMA
bKGD
        pHYs
tIME
IDATx
...
2017-02-01T21:04:00-08:00
IEND
console (sirvimes)
console (sirvimes)
Good luck! That was dnscat2 traffic on a flaky connection with lots of re-transmits. Seriously,
Good luck! That was dnscat2 traffic on a flaky connection with lots of re-transmits. Seriously, d[
good luck. :)+
```

The `flag` is indeed contained within, but there is a lot of duplicate information. One reason is the `question` section. In the `dns` protocol, both query and response use it. After filtering with ` -Y "ip.src == 192.168.43.91"`, there are still quite a few duplicate parts.

```
%2A}
%2A}
%2A}q
%2A}x
%2A}
IHDR
gAMA
bKGD
        pHYs
tIME
IDATx
HBBH
CxRH!
C1%t
ceyF
i4ZI32
rP@1
ceyF
i4ZI32
rP@1
ceyF
i4ZI32
rP@1
ceyF
i4ZI32
rP@1
```

Based on the discovered `dnscat`, we find https://github.com/iagox86/dnscat2/blob/master/doc/protocol.md which describes information about the `dnscat` protocol. This is a variant protocol that transmits data through `DNS`. The challenge file likely does not use encryption, so we can look directly at the data block information here:

```
MESSAGE_TYPE_MSG: [0x01]
(uint16_t) packet_id
(uint8_t) message_type [0x01]
(uint16_t) session_id
(uint16_t) seq
(uint16_t) ack
(byte[]) data
```

In `qry.name`, we remove the other fields and keep only the `data` block, thereby merging the data. Then we search for `89504e.....6082` in the hexadecimal data to extract the `png` and obtain the `flag`.

```python
import re


find = []

with open('hex','rb') as f:
    for i in f:
        text = re.findall(r'([\w\.]+)\.skull',i)
        if text:
            tmp =  text[0].replace('.','')
            find.append(tmp[18:])
last = []

for i in find:
    if i not in last:
        last.append(i)


print  ''.join(last)
```

*flag*

![dnscat_flag](./figure/dnscat_flag.png)



## Related Challenges

- [IceCTF-2016:Search](https://mrpnkt.github.io/2016/icectf-2016-search/)
- [EIS-2017:DNS 101](https://github.com/susers/Writeups/blob/master/2017/EIS/Misc/DNS%20101/Write-up.md)

## References

- https://github.com/lisijie/homepage/blob/master/posts/tech/dns%E5%8D%8F%E8%AE%AE%E8%A7%A3%E6%9E%90.md
- https://xpnsec.tumblr.com/post/157479786806/bsidessf-ctf-dnscap-walkthrough
