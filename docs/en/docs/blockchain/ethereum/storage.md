# Ethereum Storage

## Slots

Ethereum data storage assigns a computable storage location for each piece of data in a contract, stored in a super array with a capacity of 2^256. Each element in the array is called a slot, with an initial value of 0. Although the upper limit of the array capacity is very high, the actual storage is sparse — only non-zero (non-empty) data is actually written to storage.

```
# Slot-based array storage
----------------------------------
|               0                |     # slot 0
----------------------------------
|               1                |     # slot 1
----------------------------------
|               2                |     # slot 2
----------------------------------
|              ...               |     # ...
----------------------------------
|              ...               |     # Each slot is 32 bytes
----------------------------------
|              ...               |     # ...
----------------------------------
|            2^256-1             |     # slot 2^256-1
----------------------------------
```

When the data length is known, its specific storage location is determined at compile time. For types with indeterminate lengths (such as dynamic arrays and mappings), the storage location is calculated according to certain rules. The following is a detailed analysis of the storage model for different types of variables.

## Value Types

All types other than mappings and dynamic arrays have known data lengths, such as fixed-length integers (`int`/`uint`/...), addresses (`address`), fixed-point numbers (`fixed`/`ufixed`/...), and fixed-length byte arrays (`bytes1`-`bytes32`). At compile time, they are strictly placed in storage continuously starting from position 0, according to the field ordering. If possible, multiple variables smaller than 32 bytes will be packed into a single slot. When a piece of data exceeds 32 bytes, it requires multiple consecutive slots (`data.length / 32`). The rules are as follows:

- The first item in a storage slot is stored with lower-order alignment (i.e., right-aligned).
- Basic types use only the bytes necessary to store them.
- If the remaining space in a storage slot is insufficient to store a basic type, it is moved to the next storage slot.
- Structs and array data always occupy a whole new slot (but individual items within structs or arrays are packed according to these rules).

For example, the following contract:
```solidity
pragma solidity ^0.4.0;

contract C {
    address a;      // 0
    uint8 b;        // 0
    uint256 c;      // 1
    bytes24 d;      // 2
}
```

Its storage layout is as follows:

```
-----------------------------------------------------
| unused (11) | b (1) |            a (20)           | <- slot 0
-----------------------------------------------------
|                       c (32)                      | <- slot 1
-----------------------------------------------------
| unused (8) |                d (24)                | <- slot 2
-----------------------------------------------------
```

## Mappings

For mapping type variables like `mapping(address => uint) a;`, they cannot simply be stored sequentially like value types. For mappings, it occupies a slot at position `p` according to the rules mentioned in the previous section, but this slot is not actually used. The value corresponding to key `k` in the mapping is located at `keccak256(k . p)`, where `.` is the concatenation operator. If the value is also a non-basic type, then `keccak256(k . p)` is used as an offset to find the specific location.

For example, the following contract:
```solidity
pragma solidity ^0.4.0;

contract C {
    mapping(address => uint) a;      // 0
    uint256 b;                       // 1
}
```

Its storage layout is as follows:

```
-----------------------------------------------------
|                    reserved (a)                   | <- slot 0
-----------------------------------------------------
|                      b (32)                       | <- slot 1
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------
|                     a[addr] (32)                  | <- slot `keccak256(addr . 0)`
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------
```

## Dynamic Arrays

For dynamic arrays like `uint[] b;`, they similarly occupy a slot at position `p`, which stores the array's length, while the actual starting point of the array is at `keccak256(p)` (byte arrays and strings are an exception here, see below).

For example, the following contract:
```solidity
pragma solidity ^0.4.0;

contract C {
    uint256 a;      // 0
    uint[] b;       // 1
    uint256 c;      // 2
}
```

Its storage layout is as follows:

```
-----------------------------------------------------
|                      a (32)                       | <- slot 0
-----------------------------------------------------
|                    b.length (32)                  | <- slot 1
-----------------------------------------------------
|                      c (32)                       | <- slot 2
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------
|                      b[0] (32)                    | <- slot `keccak256(1)`
-----------------------------------------------------
|                      b[1] (32)                    | <- slot `keccak256(1) + 1`
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------
```

## Byte Arrays and Strings

If the data of `bytes` and `string` is short, their length is stored together with the data in the same slot. Specifically: if the data length is less than or equal to 31 bytes, it is stored in the higher-order bytes (left-aligned), and the lowest-order byte stores `length * 2`. If the data length exceeds 31 bytes, the main slot stores `length * 2 + 1`, and the data is stored normally at `keccak256(slot)`.

## Visibility

Since all information on Ethereum is public, even if a variable is declared as `private`, we can still read the variable's actual value.

Using the `web3.eth.getStorageAt()` method provided by web3, you can read the storage content at a specified position of an Ethereum address. So as long as you calculate the slot position corresponding to a variable, you can obtain the variable's actual value by calling this function.

Call:

```javascript
// web3.eth.getStorageAt(address, position [, defaultBlock] [, callback])
web3.eth.getStorageAt("0x407d73d8a49eeb85d32cf465507dd71d507100c1", 0)
.then(console.log);
> "0x033456732123ffff2342342dd12342434324234234fd234fd23fd4f23d4234"
```

Parameters:

- `address`: String - The address to read from
- `position`: Number - The index number in storage
- `defaultBlock`: Number|String - Optional, use this parameter to override the web3.eth.defaultBlock property value
- `callback`: Function - Optional callback function, with the first parameter being the error object and the second parameter being the result.

## Example

Using the Bank challenge from Balsn CTF 2019 as an example to explain Ethereum's storage layout in more detail. The variable and structure definitions in the challenge are as follows:

```solidity
contract Bank {
    address public owner;
    uint randomNumber = 0;
    
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
}
```

The contract's variables are stored in slots 0 through 3 according to the following layout:

```solidity
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

For the structs `SafeBox` and `FailedAttempt`, the storage layout occupied by each struct is as follows:

```
# SafeBox
-----------------------------------------------------
| unused (11) | hash (12) | callback (8) | done (1) |
-----------------------------------------------------
|                     value (32)                    |
-----------------------------------------------------

# FailedAttempt
-----------------------------------------------------
|                      idx (32)                     |
-----------------------------------------------------
|                     time (32)                     |
-----------------------------------------------------
|          origin (20)         |   triedPass (12)   |
-----------------------------------------------------
```

For the array `safeboxes`, the starting point of the array elements is at `keccak256(2)`, with each element occupying 2 slots. For the mapping `failedLogs`, you first need to compute `keccak256(addr . 3)` to get the position of the array corresponding to a specific address `addr`. That position stores the array's length, while the actual starting point of the array is at `keccak256(keccak256(addr . 3))`, with each element occupying 3 slots.

The following code can be used to conveniently calculate the actual position of elements in arrays and mappings:

```solidity
function read_slot(uint k) public view returns (bytes32 res) {
    assembly { res := sload(k) }
}

function cal_addr(uint k, uint p) public pure returns(bytes32 res) {
    res = keccak256(abi.encodePacked(k, p));
}

function cal_addr(uint p) public pure returns(bytes32 res) {
    res = keccak256(abi.encodePacked(p));
}
```

## Challenges

Attacks related to Ethereum storage generally fall into two categories:

- Exploiting the characteristic that all storage on Ethereum is essentially public, to arbitrarily read variables declared as `private`.
- Combined with arbitrary write vulnerabilities, overwriting storage at specific positions on Ethereum.

### XCTF_final 2019
- Challenge Name: Happy_DOuble_Eleven

### Balsn 2019
- Challenge Name: Bank

## References

- [Ethereum Smart Contract OPCODE Reverse Engineering Basics - Global Variable Storage Model](https://paper.seebug.org/640/#_3)
- [Solidity Documentation - Layout of State Variables in Storage](https://solidity-cn.readthedocs.io/zh/develop/miscellaneous.html)
- [web3.js - Ethereum JavaScript API](https://web3js.readthedocs.io/en/v1.3.4/web3-eth.html)
- [Balsn CTF 2019 - Bank](https://x9453.github.io/2020/01/16/Balsn-CTF-2019-Bank/)
