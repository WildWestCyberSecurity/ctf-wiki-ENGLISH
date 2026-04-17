# Comprehensive Challenges

## 2017 34c3 Software_update

It can be seen that the general idea of the program is to upload a zip archive, and then verify the signatures of the files in the signed_data directory. The final verification method is roughly to compute the sha256 hash of each file, then **XOR** them together as the input passed to RSA for signature verification. If the verification passes, the corresponding pre-copy.py and post-copy.py files will be executed.

A natural idea is to modify the pre-copy.py or post-copy.py file so that it can read the flag, and then bypass the signature again. There are mainly two approaches:

1. Obtain the corresponding private key from the given public key file, then forge a signature after modifying the file. After roughly examining the public key file, it appears nearly unbreakable, so this approach can basically be abandoned.
2. After modifying the corresponding file, exploit **the XOR property to keep the hash value the same as the original**, thereby bypassing signature verification. That is, include multiple files in the signed_data directory such that the XOR of their hash values cancels out the hash value difference caused by modifying pre-copy.py or post-copy.py.

Here, we choose the second method. Specifically, we choose to modify the pre-copy.py file. The approach is as follows:

1. Compute the original hash value of pre-copy.py.
2. Modify the pre-copy.py file so that it can read the flag. At the same time, compute the new hash value. XOR the two to obtain the XOR difference delta.
3. Find a set of files whose hash values XOR together to exactly equal delta.

The key step is step 3. This can actually be viewed as a linear combination problem, i.e., finding several 256-dimensional 0/1 vectors whose XOR value equals delta. And
$$
(F=\{0,1\},F^{256},\oplus ,\cdot)
$$
is a 256-dimensional vector space. If we can find a basis for this vector space, then we can find the required vectors for any specified value in this space.

We can use sage to assist us, as follows:

```python
# generage the base of <{0,1},F^256,xor,*>
def gen_gf2_256_base():
    v = VectorSpace(GF(2), 256)
    tmphash = compute_file_hash("0.py", "")
    tmphash_bin = hash2bin(tmphash)
    base = [tmphash_bin]
    filelist = ['0.py']
    print base
    s = v.subspace(base)
    dim = s.dimension()
    cnt = 1
    while dim != 256:
        tmpfile = str(cnt) + ".py"
        tmphash = compute_file_hash(tmpfile, "")
        tmphash_bin = hash2bin(tmphash)
        old_dim = dim
        s = v.subspace(base + [tmphash_bin])
        dim = s.dimension()
        if dim > old_dim:
            base += [tmphash_bin]
            filelist.append(tmpfile)
            print("dimension " + str(s.dimension()))
        cnt += 1
        print(cnt)
    m = matrix(GF(2), 256, 256, base)
    m = m.transpose()
    return m, filelist
```

For a more detailed solution, please refer to `exp.py`.

Here, modifying pre-copy to additionally output `!!!!come here!!!!`, as follows:

```shell
➜  software_update git:(master) python3 installer.py now.zip
Preparing to copy data...
!!!!come here!!!!
Software update installed successfully.
```

References

- https://sectt.github.io/writeups/34C3CTF/crypto_182_software_update/Readme
- https://github.com/OOTS/34c3ctf/blob/master/software_update/solution/exploit.py

## 2019 36c3 SaV-ls-l-aaS

This challenge is categorized as Crypto&Web. Let's walk through the process:

Port 60601 has a Web service running, and the challenge description provides the connection method:

```bash
url='http://78.47.240.226:60601' && ip=$(curl -s "$url/ip") && sig=$(curl -s -d "cmd=ls -l&ip=$ip" "$url/sign") && curl --data-urlencode "signature=$sig" "$url/exec"
```

It can be seen that first we access `/ip` to get the IP, then POST the IP and the command we want to execute to `/sign` to get a signature, and finally POST the signature to `/exec` to execute the command. Running this command shows the result of `ls -l`, and we can see there is a flag.txt.

Looking at the source code, the Web service is written in Go:

```go
package main

import (
	"bytes"
	"crypto/sha1"
	"encoding/json"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net"
	"net/http"
	"strings"
	"time"
)

func main() {
	m := http.NewServeMux()

	m.HandleFunc("/ip", func(w http.ResponseWriter, r *http.Request) {
		ip, _, err := net.SplitHostPort(r.RemoteAddr)
		if err != nil {
			return
		}
		fmt.Fprint(w, ip)
	})

	m.HandleFunc("/sign", func(w http.ResponseWriter, r *http.Request) {
		ip, _, err := net.SplitHostPort(r.RemoteAddr)
		if err != nil {
			return
		}
		remoteAddr := net.ParseIP(ip)
		if remoteAddr == nil {
			return
		}

		ip = r.PostFormValue("ip")
		signIP := net.ParseIP(ip)
		if signIP == nil || !signIP.Equal(remoteAddr) {
			fmt.Fprintln(w, "lol, not ip :>")
			return
		}

		cmd := r.PostFormValue("cmd")
		if cmd != "ls -l" {
			fmt.Fprintln(w, "lol, nope :>")
			return
		}

		msg := ip + "|" + cmd
		digest := sha1.Sum([]byte(msg))

		b := new(bytes.Buffer)
		err = json.NewEncoder(b).Encode(string(digest[:]))
		if err != nil {
			return
		}

		resp, err := http.Post("http://127.0.0.1/index.php?action=sign", "application/json; charset=utf-8", b)
		if err != nil || resp.StatusCode != 200 {
			fmt.Fprintln(w, "oops, hsm is down")
			return
		}

		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			fmt.Fprintln(w, "oops, hsm is bodyless?")
			return
		}

		var signature string
		err = json.Unmarshal(body, &signature)
		if err != nil {
			fmt.Fprintln(w, "oops, hsm is jsonless?")
			return
		}

		fmt.Fprint(w, signature+msg)
	})

	m.HandleFunc("/exec", func(w http.ResponseWriter, r *http.Request) {
		ip, _, err := net.SplitHostPort(r.RemoteAddr)
		if err != nil {
			return
		}
		remoteAddr := net.ParseIP(ip)
		if remoteAddr == nil {
			return
		}

		signature := r.PostFormValue("signature")
		digest := sha1.Sum([]byte(signature[172:]))

		b := new(bytes.Buffer)
		err = json.NewEncoder(b).Encode(signature[:172] + string(digest[:]))
		if err != nil {
			fmt.Fprintln(w, "oops, json encode")
			return
		}

		resp, err := http.Post("http://127.0.0.1/index.php?action=verify", "application/json; charset=utf-8", b)
		if err != nil || resp.StatusCode != 200 {
			fmt.Fprintln(w, "oops, hsm is down?")
			return
		}

		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			fmt.Fprintln(w, "oops, hsm is bodyless?")
			return
		}

		var valid bool
		err = json.Unmarshal(body, &valid)
		if err != nil {
			fmt.Fprintln(w, "oops, json unmarshal")
			return
		}

		if valid {
			t := strings.Split(signature[172:], "|")
			if len(t) != 2 {
				fmt.Fprintln(w, "oops, split")
			}

			signIP := net.ParseIP(t[0])
			if signIP == nil || !signIP.Equal(remoteAddr) {
				fmt.Fprintln(w, "lol, not ip :>")
				return
			}

			conn, err := net.DialTimeout("tcp", "127.0.0.1:1024", 1*time.Second)
			if err != nil {
				fmt.Fprintln(w, "oops, dial")
				return
			}
			fmt.Fprintf(conn, t[1]+"\n")
			conn.(*net.TCPConn).CloseWrite()
			io.Copy(w, conn)
		}
	})

	s := &http.Server{
		Addr:           ":60601",
		Handler:        m,
		ReadTimeout:    5 * time.Second,
		WriteTimeout:   5 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}
	log.Fatal(s.ListenAndServe())
}

```

The code is easy to read. It restricts cmd to only `ls -l`; anything else won't get a signature. It looks like we need to forge a signature for other commands to read the flag. Note that the signing and verification processes are passed to a locally running PHP service. Let's look at this part of the source code:

```php
<?php
define('ALGO', 'md5WithRSAEncryption');
$d = json_decode(file_get_contents('php://input'), JSON_THROW_ON_ERROR);

if ($_GET['action'] === 'sign'){
    $pkeyid = openssl_pkey_get_private("file:///var/www/private_key.pem");
    openssl_sign($d, $signature, $pkeyid, ALGO);
	echo json_encode(base64_encode($signature));
    openssl_free_key($pkeyid);
}
elseif ($_GET['action'] === 'verify') {
    $pkeyid = openssl_pkey_get_public("file:///var/www/public_key.pem");
    echo json_encode(openssl_verify(substr($d, 172), base64_decode(substr($d,0, 172)), $pkeyid, ALGO) === 1);
    openssl_free_key($pkeyid);
}

```

It uses `md5WithRSAEncryption` for signing. Testing locally, it takes the `$d` we pass in, computes its MD5, converts to hex, and appends it after `0x1ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff003020300c06082a864886f70d020505000410` to form a number, which is then signed with RSA.

The entire logic appears to have no flaws. It uses standard libraries and is basically unattackable. One idea is to use a proxy to switch IPs, which allows us to get two signatures for ip|ls -l. This way we have two sets of RSA m and c. Since the challenge provided a Dockerfile with the method for generating public/private keys using OpenSSL defaults, e is 65537, so we can find n by computing the GCD.

After obtaining two sets of signatures, to get the RSA m (the padded number), we follow the code logic. In Go, first SHA1:

```go
msg := ip + "|" + cmd
digest := sha1.Sum([]byte(msg))

b := new(bytes.Buffer)
err = json.NewEncoder(b).Encode(string(digest[:]))
```

Then PHP's MD5. We get two sets of m and c, but we can never find the common factor n. We suspect m is computed incorrectly. Looking at the code, Go JSON-encodes the SHA1 result and passes it to PHP which JSON-decodes it. This part is very suspicious—why use JSON encoding (wouldn't hex be better)? Let's set up a local environment and trace through it. (The challenge provides a Dockerfile.)

Start a Docker container, modify index.php to add a `var_dump($d);`, and modify the Go code to return PHP's result:

```go
fmt.Fprintln(w,string(body))
```

Now sign with the program and check the result:

```
string(38) "	��.���?-�KC��@�"
"K4FEmxz4yuTsjDAbRZQmHJ+MBiCSGaOnpZTLbThXpCkDYe3siAIPfihX6ppjN2Tz6XqOr4tF\/u1\/+ccfhj8NNLIL+2hknyDXbosmMBV8mEGYsMqQHAE0f+3OhDWlzN5RnteSMYNZbTipFErB8ZOWCiXmynWxsqJhyaN9J6\/\/h6I="
oops, hsm is jsonless?
```

$d turns out to be a string of length 38. It seems encoding is indeed the issue here. We need to check the result at each step. First, let's see what the JSON-encoded SHA1 result looks like in Go:

```go
package main

import (
	"bytes"
	"crypto/sha1"
	"encoding/json"
	"fmt"
)
func main() {
	msg := "172.17.0.1|ls -l"
	digest := sha1.Sum([]byte(msg))

	b := new(bytes.Buffer)
	json.NewEncoder(b).Encode(string(digest[:]))
	fmt.Print(string(b.Bytes()));
}
```

Running it:

```
"\u000e\t\u001d\ufffd\u0012\ufffd.\ufffd\ufffd\ufffd?-\ufffdKC\ufffd\u0005\ufffd@\ufffd"
```

Comparing with the normal SHA1 result:

```bash
Python 2.7.16 (default, Sep  2 2019, 11:59:44)
[GCC 4.2.1 Compatible Apple LLVM 10.0.1 (clang-1001.0.46.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> "\u000e\t\u001d\ufffd\u0012\ufffd.\ufffd\ufffd\ufffd?-\ufffdKC\ufffd\u0005\ufffd@\ufffd"
'\\u000e\t\\u001d\\ufffd\\u0012\\ufffd.\\ufffd\\ufffd\\ufffd?-\\ufffdKC\\ufffd\\u0005\\ufffd@\\ufffd'
>>> from hashlib import *
>>> sha1('172.17.0.1|ls -l').digest()
'\x0e\t\x1d\xbd\x12\x90.\xca\xf0\xd9?-\x98KC\xeb\x05\xa1@\xd1'
```

Due to Go's JSON encoding, many non-printable characters are converted to `U+fffd`, losing a lot of information.

After passing through the PHP interface, let's look at the result:

```php
$d = json_decode(file_get_contents('php://input'), JSON_THROW_ON_ERROR);
var_dump(file_get_contents('php://input'));
var_dump($d);
var_dump(bin2hex($d));
```

Result:

```
string(89) ""\u000e\t\u001d\ufffd\u0012\ufffd.\ufffd\ufffd\ufffd?-\ufffdKC\ufffd\u0005\ufffd@\ufffd"
"
string(38) "	��.���?-�KC��@�"
string(76) "0e091defbfbd12efbfbd2eefbfbdefbfbdefbfbd3f2defbfbd4b43efbfbd05efbfbd40efbfbd"
"K4FEmxz4yuTsjDAbRZQmHJ+MBiCSGaOnpZTLbThXpCkDYe3siAIPfihX6ppjN2Tz6XqOr4tF\/u1\/+ccfhj8NNLIL+2hknyDXbosmMBV8mEGYsMqQHAE0f+3OhDWlzN5RnteSMYNZbTipFErB8ZOWCiXmynWxsqJhyaN9J6\/\/h6I="
oops, hsm is jsonless?

```

`U+fffd` became `\xef\xbf\xbd`. So due to Go's JSON encoding issue, a lot of information is lost, causing the data before MD5 to have many identical characters. At the time of solving this challenge, I didn't think deeply about this. After obtaining n, I kept trying to construct signatures for arbitrary commands, and was puzzled—if we could construct them, wouldn't that mean this signature scheme is insecure? In fact, it cannot be done.

The correct solution exploits this Go encoding issue, which creates conditions for collision. We can collide to find a command like `cat *` that has the same result as `ls -l` under this encoding. However, the problem is that we need a very large number of IPs to provide collision data.

It can be observed that when Go retrieves the IP, it first uses `net.ParseIP` to parse the IP. If we prepend zeros before each number in the IP, the parsed result is still the original IP. Each number can have up to 256 leading zeros, and with four numbers, that already produces `2^32` different combinations, enough to collide between `ls -l` and `cat *`.

The official solution's C++ collision script had some issues compiling locally, so I added some additional header includes:

```c++
// g++ -std=c++17 -march=native -O3 -lcrypto -lpthread gewalt.cpp -o gewalt

#include <cassert>
#include <iomanip>
#include <string>
#include <sstream>
#include <iostream>
#include <functional>
#include <random>
#include <unordered_map>
#include <algorithm>
#include <thread>
#include <atomic>
#include <mutex>
#include <array>
#include <openssl/sha.h>

const unsigned num_threads = std::thread::hardware_concurrency();



static std::string hash(std::string const& s)
{
    SHA_CTX ctx;
    if (!SHA1_Init(&ctx)) throw;
    if (!SHA1_Update(&ctx, s.data(), s.length())) throw;
    std::string d(SHA_DIGEST_LENGTH, 0);
    if (!SHA1_Final((uint8_t *) &d[0], &ctx)) throw;
    return d;
}

static std::u32string kapot(std::string const& s)
{
    std::u32string r(s.size(), 0);
    size_t o = 0;

    for (size_t i = 0; i < s.length(); ) {

        auto T = [](uint8_t c) {
            return (c < 0x80)         ? 1   /* ASCII */
                 : (c & 0xc0) == 0x80 ? 0   /* continuation */
                 : (c & 0xe0) == 0xc0 ? 2   /* 2-byte chunk */
                 : (c & 0xf0) == 0xe0 ? 3   /* 3-byte chunk */
                 : (c & 0xf8) == 0xf0 ? 4   /* 4-byte chunk */
                 : -1;
        };

        uint32_t c = s[i++];
        auto cont = [&]() { c = (c << 6) | (s[i++] & 0x3f); };

        switch (T(c)) {

        case -1:
        case  0:
        invalid: c = 0xfffd; /* fall through */

        case  1:
        valid:   r[o++] = c; break;

        case  2:
                 if (c &= 0x1f, i+0 >= s.size() || T(s[i+0]))
                     goto invalid;
                 goto one;

        case  3:
                 if (c &= 0x1f, i+1 >= s.size() || T(s[i+0]) || T(s[i+1]))
                     goto invalid;
                 goto two;

        case  4:
                 if (c &= 0x1f, i+2 >= s.size() || T(s[i+0]) || T(s[i+1]) || T(s[i+2]))
                     goto invalid;
                 cont();
        two:     cont();
        one:     cont();
                 goto valid;

        }

    }

    r.resize(o);

    return r;
}

std::atomic<uint64_t> hcount = 0, kcount = 0;
typedef std::unordered_map<std::u32string, std::string> tab_t;
tab_t tab0, tab1;
std::mutex mtx;

std::array<uint8_t,4> ip;
std::string cmd0, cmd1;

class stuffer_t
{
    private:
        std::array<size_t,4> cnts;
        size_t step;
        std::string cmd;
    public:
        stuffer_t(size_t t, size_t s, std::string c) : cnts{t}, step(s), cmd(c) {}
        std::string operator()()
        {
            //XXX this is by far not the most efficient way of doing this, but yeah
            if (++cnts[3] >= cnts[0]) {
                cnts[3] = 0;
                if (++cnts[2] >= cnts[0]) {
                    cnts[2] = 0;
                    if (++cnts[1] >= cnts[0]) {
                        cnts[1] = 0;
                        cnts[0] += step;
                    }
                }
            }
            std::stringstream o;
            for (size_t i = 0; i < 4; ++i)
                o << (i ? "." : "")
                  << std::string(cnts[i], '0')
                  << (unsigned) ip[i];
            o << "|" << cmd;
            return o.str();
        }
};

void go(size_t tid)
{
    //XXX tid stuff is a hack, but YOLO

    bool one = tid & 1;

    stuffer_t next(tid >> 1, (num_threads + 1) >> 1, one ? cmd1 : cmd0);

    tab_t& mytab = one ? tab1 : tab0;
    tab_t& thtab = one ? tab0 : tab1;

    uint64_t myhcount = 0, mykcount = 0;

    while (1) {

        std::string r = next();

        {

            ++myhcount;

            auto h = hash(r);
            if ((h.size()+3)/4 < (size_t) std::count_if(h.begin(), h.end(),
                                            [](unsigned char c) { return c < 0x80; }))
                continue;

            ++mykcount;

            auto k = kapot(h);
            if (k.size() > 3 + (size_t) std::count(k.begin(), k.end(), 0xfffd))
                continue;

            std::lock_guard<std::mutex> lck(mtx);

            hcount += myhcount, myhcount = 0;
            kcount += mykcount, mykcount = 0;

            if (thtab.find(k) != thtab.end()) {

                mytab[k] = r;

                std::cerr << "\r\x1b[K"
                          << "\x1b[32m";
                std::cout << tab0[k] << std::endl
                          << tab1[k] << std::endl;
                std::cerr << "\x1b[0m";

                std::cerr << std::hex;
                bool first = true;
                for (uint32_t c: k)
                    std::cerr << (first ? first = false, "" : " ") << c;
                std::cerr << std::endl;

                std::cerr << std::dec << "hash count:  \x1b[35m" << hcount << "\x1b[0m";
                {
                    std::stringstream s;
                    s << std::fixed << std::setprecision(2) << log(hcount|1)/log(2);
                    std::cerr << " (2^\x1b[35m" << std::setw(5) << s.str() << "\x1b[0m" << ")" << std::endl;
                }
                std::cerr << "kapot count: " << "\x1b[35m" << kcount << "\x1b[0m";
                {
                    std::stringstream s;
                    s << std::fixed << std::setprecision(2) << log(kcount|1)/log(2);
                    std::cerr << " (2^\x1b[35m" << std::setw(5) << s.str() << "\x1b[0m)" << std::endl;
                }
                std::cerr << "table sizes: \x1b[35m"
                          << tab0.size() << "\x1b[0m \x1b[35m"
                          << tab1.size() << "\x1b[0m" << std::endl;

                exit(0);

            }

            if (mytab.size() < (1 << 20))
                mytab[k] = r;

        }

        hcount += myhcount;
        kcount += mykcount;

    }
}

void status()
{
    while (1) {

        {
            std::lock_guard<std::mutex> lck(mtx);

            std::cerr << "\r\x1b[K";
            std::cerr << "hash count: \x1b[35m" << std::setw(12) << hcount << "\x1b[0m ";
            {
                std::stringstream s;
                s << std::fixed << std::setprecision(2) << log(hcount|1)/log(2);
                std::cerr << "(2^\x1b[35m" << std::setw(5) << s.str() << "\x1b[0m) | ";
            }
            std::cerr << "kapot count: \x1b[35m" << std::setw(12) << kcount << "\x1b[0m ";
            {
                std::stringstream s;
                s << std::fixed << std::setprecision(2) << log(kcount|1)/log(2);
                std::cerr << "(2^\x1b[35m" << std::setw(5) << s.str() << "\x1b[0m) | ";
            }
            std::cerr << "tables: \x1b[35m"
                      << std::setw(9) << tab0.size() << " "
                      << std::setw(9) << tab1.size() << "\x1b[0m "
                      << std::flush;
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main(int argc, char **argv)
{

    if (argc < 2) {
        std::cerr << "\x1b[31mneed IPv4 in argv[1]\x1b[0m" << std::endl;
        exit(1);
    }
    {
        std::stringstream ss(argv[1]);
        for (auto& v: ip) {
            std::string s;
            std::getline(ss, s, '.');
            int n = std::atoi(s.c_str());
            if (n < std::numeric_limits<uint8_t>::min() || n > std::numeric_limits<uint8_t>::max())
                goto bad_ip;
            v = n;
        }
        if (!ss) {
bad_ip:
            std::cerr << "\x1b[31mbad IPv4 given?\x1b[0m" << std::endl;
            exit(2);
        }
    }


    if (argc < 4) {
        std::cerr << "\x1b[31mneed commands in argv[2] and argv[3]\x1b[0m" << std::endl;
        exit(2);
    }
    cmd0 = argv[2];
    cmd1 = argv[3];


    std::thread status_thread(status);
    std::vector<std::thread> ts;
    for (unsigned i = 0; i < num_threads; ++i)
        ts.push_back(std::thread(go, i));
    for (auto& t: ts)
        t.join();

}


```

Compilation may not find `lcrypto`. Add the lcrypto path to the compile command (on my local machine it's /usr/local/opt/openssl/lib):

```bash
g++ -std=c++17 -march=native -O3 -lcrypto -lpthread gewalt.cpp -o gewalt -L/usr/local/opt/openssl/lib
```

The script for interacting with Go:

```python
#!/usr/bin/env python3
import sys, requests, subprocess

benign_cmd = 'ls -l'
exploit_cmd = 'cat *'

ip, port = sys.argv[1], sys.argv[2]
url = 'http://{}:{}'.format(ip, port)

my_ip = requests.get(url + '/ip').text
print('[+] IP: ' + my_ip)

o = subprocess.check_output(['./gewalt', my_ip, benign_cmd, exploit_cmd])
print('[+] gewalt:' + o.decode())

payload = {}
for l in o.decode().splitlines():
    ip, cmd = l.split('|')
    payload['benign' if cmd == benign_cmd else 'pwn'] = ip, cmd

print(payload)

sig  = requests.post(url + '/sign', data={'ip': payload['benign'][0], 'cmd': payload['benign'][1]}).text
print('[+] sig: ' + sig)

r = requests.post(url + '/exec', data={'signature': sig[:172] + payload['pwn'][0]  + '|' + payload['pwn'][1]})
print(r.text)
```

```bash
 ⚙  SaV-ls-l-aaS  python solve.py 127.0.0.1 60601
[+] IP: 172.17.0.1
fffd fffd fffd fffd fffd fffd 55 fffd fffd fffd fffd c fffd fffd fffd fffd fffd fffd fffd fffd
hash count:  168104875 (2^27.32)
kapot count: 3477222 (2^21.73)
table sizes: 8745 8856
[+] gewalt:00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000172.000000000000000000000000000000000000000000000000000000000000000000000000000000000000000017.000000000000000000000000000000000000000000000000000000000000000000000000000000000.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001|ls -l
00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000172.17.000000000000000000000000.0000000000000000000000000000000000000001|cat *

{'pwn': (u'00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000172.17.000000000000000000000000.0000000000000000000000000000000000000001', u'cat *'), 'benign': (u'00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000172.000000000000000000000000000000000000000000000000000000000000000000000000000000000000000017.000000000000000000000000000000000000000000000000000000000000000000000000000000000.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001', u'ls -l')}
[+] sig: ODxSukwtu4rHICBpzT23WGD7DCJNawhA0DUN/tcyv1AgwNmS8OPUnO5FnBBDgiaVx5OTYd4OjH8LVbKiXUBUBuFx1OHDgKBKG5umkKMLt+350SlgMWY5qWny9tPIU3I+X0A9FcADCBCi6f0PkXfc0CSCZXuFu9rAKnVGsbmaUwY=00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000172.000000000000000000000000000000000000000000000000000000000000000000000000000000000000000017.000000000000000000000000000000000000000000000000000000000000000000000000000000000.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001|ls -l
hxp{FLAG}
```

References:

- https://ctftime.org/writeup/17966
