# Basic Introduction to Android Reverse Engineering

First, we need to clarify the purpose of Android reverse engineering: **to analyze and understand the functionality of a program**. We can naturally consider two aspects (methods and targets):

- Analysis methods, which can use the following approaches:
    - Static analysis: Reverse engineer the source code, then read and analyze it.
    - Dynamic analysis: Dynamically debug the code. Generally speaking, dynamic analysis cannot be separated from static analysis.
- Analysis targets, which generally fall into two categories:
    - Java layer code
    - Native layer code

It is clear that to analyze Android applications, a basic understanding of both Java layer and native layer knowledge is necessary.

Currently, Android reverse engineering is mainly applied in the following areas:

1. App security auditing
2. System vulnerability mining
3. Malware detection and analysis
4. Technical analysis of competitor products
5. Removing security mechanisms
