# Protocol Analysis Overview

Network protocols are collections of rules, standards, or conventions established for data exchange in computer networks. For example, when a microcomputer user and an operator of a large mainframe communicate over a network, since these two data terminals use different character sets, the commands entered by the operators are not recognized by each other. To enable communication, it is specified that each terminal must first convert characters from its own character set to characters of a standard character set before entering the network for transmission. After reaching the destination terminal, the characters are converted back to the character set of that terminal. Of course, for incompatible terminals, besides character set conversion, other characteristics such as display format, line length, number of lines, and screen scrolling mode also need to be converted accordingly.

Accordingly, in this chapter on protocol analysis, we will introduce this topic from the following aspects:

- Introduction to common features of `Wireshark`
- `HTTP` protocol analysis
- `HTTPS` protocol analysis
- `FTP` protocol analysis
- `DNS` protocol analysis
- `WIFI` protocol analysis
- `USB` protocol analysis
