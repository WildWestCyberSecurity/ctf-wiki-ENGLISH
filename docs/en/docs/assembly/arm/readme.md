# ARM

Introduction to ARM fundamentals.



## 1. ARM Assembly Basics

### 1. LDMIA R0 , {R1,R2,R3,R4}

LDM is a multi-register "load from memory" instruction.
IA means that R0 is incremented by 1 word after each LDM instruction completes.
The final result is R1 = [R0], R2 = [R0+#4], R3 = [R0+#8], R4 = [R0+#0xC].

### 2. Stack Addressing (FA, EA, FD, ED)

STMFD SP! , {R1-R7,LR} @ Push R1~R7 and LR onto the stack
LDMFD SP! , {R1-R7,LR} @ Pop R1~R7 and LR from the stack

### 3. Block Copy Addressing

LDM and STM are instruction prefixes that indicate multi-register addressing, with instruction suffixes (IA, DA, IB, DB).
LDMIA R0!, {R1-R3} @ Sequentially load 3 words from the memory address pointed to by R0 into registers R1, R2, R3
STMIA R0!, {R1-R3} @ Sequentially store the contents of R1, R2, R3 into the memory pointed to by R0.

### 4. Relative Addressing

```
Uses the current value of the program counter PC as the base address, and the
label-marked position as the offset. The two are added together to obtain the
effective address.


BL NEXT
    ...        
NEXT:
    ...
```

## 2. Instruction Set

### 1. Due to the rapid evolution of ARM chips, there are many instruction sets. The most commonly used are the ARM instruction set and the Thumb instruction set.



### 2. Branch Instructions

ARM implements two types of branching: one uses branch instructions directly, and the other assigns a value directly to the PC register.

#### 1. B Branch Instruction

```
Format: B{cond} label    
Branch directly, e.g., `BNE LABEL`
```

#### 2. BL Branch with Link Instruction

```
Format: BL{cond} label    
When executing a BL instruction, if the condition is met, the address of the next instruction
after the current one is first assigned to the R14 register (LR), then execution jumps to the
address marked by label. This is commonly used in procedure calls, and the procedure returns
via `MOV PC, LR` after completion.
```

#### 3. BX Branch with State Switch Instruction

```
Format: BX{cond}Rm   
When executing a BX instruction, if the condition is met, it checks whether bit[0] of the Rm register is 1. If it is 1, the T flag bit in the CPSR register is automatically set to 1 upon branching, and the instructions at the target location are interpreted as Thumb instructions. Conversely, if bit[0] of the Rm register is 0, the T flag bit in the CPSR register is cleared, and the instructions at the target location are interpreted as ARM instructions.
```

As shown below:

```
ADR R0, thumbcode + 1
BX R0       @ Jump to thumbcode, and the processor runs in Thumb mode
thumbcode:
.code 16
```



#### 4. BLX Branch with Link and State Switch Instruction

```
Format: BLX{cond}Rm
The BLX instruction combines the functionality of BL and BX. In addition to the BX functionality, it also saves the return address to R14 (LR).
```

### 3. Memory Access Instructions

Memory access instruction operations include loading data from memory, storing data to memory, and exchanging data between registers and memory.

#### `LDR`

Loads data from memory into a register.

Instruction examples:

```
LDRH R0, [R1]         ; Load the halfword data at memory address R1 into register R0, and clear the upper 16 bits of R0.
LDRH R0, [R1, #8]     ; Load the halfword data at memory address R1+8 into register R0, and clear the upper 16 bits of R0.
LDRH R0, [R1, R2]     ; Load the halfword data at memory address R1+R2 into register R0, and clear the upper 16 bits of R0.
```


#### `STR`

STR is used to store data to a specified address. The format is as follows:
STR{type}{cond}Rd,label
STRD{cond}Rd,Rd2,label
Usage:
`STR R0,[R2,#04]` stores the value of R0 to the address R2+4.

#### `LDM`

```
LDM{addr_mode}{cond}Rn{!}reglist
```

This instruction batch-loads data from the stack in memory into registers, i.e., a pop operation.

> Note: ! is an optional suffix. If ! is present, the final address is written back to the Rn register.

#### `STM`

STM stores the data from a register list to the specified memory address units. The format is as follows:

```
STM{addr_mod}{cond}Rn{!}reglist
```

#### `PUSH&&POP`

The format is as follows:
PUSH{cond}reglist
POP{cond}reglist
Stack operation instructions.

```
PUSH {r0,r4-r7}
POP {r0,r4-r7}
```



#### `SWP`

### Data Exchange Between Registers.

The format is `SWP{B}{cond}Rd,Rm,[Rn]`
B is an optional byte specifier. If B is present, bytes are exchanged; otherwise, words are exchanged.
Rd is the register for temporary storage, Rm is the value `to replace with`.
Rn is the data address `to be replaced`.

### References

[ARM Instruction Study](https://ring3.xyz/2017/03/05/[%E9%80%86%E5%90%91%E7%AF%87]arm%E6%8C%87%E4%BB%A4%E5%AD%A6%E4%B9%A0/)

[Common ARM Instructions](http://www.51-arm.com/upload/ARM_%E6%8C%87%E4%BB%A4.pdf)

[arm-opcode-map](http://imrannazar.com/ARM-Opcode-Map)
