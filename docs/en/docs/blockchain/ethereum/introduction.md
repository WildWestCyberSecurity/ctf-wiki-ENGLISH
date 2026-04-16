# Ethereum Overview

!!! info
    In CTF competitions, the most commonly encountered blockchain security content so far has been Ethereum security.

## Definition

> Ethereum is a decentralized, open-source blockchain featuring smart contract functionality. Ether (ETH) is the native cryptocurrency of the platform. It is the second-largest cryptocurrency by market capitalization, after Bitcoin. Ethereum is the most actively used blockchain.  ------  from [wikipedia](https://en.wikipedia.org/wiki/Ethereum)

Ethereum is the representative product of Blockchain 2.0. Because its underlying technology uses blockchain, it inherits various blockchain characteristics, one of which is that **once code is deployed on-chain, it is difficult to tamper with or modify**. Therefore, we need to pay extra attention to its security.

Smart Contracts are the most important concept in Ethereum, allowing trusted transactions to be conducted without third parties. These transactions are traceable and irreversible.

## Blockchain in CTF

Ethereum Security challenges in CTF competitions are relatively straightforward, mainly involving Solidity Security. Below is an introduction to the basic skills required.

### Requirements

- Have an understanding of basic blockchain knowledge and the nature of transactions
- Be proficient in the Solidity programming language and the Ethereum Virtual Machine (EVM) execution mechanism
- Be familiar with various test networks, including private chains
- Be proficient in using tools and libraries such as Remix, MetaMask, web3.js, and web3.py
- Understand and master various Ethereum smart contract vulnerabilities and their attack principles
- Have a thorough understanding of the underlying opcodes
- Have strong program comprehension and reverse analysis abilities

!!! note
    Note: Most Ethereum smart contracts do not have their source code publicly available; only the bytecode is available. Therefore, reverse analysis skills are required.
