# CSRF

## Introduction to CSRF

CSRF, short for Cross-Site Request Forgery, is often confused with XSS. The two key aspects of CSRF are: the cross-site nature of the request and the forgery of the request. Because the target site lacks token or referer defenses, every parameter of a user's sensitive operations can be known by an attacker, who can then forge an identical request to achieve malicious purposes under the user's identity.

## Types of CSRF

By request type, CSRF can be classified into GET-based and POST-based.

By attack method, it can be classified into HTML CSRF, JSON Hijacking, Flash CSRF, etc.

### HTML CSRF

Exploiting HTML elements to send CSRF requests is the most common CSRF attack.

HTML tags that can set `src/href` and other link address attributes can all initiate a GET request, such as:

```html
<link href="">
<img src="">
<img lowsrc="">
<img dynsrc="">
<meta http-equiv="refresh" content="0; url=">
<iframe src="">
<frame src="">
<script src=""></script>
<bgsound src=""></bgsound>
<embed src=""></bgsound>
<video src=""></video>
<audio src=""></audio>
<a href=""></a>
<table background=""></table>
......
```

There are also CSS styles:

```css
@import ""
background:url("")
......
```

Forms can also be used to forge POST-based requests.

```html
<form action="http://www.a.com/register" id="register" method="post">
  <input type=text name="username" value="" />
  <input type=password name="password" value="" />
</form>
<script>
  var f = document.getElementById("register");
  f.inputs[0].value = "test";
  f.inputs[1].value = "passwd";
  f.submit();
</script>
```

### Flash CSRF

Flash also has various ways to initiate network requests, including POST.

```js
import flash.net.URLRequest;
import flash.system.Security;
var url = new URLRequest("http://target/page");
var param = new URLVariables();
param = "test=123";
url.method = "POST";
url.data = param;
sendToURL(url);
stop();
```

Flash can also use `getURL`, `loadVars`, and other methods to initiate requests.

```js
req = new LoadVars();
req.addRequestHeader("foo", "bar");
req.send("http://target/page?v1=123&v2=222", "_blank", "GET");
```

## CSRF Defenses

### CAPTCHA

A CAPTCHA forces the user to interact with the application before completing the final request.

### Referer Check

Check whether the request comes from a legitimate source. However, the server cannot always obtain the Referer.

### Token

The fundamental reason CSRF attacks succeed is that all parameters of critical operations can be guessed by the attacker.

Keep the original parameters unchanged, and add a new parameter Token with a random value. In practice, the Token can be stored in the user's Session or in the browser's Cookies.

The Token must be sufficiently random. Additionally, the purpose of the Token is not to prevent duplicate submissions, so for convenience, the same Token can be used within a user's valid session lifetime before it is consumed. However, if the user has already submitted the form, the Token has been consumed and a new Token should be generated.

The Token should also be kept confidential. If the Token appears in the URL, it may be leaked through the Referer. The Token should be placed in forms as much as possible, and sensitive operations should be changed from GET to POST, submitted via forms or AJAX, to prevent Token leakage.
