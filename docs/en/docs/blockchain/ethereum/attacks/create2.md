# CREATE2

`CREATE2` is a new opcode introduced in Ethereum's "Constantinople" hard fork upgrade. Unlike `CREATE`, it uses a new method to calculate contract addresses, making the generated contract addresses more controllable. Through `CREATE2`, many interesting techniques can be derived. In CTF competitions, the most common use is leveraging this controllability to deploy contracts with completely different bytecode at the same address sequentially.

## Principle

### CREATE

If an external account or a contract account using the `CREATE` opcode creates a contract, it is easy to determine the address of the created contract. Each account has an associated `nonce`: for external accounts, the `nonce` increments by 1 with each transaction sent; for contract accounts, the `nonce` increments by 1 with each contract created. The new contract's address is calculated from the sender account address of the contract creation transaction and its `nonce` value, with the specific formula as follows:

```python
keccak256(rlp.encode(address, nonce))[12:]
```

### CREATE2

Unlike the original `CREATE` opcode, in the method of calculating the contract address, `CREATE2` no longer depends on the account's `nonce`. Instead, it hashes the following parameters to derive the new address:

- The address of the contract creator (`address`)
- A salt value passed as a parameter (`salt`)
- The contract creation code (`init_code`)

The specific calculation formula is as follows:

```python
keccak256(0xff ++ address ++ salt ++ keccak256(init_code))[12:]
```

An important detail to note is that the last parameter needed to calculate the contract address is not the contract code itself, but its creation code. This code is used to create the contract, and after the contract is created, it returns the runtime bytecode.

This means that if we control the contract's creation code and keep it unchanged, then control the runtime bytecode returned by the contract constructor, we can easily deploy completely different contracts at the same address repeatedly. In fact, this characteristic of `CREATE2` that allows contracts to be re-deployed after deployment poses potential security concerns, which has sparked [discussion](https://ethereum-magicians.org/t/potential-security-implications-of-create2-eip-1014/2614).

In CTF competitions, this characteristic is often used as a technique to bypass different validations by deploying different contracts at the same address.

## Example

Using the PoC provided in the writeup for Creativity from the 2019 Balsn CTF as an example to explain the clever use of `CREATE2`:

```solidity
pragma solidity ^0.5.10;

contract Deployer {
    bytes public deployBytecode;
    address public deployedAddr;

    function deploy(bytes memory code) public {
        deployBytecode = code;
        address a;
        // Compile Dumper to get this bytecode
        bytes memory dumperBytecode = hex'6080604052348015600f57600080fd5b50600033905060608173ffffffffffffffffffffffffffffffffffffffff166331d191666040518163ffffffff1660e01b815260040160006040518083038186803b158015605c57600080fd5b505afa158015606f573d6000803e3d6000fd5b505050506040513d6000823e3d601f19601f820116820180604052506020811015609857600080fd5b81019080805164010000000081111560af57600080fd5b8281019050602081018481111560c457600080fd5b815185600182028301116401000000008211171560e057600080fd5b50509291905050509050805160208201f3fe';
        assembly {
            a := create2(callvalue, add(0x20, dumperBytecode), mload(dumperBytecode), 0x9453)
        }
        deployedAddr = a;
    }
}

contract Dumper {
    constructor() public {
        Deployer dp = Deployer(msg.sender);
        bytes memory bytecode = dp.deployBytecode();
        assembly {
            return (add(bytecode, 0x20), mload(bytecode))
        }
    }
}
```

Every time we use the `deploy(code)` function to deploy a pre-constructed contract, since the actual `init_code` is always the same `dumperBytecode`, combined with a fixed contract address and `salt`, the contract deployed through `deploy(code)` will always end up at the same address. The loaded contract, during constructor execution, jumps to the `code` passed in the function call, so no matter what contract we deploy with the `deploy(code)` function, it will always be deployed to the same address.

Knowing that the `Deployer` contract address is 0x99Ed0b4646a5F4Ee0877B8341E9629e4BF30c281, we can calculate the deployed contract's address as 0x4315DBef1aC19251d54b075d29Bcc4E81F1e3C73:

```solidity
function getAddress(address addr, bytes memory bytecode, uint salt) public view returns (address) {
    bytes32 hash = keccak256(
        abi.encodePacked(
            bytes1(0xff),
            addr,
            salt,
            keccak256(bytecode)
        )
    );

    // NOTE: cast last 20 bytes of hash to address
    return address(uint160(uint256(hash)));
}
```

Using this contract, we successfully deployed two different contracts at the same address sequentially:

![First deployment](./figure/create2_0.png)
![Second deployment](./figure/create2_1.png)


## Challenges

### Balsn 2019
- Challenge Name: Creativity

### QWB 2020
- Challenge Name: EasyAssembly

## References

- [EIP-1014: Skinny CREATE2](https://eips.ethereum.org/EIPS/eip-1014)
- [Getting the Most out of CREATE2](https://ethfans.org/posts/getting-the-most-out-of-create2)
- [Balsn CTF 2019 - Creativity](https://x9453.github.io/2020/01/04/Balsn-CTF-2019-Creativity/)
