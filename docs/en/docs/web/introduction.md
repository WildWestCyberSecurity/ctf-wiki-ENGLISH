# Web Introduction

With the emergence of Web 2.0, social networks, microblogs, and many other new types of internet products, web-based internet applications have become increasingly widespread. During the process of enterprise informatization, various applications are built on the web platform. The rapid development of web business has also attracted strong attention from hackers, leading to the prominence of web security threats. Hackers exploit vulnerabilities in website operating systems and web service programs to gain control of web servers — in mild cases they tamper with web page content, in more serious cases they steal important internal data, and in the worst cases they inject malicious code into web pages, causing harm to website visitors.

In CTF competitions, WEB is one of the major categories. WEB challenges are diverse, involve many scattered knowledge points, are highly time-sensitive, and closely follow current hot vulnerabilities, making them close to real-world scenarios.

WEB challenges include but are not limited to: SQL injection, XSS (Cross-Site Scripting), CSRF (Cross-Site Request Forgery), file upload, file inclusion, framework security, common PHP vulnerabilities, code auditing, etc.

## SQL Injection

SQL injection is an attack that involves inserting or appending SQL syntax into application (user) input parameters, which are then passed to the backend SQL server for parsing and execution. Its root causes can be attributed to the combination of the following two factors:

1. Developers construct SQL statements using string concatenation when handling interactions between the application and the database
2. User-controllable parameters are concatenated into SQL statements without sufficient filtering

## XSS (Cross-Site Scripting)

Cross-Site Scripting (XSS) — abbreviated as XSS to avoid confusion with Cascading Style Sheets (CSS). Malicious attackers insert malicious HTML code into web pages. When users browse these pages, the embedded HTML code is executed, allowing attackers to achieve special malicious purposes against users.

## Command Execution

When an application needs to call external programs to process content, it uses functions that execute system commands. For example, in PHP, functions like `system`, `exec`, `shell_exec`, etc. When users can control the parameters of command execution functions, they can inject malicious system commands into normal commands, resulting in command execution attacks. This section primarily covers command execution vulnerabilities in PHP; details for Java and other applications are to be supplemented.

## File Inclusion

If client-side user input is allowed to control files that are dynamically included on the server side, it may lead to malicious code execution and sensitive information disclosure. This mainly includes two forms: Local File Inclusion (LFI) and Remote File Inclusion (RFI).

## CSRF (Cross-Site Request Forgery)

Cross-Site Request Forgery (CSRF) is an attack that forces logged-in users to perform certain actions without their knowledge. Because the attacker cannot see the response to the forged request, CSRF attacks are primarily used to perform actions rather than steal user data. When the victim is a regular user, CSRF can perform operations such as transferring funds or sending emails without the user's knowledge. However, if the victim is an administrator, CSRF could threaten the security of the entire web system.

## SSRF (Server-Side Request Forgery)

SSRF (Server-Side Request Forgery) is a security vulnerability where the attacker constructs a request that is initiated by the server. Generally, SSRF attacks target internal systems that are inaccessible from the external network.

## File Upload

During website operations, it is inevitable to update certain pages or content, which requires the use of file upload functionality. If uploaded files are not restricted or if restrictions can be bypassed, this functionality may be exploited to upload executable files or scripts to the server, further leading to server compromise.

## Clickjacking

Clickjacking was first coined by internet security experts Robert Hansen and Jeremiah Grossman in 2008.

It is a visual deception technique. On the web, it uses an iframe to embed a transparent, invisible page, causing users to unknowingly click on positions that the attacker wants to trick them into clicking.

Due to the emergence of clickjacking, anti-frame nesting techniques were developed, since clickjacking relies on iframe-embedded pages for the attack.

The following code is the most common example of preventing frame nesting:

```js
if(top.location!=location)
    top.location=self.location;
```

## VPS (Virtual Private Server)

VPS (Virtual Private Server) technology divides a single server into multiple virtual dedicated servers. VPS implementation technologies include container technology and virtualization technology. Within a container or virtual machine, each VPS can be assigned an independent public IP address, independent operating system, and achieve isolation of disk space, memory, CPU resources, processes, and system configuration between different VPS instances, simulating the experience of exclusive use of computing resources for users and applications. A VPS can reinstall operating systems, install programs, and restart the server independently, just like a dedicated server. VPS provides users with the freedom of management and configuration, and can be used for enterprise virtualization or IDC resource leasing.

IDC resource leasing is provided by VPS providers. Differences in hardware, VPS software, and sales strategies among different VPS providers lead to significant variations in the VPS experience. Especially when VPS providers oversell, causing physical servers to be overloaded, VPS performance will be greatly affected. Relatively speaking, container technology has higher hardware utilization efficiency than virtual machine technology and is easier to oversell, so container VPS prices are generally lower than virtual machine VPS prices.

## Race Condition

Race condition vulnerabilities are server-side vulnerabilities. Since the server processes requests from different users concurrently, improper handling of concurrency or unreasonable design of operation logic sequences can lead to such issues.

## XXE

XXE Injection, or XML External Entity Injection, is a security issue that arises when processing insecure external entity data.

In the XML 1.0 standard, the XML document structure defines the concept of entities. Entities can be called within a document through predefined references, and entity identifiers can access local or remote content. If a "contaminated" source is introduced during this process, it may lead to information disclosure and other security issues after XML document processing.

## XSCH

Due to negligence by website developers during development with Flash, Silverlight, etc., the cross-domain policy file (crossdomain.xml) is not configured correctly, leading to security issues. For example:

```xml
<cross-domain-policy>
    <allow-access-from domain="*"/>
</cross-domain-policy>
```

Because the cross-domain policy file is configured with `*`, Flash from any domain can interact with it, allowing requests to be made and data to be retrieved.

## Privilege Escalation (Missing Function Level Access Control)

Privilege escalation vulnerabilities are common security vulnerabilities in web applications. The threat lies in the fact that a single account can control all user data on the site. Of course, this data is limited to the data corresponding to the vulnerable functionality. The root cause of privilege escalation vulnerabilities is that developers place too much trust in client request data during data addition, deletion, modification, and querying operations, and neglect permission checks. Therefore, testing for privilege escalation is a process of competing with developers on attention to detail.

## Sensitive Information Disclosure

Sensitive information refers to information that is not publicly known, has actual and potential value, and whose loss, improper use, or unauthorized access may cause harm to society, enterprises, or individuals. This includes: personal privacy information, business operation information, financial information, personnel information, IT operations information, etc.
Disclosure channels include Github, Baidu Wenku, Google Code, website directories, etc.

## Security Misconfiguration

Security Misconfiguration: Sometimes, using default security configurations may make an application vulnerable to various attacks. It is crucial that the best available security configurations are used across deployed applications, web servers, database servers, operating systems, code libraries, and all components related to the application.

## HTTP Request Smuggling
In the HTTP protocol, there are two headers that specify the end of a request: Content-Length and Transfer-Encoding. In complex network environments, different servers implement the RFC standard in different ways. Therefore, the same HTTP request may be processed differently by different servers, creating security risks.

## TLS Poisoning

In the TLS protocol, there is a session resumption mechanism. When a client supporting this feature visits a malicious TLS server, the client stores the session issued by the malicious server. When the client resumes the session, combined with DNS Rebinding, it is possible to make the client send the malicious session to internal network services, achieving an SSRF attack effect. This can include arbitrary writes to internal services such as Memcached, which combined with other vulnerabilities can lead to RCE and other hazards.

## XS-Leaks

Cross-Site Leaks (also known as XS-Leaks/XSLeaks) are a class of vulnerabilities derived from side channels built into the web platform. The principle is to leverage these side channels on the web to reveal sensitive user information, such as user data in other web applications, information about the user's local environment, or information about the internal network the user is connected to.

This attack exploits a core principle of the web platform — composability — which allows websites to interact with each other, and abuses legitimate mechanisms to infer user information. The main difference between this attack and Cross-Site Request Forgery (CSRF) is that XS-Leaks does not forge user requests to perform actions, but rather is used to infer and obtain user information.

Browsers provide a variety of features to support interaction between different web applications; for example, browsers allow one website to load sub-resources, navigate, or send messages to another application. While these behaviors are typically restricted by web platform security mechanisms (such as the Same-Origin Policy), XS-Leaks exploit various behaviors during website interactions to leak user information.


## WAF

Web Application Firewall (WAF), also known as Website Application-Level Intrusion Prevention System. According to an internationally recognized definition: a WAF is a product that provides protection specifically for web applications by enforcing a series of security policies for HTTP/HTTPS.

## IDS

IDS stands for Intrusion Detection Systems. Professionally speaking, it refers to the monitoring of network and system operations through software and hardware according to certain security policies, detecting various attack attempts, attack behaviors, or attack results as much as possible, to ensure the confidentiality, integrity, and availability of network system resources. To use an analogy: if a firewall is the door lock of a building, then IDS is the surveillance system inside the building. Once a thief climbs in through a window, or an insider engages in unauthorized behavior, only a real-time surveillance system can detect the situation and issue a warning.

## IPS

Intrusion Prevention System (IPS) is a network security facility that complements antivirus software and firewalls (packet filter, application gateway). An IPS is a network security device that can monitor network or network device data transmission behavior and can immediately interrupt, adjust, or isolate abnormal or harmful network data transmissions.

## References

- [WEB Penetration Wiki](http://wiki.jeary.org/#!websec.md)
