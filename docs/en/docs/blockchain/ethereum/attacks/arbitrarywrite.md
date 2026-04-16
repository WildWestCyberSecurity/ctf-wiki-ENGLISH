# Arbitrary Writing

## Principle

The arbitrary Storage write vulnerability in dynamic arrays. According to the [official documentation](https://docs.soliditylang.org/en/v0.8.1/internals/layout_in_storage.html#), the following summary can be made:

- In the EVM, there are three places where variables can be stored: Memory, Stack, and Storage. Memory and Stack are temporary storage spaces generated during execution, mainly responsible for runtime data storage. Storage is permanently stored on the blockchain.
    + Memory: Memory with a lifetime spanning only the entire method execution period. It is reclaimed after the function call. Since it only holds temporary variables, the Gas overhead is very small.
    + Storage: Permanently stored on the blockchain. Since it permanently saves contract state variables, the Gas overhead is the largest.
    + Stack: Stores some local value type variables. Nearly free to use, but has quantity limitations.

- The EVM maintains a huge **key-value** storage structure for each smart contract for persistent data storage, which we call Storage. All types of variables except map mappings and dynamic arrays are arranged consecutively in Storage starting from slot 0. There are a total of 2^256 slots, and each slot can store 32 bytes of data. The Storage structure is determined when the contract is created — it depends on the state variables declared by the contract, but the content can be changed through Transactions.
- Storage variables can be roughly divided into 4 types: fixed-length variables, structs, map mappings, and dynamic arrays. If multiple variables occupy less than 32 bytes, according to the tight packing principle, they will be packed into a single slot as much as possible. The specific rules are as follows:
    + In a slot, data is stored with lower-order alignment, i.e., big-endian
    + Basic type variables are stored using only the bytes they actually need
    + If a basic type variable cannot fit into the remaining space of a slot, it will be placed in the next slot
    + Maps and dynamic arrays always use a brand new slot and occupy the entire slot, but for each variable within them, the above rules still apply

### Slot Calculation Rules

First, let's analyze the storage and access patterns of various object structures in the EVM.

#### Fixed-length Variables and Structs

Fixed-length variables in Solidity have their lengths constrained at definition time. For example, fixed-length integers (uint, uint8), address constants (address), fixed-length byte arrays (bytes1-32), etc. These types of variables are stored sequentially in Storage, packed into 32-byte blocks as much as possible.

Solidity structs do not have a special storage model and can be analyzed using the same rules as fixed-length variables in Storage.

#### Map Mappings

In Solidity, map keys are not stored — only the values corresponding to the keys are stored. Values are found through hash indexing of the keys. Let $slotM$ represent the slot position where the map is declared, $key$ represent the key, $value$ represent the value corresponding to $key$, and $slotV$ represent the storage position of $value$. Then:

- $slotV = keccak256(key|slotM)$

- $value = sload(slotV)$

#### Dynamic Arrays

Let $slotA$ represent the position where the dynamic array is declared, $length$ represent the length of the dynamic array, $slotV$ represent the storage position of the dynamic array data, $value$ represent the value of a specific element in the dynamic array, and $index$ represent the index corresponding to $value$. Then:

- $length = sload(slotA)$

- $slotV = keccak256(slotA) + index$

- $value = sload(slotV)$

The length of a dynamic array cannot be known at compile time, so there is no way to pre-allocate storage space. Therefore, Solidity stores the length of the dynamic array at position $slotA$.

!!! note
    Note: The actual data of a dynamic array is stored in a contiguous storage area after the keccak256 hash calculation. This is different from Map mappings.

### Vulnerability Introduction

In the design philosophy of the Ethereum EVM, all Storage variables share a single storage space of 2^256*32 bytes, without separate storage area divisions.

Even though the Storage space is very large, it is finite. When the length of a dynamic array is very large, considering the extreme case where the length reaches 2^256, it becomes possible to perform read/write operations on any Storage variable, which is very dangerous.

## Example

### Source

```solidity
pragma solidity ^0.4.24;

contract ArrayTest  {

    address public owner;
    bool public contact;
    bytes32[] public codex;
    
    constructor() public {
        owner = msg.sender;
    }

    function record(bytes32 _content) public {
        codex.push(_content);
    }

    function retract() public {
        codex.length--;
    }

    function revise(uint i, bytes32 _content) public {
        codex[i] = _content;
    }
}
```

How can the attacker become the owner here? The initial owner is 0x73048cec9010e92c298b016966bde1cc47299df5.

### Analysis

- The slot of array codex is 1, which is also where the array length is stored. The actual content of codex is stored starting at the position keccak256(bytes32(1)).

!!! info
    Keccak256 uses tight packing, meaning parameters are not padded and multiple parameters are concatenated directly. That's why we use keccak256(bytes32(1)).

- Now that we know the actual storage slot of codex, we can summarize the method for calculating the storage position of variables in a dynamic array as: array[index] == sload(keccak256(slot(array)) + index).

- Since there are a total of 2^256 slots, to modify slot 0, assuming the actual slot of codex is x (for this challenge, the array's slot is 1, so x=keccak256(bytes32(1))), then when we modify codex[y] (y=2^256-x+0), we can modify slot 0 and thus modify the owner.
    - Calculate the codex position as slot 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6
    - So y = 2^256 - 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6 + 0
    - i.e., y = 35707666377435648211887908874984608119992236509074197713628505308453184860938

- We can see that y is very large. To modify codex[y], we need to satisfy y < codex.length. At this point codex.length = 0, but we can trigger an underflow through retract() to make the length wrap around, and then we can manipulate codex[y].
- As calculated above, codex[35707666377435648211887908874984608119992236509074197713628505308453184860938] corresponds to slot 0, which stores both contact and owner. We just need to replace owner with the attacker's address. Assuming the attacker's address is 0x88d3052d12527f1fbe3a6e1444ea72c4ddb396c2, it would be as follows:

```
contract.revise('35707666377435648211887908874984608119992236509074197713628505308453184860938','0x00000000000000000000000088d3052d12527f1fbe3a6e1444ea72c4ddb396c2')
```

## Challenges

### XCTF_final 2019
- Challenge Name: Happy_DOuble_Eleven

### Balsn 2019
- Challenge Name: Bank

### 1st Diaoyucheng Cup 2020
- Challenge Name: StrictMathematician

### RCTF 2020
- Challenge Name: roiscoin

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
