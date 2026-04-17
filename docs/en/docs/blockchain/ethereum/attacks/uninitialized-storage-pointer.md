# Uninitialized Storage Pointer

## Principle
An uninitialized storage pointer refers to a storage variable in the EVM that has not been initialized. This variable will point to the area of other variables, thereby modifying the values of other variables.

## Example
### Typical Example
Let's look at the following example:

```solidity
pragma solidity ^0.4.24;

contract example1{
    uint public a;
    address public b;

    struct Wallet{
        uint value;
        address addr;
    }

    function setValue(uint _a,address _b) public {
        a = _a;
        b = _b;
    }

    function attack(uint _value, address _addr) public {
        Wallet wallet;
        wallet.value = _value;
        wallet.addr = _addr;
    }
}
```

When this code is placed in Remix, it will show an Uninitialized Storage Pointer warning:

![uninitialized-storage-pointer-error](figure/uninitialized-storage-pointer-error.png)

After deployment, we first use the setValue function to set the values of a and b to 1 and 0x10aA1C20aD710B823f8c1508cfC12D5d1199117E respectively. We can see from the transaction that the setting was successful:

![uninitialized-storage-pointer-setvalue](figure/uninitialized-storage-pointer-setvalue.png)

Then we call the attack function, passing _value and _addr as 2 and 0xa3b0D4BBF17F38e00F68Ce73f81D122FB1374ff6 respectively. We can see from the transaction that a and b were overwritten by the passed _value and _addr values:

![uninitialized-storage-pointer-attack](figure/uninitialized-storage-pointer-attack.png)

The fix for this example is to use a mapping for struct initialization and use storage for copying:

```solidity
pragma solidity ^0.4.24;

contract example1{
    uint public a;
    address public b;

    struct Wallet{
        uint value;
        address addr;
    }

    mapping (uint => Wallet) wallets;

    function setValue(uint _a,address _b) public {
        a = _a;
        b = _b;
    }

    function fix(uint _id, uint _value, address _addr) public {
        Wallet storage wallet = wallets[_id];
        wallet.value = _value;
        wallet.addr = _addr;
    }
}
```

It's not just structs that encounter this problem — arrays have the same issue. Let's look at another example:

```solidity
pragma solidity ^0.4.24;

contract example2{
    uint public a;
    uint[] b;

    function setValue(uint _a) public {
        a = _a;
    }

    function attack(uint _value) public {
        uint[] tmp;
        tmp.push(_value);
        b = tmp;
    }
}
```

When this code is placed in Remix, it will also show an Uninitialized Storage Pointer warning:

![uninitialized-storage-pointer-error2](figure/uninitialized-storage-pointer-error2.png)

After deployment, we first use the setValue function to set the value of a to 1. We can see from the transaction that the setting was successful:

![uninitialized-storage-pointer-setvalue2](figure/uninitialized-storage-pointer-setvalue2.png)

Then we call the attack function, passing _value as 2. This is because the declared tmp array also uses slot 0. The slot where the array is declared stores its own length, so when push causes the array length to increase by 1, slot 0 stores the value 2 = a(old) + 1, hence a(new) = 2:

![uninitialized-storage-pointer-attack2](figure/uninitialized-storage-pointer-attack2.png)

The fix for this example is to initialize the local variable tmp when declaring it:

```solidity
pragma solidity ^0.4.24;

contract example2{
    uint public a;
    uint[] b;

    function setValue(uint _a) public {
        a = _a;
    }

    function fix(uint _value) public {
        uint[] tmp = b;
        tmp.push(_value);
    }
}
```

### 2019 BalsnCTF Bank
Using the writeup of the Bank challenge from 2019 Balsn CTF as a reference, here is an explanation of the uninitialized storage pointer attack method. The source code of the challenge contract is as follows:
```solidity
pragma solidity ^0.4.24;

contract Bank {
    event SendEther(address addr);
    event SendFlag(address addr);
    
    address public owner;
    uint randomNumber = 0;
    
    constructor() public {
        owner = msg.sender;
    }
    
    struct SafeBox {
        bool done;
        function(uint, bytes12) internal callback;
        bytes12 hash;
        uint value;
    }
    SafeBox[] safeboxes;
    
    struct FailedAttempt {
        uint idx;
        uint time;
        bytes12 triedPass;
        address origin;
    }
    mapping(address => FailedAttempt[]) failedLogs;
    
    modifier onlyPass(uint idx, bytes12 pass) {
        if (bytes12(sha3(pass)) != safeboxes[idx].hash) {
            FailedAttempt info;
            info.idx = idx;
            info.time = now;
            info.triedPass = pass;
            info.origin = tx.origin;
            failedLogs[msg.sender].push(info);
        }
        else {
            _;
        }
    }
    
    function deposit(bytes12 hash) payable public returns(uint) {
        SafeBox box;
        box.done = false;
        box.hash = hash;
        box.value = msg.value;
        if (msg.sender == owner) {
            box.callback = sendFlag;
        }
        else {
            require(msg.value >= 1 ether);
            box.value -= 0.01 ether;
            box.callback = sendEther;
        }
        safeboxes.push(box);
        return safeboxes.length-1;
    }
    
    function withdraw(uint idx, bytes12 pass) public payable {
        SafeBox box = safeboxes[idx];
        require(!box.done);
        box.callback(idx, pass);
        box.done = true;
    }
    
    function sendEther(uint idx, bytes12 pass) internal onlyPass(idx, pass) {
        msg.sender.transfer(safeboxes[idx].value);
        emit SendEther(msg.sender);
    }
    
    function sendFlag(uint idx, bytes12 pass) internal onlyPass(idx, pass) {
        require(msg.value >= 100000000 ether);
        emit SendFlag(msg.sender);
        selfdestruct(owner);
    }

}
```

Our goal is to execute emit SendFlag(msg.sender). Obviously, we cannot trigger it through the sendFlag function because we certainly cannot satisfy msg.value >= 100000000 ether.

If we carefully observe the code, we'll find two uninitialized storage pointers:

```solidity
modifier onlyPass(uint idx, bytes12 pass) {
[...]
    FailedAttempt info; <--
[...]
}

function deposit(bytes12 hash) payable public returns(uint) {
[...]
    SafeBox box; <--
[...]
}
```

Now we need to think about how to exploit them. Let's first look at the slot layout when the contract is first created:

```
-----------------------------------------------------
|     unused (12)     |          owner (20)         | <- slot 0
-----------------------------------------------------
|                 randomNumber (32)                 | <- slot 1
-----------------------------------------------------
|               safeboxes.length (32)               | <- slot 2
-----------------------------------------------------
|       occupied by failedLogs but unused (32)      | <- slot 3
-----------------------------------------------------
```

The layout of FailedAttempt in onlyPass is as follows — it will overwrite the contents of the original slot 0 through slot 2:

```
-----------------------------------------------------
|                      idx (32)                     |
-----------------------------------------------------
|                     time (32)                     |
-----------------------------------------------------
|          tx.origin (20)      |   triedPass (12)   |
-----------------------------------------------------
```

The layout of SafeBox in deposit is as follows — it will overwrite the contents of the original slot 0 through slot 1:

```
-----------------------------------------------------
| unused (11) | hash (12) | callback (8) | done (1) |
-----------------------------------------------------
|                     value (32)                    |
-----------------------------------------------------
```

If the tx.origin in FailedAttempt is large enough, it can overwrite safeboxes.length and change it to a sufficiently large value. This way, when calling the withdraw function, we can access failedLogs, allowing us to control callback to arbitrary content and control program execution flow.

So what do we need to control the execution flow to? As introduced in the opcodes section, jump instructions can only jump to JUMPDEST locations. We need to control the program execution flow to jump to the location just before emit SendFlag(msg.sender), which is location 070F shown below:

```
06F6 6A PUSH11 0x52b7d2dcc80cd2e4000000
0702 34 CALLVALUE
0703 10 LT
0704 15 ISZERO
0705 15 ISZERO
0706 15 ISZERO
0707 61 PUSH2 0x070f
070A 57 JUMPI
070B 60 PUSH1 0x00
070D 80 DUP1
070E FD REVERT
070F 5B JUMPDEST <---- here
0710 7F PUSH32 0x2d3bd82a572c860ef85a36e8d4873a9deed3f76b9fddbf13fbe4fe8a97c4a579
0731 33 CALLER
0732 60 PUSH1 0x40
0734 51 MLOAD
0735 80 DUP1
0736 82 DUP3
```


Finally, let's describe the specific attack steps:

- Find an account with a large starting address; all subsequent operations should use this account.
- Since failedLogs is a mapping plus array structure, calculate `target = keccak256(keccak256(msg.sender||3)) + 2`, which is the slot position of tx.origin | triedPass in failedLogs[msg.sender][0].
- Calculate the slot position of the first element in the safeboxes array, which is `base = keccak256(2)`.
- Calculate the index of target in the safeboxes array. Since each element in the safeboxes array occupies two slots, the index is `idx = (target - base) // 2`.
- Check if (target - base) % 2 equals 0. If so, tx.origin | triedPass can exactly overwrite unused | hash | callback | done, thus controlling callback; otherwise, return to the first step.
- Check if (msg.sender << (12 * 8)) is greater than idx. If so, safeboxes can access the target position; otherwise, return to the first step.
- Call the `deposit` function, setting the input hash value to 0x000000000000000000000000 and attaching 1 ether. This way, we set safeboxes[0].callback = sendEther.
- Call the `withdraw` function, setting the input idx value to 0 and pass value to 0x111111111111110000070f00. Since in the previous step we set safeboxes[0].callback = sendEther, this step will call the sendEther function, entering the if branch in onlyPass, which modifies triedPass in failedLogs[msg.sender][0] to the pass value we passed in. This operation also modifies safeboxes.length to msg.sender | pass.
- Call the `withdraw` function, setting the input idx value to the idx calculated in step four, and pass value to 0x000000000000000000000000. The program execution flow will then jump to emit SendFlag(msg.sender) and continue execution. Ultimately, the target contract will self-destruct, and the attack succeeds.

!!! note
    Note: The slot calculation rules in the attack steps can be found in the Ethereum Storage section.


## Challenges

### Balsn 2019
- Challenge Name: Bank

### RCTF 2020
- Challenge Name: roiscoin

### Byte 2019
- Challenge Name: hf

### Digital Economy Competition 2019
- Challenge Name: cow
- Challenge Name: rise

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.

## References

- [Analysis of Solidity Uninitialized Storage Pointers Security Risk](https://github.com/slowmist/papers/blob/master/Solidity_Unintialised_Storage_Pointers_Security_Risk.pdf)
- [Balsn CTF 2019 - Bank](https://x9453.github.io/2020/01/16/Balsn-CTF-2019-Bank/)
