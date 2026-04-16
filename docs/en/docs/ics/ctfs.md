# ICS_CTF Competitions

> The content of this section on ICS CTF competitions comes from the author's own competition experience. If there are any inaccuracies, please feel free to point them out.

## Key Focus Areas of Domestic ICS Competitions

Using the CTF classification model, we summarize and analyze the key points in current ICS competitions.

|Competition Type|Focus Areas|Similarities/Differences with CTF|
|-------|------|-------|
|Intranet Penetration|Web-side penetration testing, CMS systems, ICS display systems, database systems|Related to Web penetration
|Reverse Analysis|Firmware analysis, ICS software reverse engineering|Real-world scenario reverse engineering|
|ICS Protocols|ICS traffic analysis, Misc category|Misc traffic analysis, ICS scenario traffic characteristics|
|ICS Programming|PLC programming, HMI configuration, RTU programming, etc.|Practical use of ICS configuration software, ladder diagram recognition and analysis|

Based on vulnerability types, the challenge categories can be further refined, including common ICS security issues such as Web injection, firmware weak passwords, backdoor programs, protocol replay and logic issues, and configuration deployment problems.

|Competition Type|Vulnerability Type|
|-------|------|
|Intranet Penetration|Web-based (SQL, XSS, command injection, sensitive file disclosure `.git/.idea/.project`, etc.)
|Reverse Analysis|Firmware analysis, ICS software reverse engineering|Real-world software, DLL, ELF, MIPS reverse engineering|
|ICS Protocols|ICS traffic analysis, Misc category|Misc traffic analysis, ICS scenario traffic characteristics|
|ICS Programming|PLC programming, HMI configuration|Practical use of ICS configuration software, ladder diagram recognition and analysis|

Regarding the ICS CTF challenge types that have appeared or currently exist, there are actually many overlapping points with CTF competitions, so we won't elaborate on those here. Instead, we will mainly discuss the areas where ICS CTF differs from regular CTF competitions.

## Web Penetration (Web)

This section mainly discusses the characteristics of ICS Web penetration:

- Highly integrated with business scenarios. For example, in industrial control, the Web side mainly displays control parameters and operational status information of the current usage scenario. If a man-in-the-middle attack occurs on the intranet and the HMI display device cannot synchronize with real-time running devices such as PLCs, the system will raise alarms or errors.
- Generally uses common technologies to present Web interfaces, with Windows operating systems as the primary platform, including seemingly outdated systems such as WinCC, Windows Server, Windows 98/2000/XP.
- Web penetration targets often have multiple open ports, such as FTP, HTTPS, Telnet, SNMP, NTP service ports. If Web penetration cannot break through, you can try other ports.
- Since ICS environments are generally on internal networks, intranet hijacking is often quite effective. However, if the intranet is configured with static IPs or other protective measures, ARP spoofing and similar intranet hijacking methods may not work.
- Sensitive information disclosure and incomplete configuration files are common issues in ICS Web publishing. This includes not only project information like `.git/.idea/.project`, but also potential path traversal, command injection, and weak password issues.

### Example Challenge

To be supplemented

## Reverse Analysis (Reverse)

This section mainly discusses the characteristics of ICS reverse engineering:

- ICS operating systems are generally RTOS (Real Time Operating System), such as VxWorks, uC/OS, and other real-time operating systems. Before reverse engineering, you need to be fairly familiar with their architecture and instruction sets. If you are not, please study them on your own.
- Common targets in ICS firmware reverse engineering include ICS project encryption algorithms, hardcoded keys, hardcoded backdoors, and other common firmware reverse engineering vulnerabilities. If a stack overflow vulnerability is found, it can often cause the target device to crash (i.e., a DoS consequence).
- ICS firmware often has encryption and compression. In the first step of extraction, you need to decompress or decrypt it. This depends on the specific vendor and cannot be generalized.
- There are cases where ICS firmware cannot be fully reverse-analyzed.

### Example Challenge

Challenge name: tplink_tddp

From the challenge description, we know that the key analysis target is tddp. The challenge attachment is a firmware image. Using binwalk to parse it, we find tddp in `usr/bin`. Then by searching the keyword "tddp" on Google, we can discover that there is a vulnerability for this protocol in TP-Link routers. It runs on UDP port 1040. When the second byte of the sent data (using tddp v1 protocol) is 0x31 (CMD_FTEST_CONFIG), it leads to remote code execution. Reference link: https://paper.seebug.org/879/ .
Drag tddp into IDA and search for the string `CMD_FTEST_CONFIG` to find that it will execute the `sub_A580` function.

![tddp](./figure/tddp_1.png)

At this point, the flag can already be submitted, but you can also continue the analysis by setting up an ARM environment with QEMU to run the file system for dynamic debugging.

## ICS Protocols (Protocol)

Challenge characteristics:

- ICS protocols are designed for industrial control scenarios and have characteristics such as simplicity, efficiency, and low latency. Therefore, attacks against these protocols can employ simple attack methods such as replay and command injection.
- ICS protocols include not only public protocols but also numerous proprietary protocols. The specific details of these protocols need to be reverse-engineered or data needs to be collected to reconstruct data functionality. Examples include Modbus, DNP3, Melsec-Q, S7, Ethernet/IP, etc.
- ICS protocols may cause target PLC, DCS, RTU, and other devices to crash or become unable to restart. Using Fuzz-based methods can quickly and efficiently discover PLC crash vulnerabilities.
- ICS protocols may contain numerous operations targeting devices such as PLCs. Users need to distinguish between legitimate requests and anomalous requests, which requires experience and the ability to research and infer the usage logic of the current traffic. This scenario is well-suited for machine learning, which could be an exploration direction worth considering.
- The best practical defense solution for ICS scenarios is actually bypass detection — using optical splitters to feed traffic into analysis systems, enabling security monitoring of the target system without affecting normal business operations.

### Example Challenge

Challenge name: mms.pcap

This is a challenge about MMS protocol analysis. When you get the file, open it with Wireshark (some challenges require using tcpdump to open).

![mms](./figure/mms_1.png)

You can see that MMS appears in the challenge. This is a protocol from the IEC 61850 standard for substations. Before solving the challenge, you should be familiar with the protocol's hierarchical structure and function codes (especially handshake packets and read/write function packets). Then filter MMS protocol packets and search for the "flag" string.

![mms](./figure/mms_2.png)

You can see that packet 1764 contains flag.txt, with the function code being fileDirectory. So we guess that fileOpen will be used to open it. Filter the request packets for this function code.

![mms](./figure/mms_3.png)

We can find that after packet 1764, packets 1771, 1798, 1813, and 4034 all performed open operations on flag.txt. Next, check the packet after each of these to see if there is a fileRead that performed a read. Ultimately, at packet 1800, we find that it read flag.txt, so packet 1801 is the response packet.

![mms](./figure/mms_4.png)

The content is a base64-encoded image. Decoding it gives you the corresponding FLAG content.

This is a simple protocol analysis challenge that mainly tests the solver's familiarity with the protocol structure hierarchy and function codes, as well as proficiency with Wireshark's filtering and analysis features, which can achieve twice the result with half the effort. In competition challenges, it may not always be possible to find relevant clues by searching for the "flag" string. Related information may be hidden in the payloads of multiple packets or in length fields, requiring careful observation and summarization.



## ICS Programming and Configuration (Program)

ICS programming and configuration are the core and focus of ICS system operation. The characteristics of this type of challenge are generally:

- The core of ICS programming is understanding ICS business logic. ICS programming follows IEC 61131-3 (the first standard in ICS history to achieve combined programming for PLC, DCS, motion control, SCADA, etc. — IEC 61131-3), which includes 5 programming language standards: 3 graphical languages (Ladder Diagram, Sequential Function Chart, and Function Block Diagram), and 2 textual languages (Instruction List and Structured Text).
- ICS devices can often be debugged online, allowing control of certain input/output ports to implement forced start/stop functionality. If these functions can be replayed remotely, the attack impact becomes even more severe.
- ICS devices have diverse connection methods. Serial connections are generally used, but current device development also supports Ethernet, USB interfaces, and other new methods. If the network port doesn't work, try serial or USB.
- ICS configurations can be very complex, potentially connecting hundreds or even thousands of inputs and outputs. Configurations become even more complicated when new components are added. In such cases, you need to go through it slowly and sort it out bit by bit.

### Example Challenge

Challenge name: PLC Ladder Diagram Calculation

This type of challenge mainly tests the solver's familiarity with the programming software, programming languages, and workflow of the corresponding products (PLC, RTU, upper computer software). When you receive the challenge, you need to have the appropriate supporting software to open it. For example, the figure below shows a Siemens PLC programming challenge where the flag is the final output value of the ladder diagram. So we use TIA Portal software to open the challenge:

![plc](./figure/PLC_1.png)

There are generally two approaches for this type of challenge:

1. Simulate running the challenge code, then directly check the value at the corresponding location. However, this generally encounters many errors and may be difficult to run directly.
2. Read it directly — for those familiar with ladder diagrams, this is very straightforward.

The above are some of my insights and experiences from participating in ICS competitions. I hope they can provide some guidance for future participants.
