# Short Address Attack

## Principle

The short address attack exploits the EVM's behavior of automatically padding zeros on the right when parameter lengths are insufficient. By removing the trailing zeros from a wallet address, the transfer amount is shifted left and amplified.

## Example

```solidity
pragma solidity ^0.4.10;

contract Coin {
    address owner;
    mapping (address => uint256) public balances;

    modifier OwnerOnly() { require(msg.sender == owner); _; }

    function ICoin() { owner = msg.sender; }
    function approve(address _to, uint256 _amount) OwnerOnly { balances[_to] += _amount; }
    function transfer(address _to, uint256 _amount) {
        require(balances[msg.sender] > _amount);
        balances[msg.sender] -= _amount;
        balances[_to] += _amount;
    }
}
```

A contract Coin that implements token functionality. When Account A transfers tokens to Account B, the `transfer()` function is called. For example, Account A (0x14723a09acff6d2a60dcdf7aa4aff308fddc160c) transfers 8 Coins to Account B (0x4b0897b0513fdc7c541b6d9d7e929c4e5364d2db). The `msg.data` would be:

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) function signature
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d2db  -> Account B address (padded with leading 0s to 32 bytes)
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8 (padded with leading 0s to 32 bytes)
```

So how does the short address attack work? The attacker finds an account address ending with `00`, say 0x4b0897b0513fdc7c541b6d9d7e929c4e5364d200. Under normal circumstances, the full `msg.data` for the call should be:

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) function signature
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d200  -> Account B address (note the trailing 00)
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8 (padded with leading 0s to 32 bytes)
```

But if we strip the `00` from Account B's address and don't pass it, meaning we pass 1 fewer byte, making it 4+31+32:

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) function signature
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d2  -> Account B address (31 bytes)
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8 (padded with leading 0s to 32 bytes)
```

When the above data enters the EVM for processing, after encoding alignment, `00` is padded, becoming:

```
0xa9059cbb
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d200
0000000000000000000000000000000000000000000000000000000000000800
```

In other words, the maliciously crafted `msg.data`, through the EVM's zero-padding parsing operation, causes the original 0x8 = 8 to become 0x800 = 2048.

The EVM's behavior of padding malformed bytes in `msg.data` is essentially the principle behind the short address attack.

## Challenges

There are currently no challenges for this, as it has basically been patched. However, it can be reproduced successfully, but not through Remix because the client checks the address length; nor through sendTransaction() because `web3` has also added protection.

However, you can use **geth** to set up a private chain and use sendRawTransaction() to send transactions for reproduction. You can try this on your own.

!!! note
    Note: Currently, this issue is mainly avoided by having clients proactively check address lengths. Additionally, protection has been added at the `web3` layer for parameter format validation. Although it can still be reproduced at the EVM level, there are essentially no issues in real-world application scenarios.
