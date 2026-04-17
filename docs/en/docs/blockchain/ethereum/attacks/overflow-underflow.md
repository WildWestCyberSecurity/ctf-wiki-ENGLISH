# Integer Overflow and Underflow

## Principle

The EVM has two integer types: `int` and `uint`, corresponding to signed and unsigned cases respectively. After `int` or `uint`, a multiple of 8 can follow to indicate the bit width of the integer, such as the 8-bit `uint8`. The maximum bit width is 256 bits, and `int` and `uint` are aliases for `int256` and `uint256` respectively. `uint` is generally used more often.

When an integer exceeds the upper or lower limit of its bit width, a silent modulo operation is performed. Typically, we want fees to overflow upward to become smaller, or deposits to underflow downward to become larger. Integer overflow vulnerabilities can be defended against using the SafeMath library, which rolls back the transaction when an overflow occurs.

## Example

Using the Token sale challenge from Capture The Ether as an example:

```solidity
pragma solidity ^0.4.21;

contract TokenSaleChallenge {
    mapping(address => uint256) public balanceOf;
    uint256 constant PRICE_PER_TOKEN = 1 ether;

    function TokenSaleChallenge(address _player) public payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance < 1 ether;
    }

    function buy(uint256 numTokens) public payable {
        require(msg.value == numTokens * PRICE_PER_TOKEN);

        balanceOf[msg.sender] += numTokens;
    }

    function sell(uint256 numTokens) public {
        require(balanceOf[msg.sender] >= numTokens);

        balanceOf[msg.sender] -= numTokens;
        msg.sender.transfer(numTokens * PRICE_PER_TOKEN);
    }
}
```

In this challenge, purchasing a single token costs 1 ether, i.e., `msg.value == numTokens * PRICE_PER_TOKEN`. In the EVM, currency is denominated in wei, and 1 ether is actually $10 ^ { 18 }$ wei, i.e., 0xde0b6b3a7640000 wei. If we make `numTokens` large enough, the product can overflow. For example, if we purchase $2 ^ { 256 } // 10 ^ { 18 } + 1$ tokens, after multiplying by $10 ^ { 18 }$, an overflow occurs, and the final cost is only about 0.4 ether for a large number of tokens. Then we sell some of the purchased tokens to complete the challenge requirements.

An example of integer underflow is the subtraction operation. Suppose a contract implements the following functionality:

```solidity
contract Bank {
    mapping(address => uint256) public balanceOf;
    ...
    function withdraw(uint256 amount) public {
        require(balanceOf[msg.sender] - amount >= 0);
        balanceOf[msg.sender] -= amount;
        msg.sender.send.value(amount)();
    }
}
```

At first glance, it seems fine. In reality, the result of `balanceOf[msg.sender]-amount` on the require line, as an unsigned integer, is always greater than or equal to 0, allowing us to withdraw arbitrarily. The correct way to write it is `require(balanceOf[msg.sender] >= amount)`.

Another example of integer underflow is related to re-entrancy attacks, such as selling an item with a holding count of 1 twice, or withdrawing 1 ether deposit twice, resulting in a negative number that, when stored as uint, becomes an extremely large positive number.

## Challenges

The vast majority of re-entrancy attack challenges involve underflow. Refer to the re-entrancy attack section. Challenges that don't involve re-entrancy attacks are relatively rare. The following challenges can be referenced:

### ByteCTF 2019
- Challenge Name: hf
- Challenge Name: bet

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
