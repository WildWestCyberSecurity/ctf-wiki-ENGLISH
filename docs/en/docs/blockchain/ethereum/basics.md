# Ethereum Basics

An introduction to some basic knowledge of smart contracts.

## Solidity

> Solidity is an object-oriented programming language for writing smart contracts. It is used for implementing smart contracts on various blockchain platforms, most notably, Ethereum. It was developed by Christian Reitwiessner, Alex Beregszaszi, and several former Ethereum core contributors to enable writing smart contracts on blockchain platforms such as Ethereum.  ------  from [wikipedia](https://en.wikipedia.org/wiki/Solidity)

Solidity is a high-level language for writing smart contracts, with syntax similar to JavaScript. On the Ethereum platform, smart contracts written in Solidity can be compiled into bytecode that runs on the Ethereum Virtual Machine (EVM).

You can refer to the [official website](https://docs.soliditylang.org/en/latest/) for learning; we will not go into further detail here.

## MetaMask

A very useful and the most widely used Ethereum wallet, recognized by its fox logo. Chrome provides a plugin for it. It can not only manage external accounts but also conveniently switch between test network environments and supports custom RPC networks.

!!! info
    An external account is typically controlled by a private key file. A user who possesses the private key has the right to use the Ether in the corresponding address's account. We usually call the software that manages these digital keys a wallet, and what we refer to as backing up a wallet is actually backing up the account's private key file.

## Remix

A browser-based Solidity compiler and integrated development environment that provides an interactive interface along with a series of features including compilation, testing, and deployment. It is very convenient to use. [http://remix.ethereum.org/](http://remix.ethereum.org/#optimize=false&runs=200&evmVersion=null)

## Accounts

In Ethereum, an important concept is the Account.

There are two types of accounts in Ethereum: Externally Owned Accounts (EOA) and Contract Accounts.

### Externally Owned Accounts

Externally Owned Accounts are created by humans and can store Ether. They are accounts controlled by public and private keys. Each external account has a pair of public and private keys, which are used to sign transactions, and its address is determined by the public key. External accounts cannot contain Ethereum Virtual Machine (EVM) code.

An external account has the following characteristics:

- Holds a certain amount of Ether
- Can send transactions and is controlled by a private key
- Has no associated code

### Contract Accounts

Contract accounts are accounts created by external accounts and contain contract code. The address of a contract account is calculated from the address of the contract creator and the number of transactions sent from that address.

A contract account has the following characteristics:

- Holds a certain amount of Ether
- Has associated code that is activated through transactions or calls sent by other contracts
- When a contract is executed, it can only manipulate the specific storage owned by the contract account

!!! note
    The private key generates the public key through a hash algorithm (Elliptic Curve Algorithm ECDSA-secp256k1). The Keccak-256 hash of the public key is computed, and then the last 160 bits (usually represented as a 40-character hexadecimal string) form the address. The public key and address can both be published, but the private key must be kept secret — do not lose it, because the assets in your account will be lost as well; do not let it be stolen, because the assets in your account will be stolen too. Therefore, the safekeeping of the private key is extremely important.

In Ethereum, these two types of accounts are collectively called "state objects" (storing state). External accounts store the Ether balance state, while contract accounts store the balance as well as the state of smart contracts and their variables. Through the execution of transactions, these state objects change, and Merkle trees are used to index and verify updates to state objects. An Ethereum account contains 4 parts:

- nonce: The total number of executed transactions, used to indicate the number of transactions sent from the account.
- balance: The amount of coins held, recording the account's Ether balance.
- storageRoot: The hash of the storage area, pointing to the storage data area of the smart contract account.
- codeHash: The hash of the code area, pointing to the smart contract code stored in the smart contract account.

Transactions between two external accounts are simply value transfers. However, a transaction from an external account to a contract account activates the contract account's code, allowing it to perform various operations (such as transferring tokens, writing to internal storage, creating new tokens, performing calculations, creating new contracts, etc.).

Unlike external accounts, contract accounts cannot initiate new transactions on their own. Instead, contract accounts can only trigger transactions in response to other transactions (from externally owned accounts or other contract accounts).

!!! note 
    Note: The biggest difference between contract accounts and external accounts is that contract accounts also store smart contracts.

## Transactions

Transactions in Ethereum mainly refer to signed data packets of messages sent from an external account to another account on the blockchain. They mainly contain the sender's signature, the recipient's address, and the amount of Ether transferred from the sender to the recipient, among other content. Every transaction on Ethereum requires a certain fee to pay for the computational overhead required for transaction execution. The fee for computational overhead is not calculated directly in Ether, but rather introduces Gas as the basic unit of execution overhead, which is converted to Ether through GasPrice.

GasPrice is adjusted based on market fluctuations to prevent the value of Ether from being affected by market prices. Transactions are an important part of Ethereum's overall structure, connecting Ethereum accounts and serving as a means of value transfer.

### Transaction Fees
- Gas: The basic unit measuring the computational resources consumed by a transaction
- Gas Price: The fee (in Ether) required per unit of Gas
- Gas Limit: The maximum amount of Gas the transaction sender is willing to pay for the execution of the transaction

!!! note 
    Note: If the actual Gas consumed by a transaction (Gas Used) is less than the Gas Limit, the executing miner will only charge the transaction fee corresponding to the actual computational overhead (Gas Used * Gas Price). However, if Gas Used exceeds the Gas Limit, the miner will find during execution that the Gas has been exhausted while the transaction has not been completed. In this case, the miner will roll back to the state before program execution and still charge the fee corresponding to the Gas Limit (GasPrice * Gas Limit). In other words, **GasPrice * Gas Limit** represents the maximum amount a user is willing to pay for a transaction.

### Transaction Content
A Transaction in Ethereum refers to a signed data packet storing a message sent from an external account to another account on the blockchain. It can be a simple transfer or a message containing smart contract code. A transaction contains the following:

- from: The address of the transaction sender, required;
- to: The address of the transaction recipient. If empty, it means the transaction creates a smart contract;
- value: The amount of Ether the sender wants to transfer to the recipient;
- data: An existing data field. If present, it indicates the transaction creates or invokes a smart contract;
- Gas Limit: The maximum amount of Gas the transaction is allowed to consume;
- GasPrice: The Gas unit price the sender is willing to pay to miners;
- nonce: A marker used to distinguish different transactions sent from the same account;
- hash: A hash value generated from the above information;
- r, s, v: Three parts of the transaction signature, generated by the sender's private key signing the transaction hash.

The above are the possible contents of a transaction in Ethereum. In different scenarios, there are three types of transactions:

- Transfer Transaction

A transfer is the simplest type of transaction, sending Ether from one account to another. When sending a transfer transaction, you only need to specify the sender, recipient, and the amount of Ether to transfer (when sending transactions via the client, Gas Limit, Gas Price, nonce, hash, and signature can be generated using default methods), as shown below:

```nodejs
web3.eth.sendTransaction({
    from: "0x88D3052D12527F1FbE3a6E1444EA72c4DdB396c2",
    to: "0x75e65F3C1BB334ab927168Bd49F5C44fbB4D480f",
    value: 1000
})
```

- Contract Creation Transaction

Contract creation refers to deploying a contract onto the blockchain, which is also accomplished through a transaction. When creating a contract, the to field is an empty string, and the data field contains the compiled binary code of the contract. When the contract is later invoked, the execution result of this code will serve as the contract code, as shown below:

```
web3.eth.sendTransaction({
    from: "0x88D3052D12527F1FbE3a6E1444EA72c4DdB396c2",
    data: "contract binary code"
})
```

- Contract Execution Transaction

In this type of transaction, the to field is the address of the smart contract to be called, and the data field specifies the method to be called and the parameters passed to that method, as shown below:

```
web3.eth.sendTransaction({
    from: "0x88D3052D12527F1FbE3a6E1444EA72c4DdB396c2",
    to: "0x75e65F3C1BB334ab927168Bd49F5C44fbB4D480f",
    data: "hash of the invoked method signature and encoded parameters"
})
```

!!! info
    Based on the content of the to and data fields, you can also determine the type of transaction in reverse, and then continue the analysis.

## Interact with Contracts

- Interact directly through Remix
- Remix cannot achieve automation, so developers have done some work
    - Python's web3.py library
    - Node.js's web3.js library
    - [Infura](https://infura.io/) provides RPC APIs for developers to call, currently supporting Ethereum, Eth2, and Filecoin

Using the RPC API provided by [Infura](https://infura.io/), you can interact automatically with smart contracts using the web3.py or web3.js library.

Infura currently supports access points for the following networks:

|Network       |Description             |URL|
|-------------|------------------------|---------------------------------------|
|Mainnet      |JSON-RPC over HTTPs     |https://mainnet.infura.io/v3/YOUR-PROJECT-ID |
|Mainnet      |JSON-RPC over websockets|wss://mainnet.infura.io/ws/v3/YOUR-PROJECT-ID|
|Ropsten      |JSON-RPC over HTTPs     |https://ropsten.infura.io/v3/YOUR-PROJECT-ID |
|Ropsten      |JSON-RPC over websockets|wss://ropsten.infura.io/ws/v3/YOUR-PROJECT-ID|
|Rinkeby      |JSON-RPC over HTTPs     |https://rinkeby.infura.io/v3/YOUR-PROJECT-ID |
|Rinkeby      |JSON-RPC over websockets|wss://rinkeby.infura.io/ws/v3/YOUR-PROJECT-ID|
|Kovan        |JSON-RPC over HTTPs     |https://kovan.infura.io/v3/YOUR-PROJECT-ID   |
|Kovan        |JSON-RPC over websockets|wss://kovan.infura.io/ws/v3/YOUR-PROJECT-ID  |
|Görli        |JSON-RPC over HTTPs     |https://goerli.infura.io/v3/YOUR-PROJECT-ID  |
|Görli        |JSON-RPC over websockets|wss://goerli.infura.io/ws/v3/YOUR-PROJECT-ID |
|Mainnet(eth2)|JSON-RPC over HTTPs     |https://YOUR-PROJECT-ID:YOUR-PROJECT-SECRET@eth2-beacon-mainnet.infura.io|
|pyrmont(eth2)|JSON-RPC over websockets|wss://YOUR-PROJECT-ID:YOUR-PROJECT-SECRET@eth2-beacon-mainnet.infura.io|
|Filecoin     |JSON-RPC over HTTPs     |https://YOUR-PROJECT-ID:YOUR-PROJECT-SECRET@filecoin.infura.io|
|Filecoin     |JSON-RPC over websockets|wss://YOUR-PROJECT-ID:YOUR-PROJECT-SECRET@filecoin.infura.io|

!!! note 
    Note: When using these, please make sure to replace YOUR-PROJECT-ID or YOUR-PROJECT-SECRET in the above URLs with the Project ID or Project Secret from your Infura dashboard.

Below is an example of using web3.py and the Infura API to interact with a smart contract by calling a function with the function selector 0x00774360:

```python
from web3 import Web3, HTTPProvider

w3 = Web3(Web3.HTTPProvider("https://rinkeby.infura.io/v3/YOUR-PROJECT-ID"))

contract_address = "0x31c883a9aa588d3f890c26c7844062d99444b5d6"
private = "your private key"
public = "0x75e65F3C1BB334ab927168Bd49F5C44fbB4D480f"

def deploy(public):
    txn = {
        'from': Web3.toChecksumAddress(public),
        'to': Web3.toChecksumAddress(contract_address),
        'gasPrice': w3.eth.gasPrice,
        'gas': 3000000,
        'nonce': w3.eth.getTransactionCount(Web3.toChecksumAddress(public)),
        'value': Web3.toWei(0, 'ether'),
        'data': '0x007743600000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000001a6100016100016100016100016100016100650361000161fbfbf1000000000000',
    }
    signed_txn = w3.eth.account.signTransaction(txn, private)
    txn_hash = w3.eth.sendRawTransaction(signed_txn.rawTransaction).hex()
    txn_receipt = w3.eth.waitForTransactionReceipt(txn_hash)
    print("txn_hash=", txn_hash)
    return txn_receipt

print(deploy(public))
```

## tx.origin vs msg.sender

- Here we distinguish between tx.origin and msg.sender. `msg.sender` is the direct caller of the function. When a user manually calls the function, it is the address of the account that initiated the transaction, but it can also be the address of a smart contract that calls the function. On the other hand, `tx.origin` is always the original initiator of the transaction, regardless of how many intra-contract or cross-contract function calls occur in between, and it is always an account address rather than a contract address.
- Given a scenario like: a user calls Contract B through Contract A, then:
    + For Contract A: both tx.origin and msg.sender are the user
    + For Contract B: tx.origin is the user, msg.sender is Contract A
