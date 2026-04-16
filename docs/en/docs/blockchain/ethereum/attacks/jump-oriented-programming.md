# Jump Oriented Programming

## Principle

Similar to ROP in pwn, the EVM also has JOP (Jump Oriented Programming). The concept of JOP is similar to ROP: chaining together small code snippets (gadgets) to achieve a certain objective.

The three bytecodes involved in JOP are:

- 0x56 JUMP
- 0x57 JUMPI
- 0x5B JUMPDEST

In the EVM, both the unconditional jump `JUMP` and the conditional jump `JUMPI` must target a `JUMPDEST`, which differs from ROP where any return address can be chosen.

Another important note is that although the EVM uses variable-length instructions, it does not allow jumping to the middle of an instruction like ROP does. For example, in x86-64, `pop r15` is `A_`, and during ROP, landing directly on the second byte can be used as `pop rdi`. In the EVM, the `0x5B` within `PUSH1 0x5B` cannot be used as a `JUMPDEST`.

Contracts that typically require JOP usually have hidden backdoors mixed with inline assembly, and manual reverse engineering is needed to identify two things:

1. A starting point where the control flow is reachable and the jump address can be controlled
2. Various gadgets that start with `JUMPDEST`, implement some special functionality, and then end with a `JUMP` instruction

The functionality that gadgets need to implement varies depending on the challenge requirements or examination points. For example, to implement an external contract call, you would need to arrange various offsets, gas, and other data on the stack in order. At the end of JOP, a `JUMPDEST; STOP` is needed as a landing point for termination; otherwise, an execution error would cause the transaction to be rolled back.

In addition to the three bytecodes above, EIP-2315 also proposed three bytecodes: `BEGINSUB`, `RETURNSUB`, and `JUMPSUB`. Among them, `JUMPSUB` is similar to `JUMP`, except the jump destination must be `BEGINSUB`; while `RETURNSUB` is equivalent to `ret` in ROP, with no restrictions on the target address. EIP-2315 was once included in the upgrade list before the Berlin upgrade but was later removed. It is currently still in the draft stage.

## Challenges

### RealWorldCTF Final 2018
- Challenge Name: Acoraida Monica

### RealWorldCTF 3rd 2021
- Challenge Name: Re: Montagy

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
