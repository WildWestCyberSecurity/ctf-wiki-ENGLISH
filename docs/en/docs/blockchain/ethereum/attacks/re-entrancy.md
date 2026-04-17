# Re-Entrancy

Re-entrancy is a classic attack on smart contracts. The re-entrancy attack on Ethereum's The DAO project directly led to the hard fork of Ethereum (ETH) and Ethereum Classic (ETC).

## Principle
Suppose a bank contract implements the following withdrawal functionality. When `balanceOf[msg.sender]` is sufficient, the contract transfers the corresponding amount of Ether to the caller and reduces `balanceOf` by the corresponding value:

```solidity
contract Bank {
    mapping(address => uint256) public balanceOf;
    ...
    function withdraw(uint256 amount) public {
        require(balanceOf[msg.sender] >= amount);
        msg.sender.call.value(amount)();
        balanceOf[msg.sender] -= amount;
    }
}
```

The problem with this implementation is "sending money before bookkeeping." In Ethereum, the caller of a contract can be another smart contract, and the receiving contract's fallback function is called during the transfer. If the fallback function calls the withdraw function again, since `balanceOf` has not yet been reduced, the require condition is still met, allowing another withdrawal. Note that the fallback function needs to limit the number of re-entries; otherwise, infinite recursive calling would lead to running out of gas. Assuming the attack contract has a deposit of 1 ether, the following can be implemented to withdraw 2 ether:

```solidity
contract Hacker {
    bool status = false;
    Bank b;

    constructor(address addr) public {
        b = Bank(addr);
    }

    function hack() public {
        b.withdraw(1 ether);
    }

    function() public payable {
        if (!status) {
            status = true;
            b.withdraw(1 ether);
        }
    }
}
```

Additionally, there are several important notes:

- When the target contract uses call to send Ether, all remaining gas is provided by default. Changing the call operation to invoke the withdrawer's contract can also implement the attack. However, if transfer or send is used to send Ether, only 2300 gas is available for the attack contract, which is insufficient to complete a re-entrancy attack.
- Before executing a re-entrancy attack, you need to confirm that the target contract has enough Ether to transfer to us multiple times. If the target contract does not have a payable fallback function, you need to create a new contract and force-send funds through `selfdestruct`.
- In the above fallback implementation, `status` is modified first before re-entering. If reversed, it would still result in infinite recursive calling, which is consistent with the principle of the re-entrancy vulnerability.

Re-entrancy vulnerabilities are closely related to integer underflow vulnerabilities. After the above attack, the attack contract's deposit changes from 1 ether to -1 ether. Note that since the deposit is stored as uint256, the negative number is actually stored as an extremely large positive number. Subsequently, the attack contract can continue to use this large balance amount.

## Challenges

### QWB 2019
- Challenge Name: babybank

### N1CTF 2019
- Challenge Name: h4ck

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
