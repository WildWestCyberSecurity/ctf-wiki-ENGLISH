# RISC-V

Introduction to RISC-V fundamentals.

## 0x01 Architecture Characteristics

RISC-V is an emerging open instruction set architecture (ISA) with a high degree of flexibility and extensibility. Its core design philosophy is modularity, allowing developers to select different extension modules to build processors based on actual requirements. For example, the most basic instruction sets include RV32I and RV64I, while optional extensions cover multiplication and division (M), atomic operations (A), floating-point arithmetic (F, D), compressed instructions (C), and vector instruction sets (V) suitable for high-performance computing.

The architecture supports multiple bit widths, including 32-bit (RV32), 64-bit (RV64), and a future reserved 128-bit (RV128) mode, thus meeting diverse application scenarios from embedded systems to high-performance servers. RISC-V also explicitly distinguishes processor privilege levels, defining User mode (U-mode), Supervisor mode (S-mode), and Machine mode (M-mode), providing solid support for operating system execution and security mechanisms.

The following table lists some common instruction set extensions and their brief descriptions:

| Name             | Description                                 |
|------------------|--------------------------------------|
| RV32I / RV64I     | Base integer instruction set, the core for building minimal systems   |
| M Extension           | Supports integer multiplication and division operations                   |
| A Extension           | Provides atomic operation capabilities, suitable for multi-core systems     |
| F / D Extension       | Supports floating-point arithmetic: F for single precision, D for double precision |
| C Extension           | 16-bit compressed instructions, improving code density           |
| V Extension           | Vector processing capabilities for data-parallel computation       |
| Zicsr / Zifencei | Control and status register operations and instruction synchronization control     |

## 0x02 Registers

The RISC-V processor architecture defines a set of general-purpose registers and Control and Status Registers (CSRs) for data processing, control flow, and state information storage. Below is a brief introduction to these two categories of registers.

### (1) General-Purpose Registers

RISC-V has a total of **32 general-purpose registers**, numbered `x0` through `x31`, each being 32-bit (RV32) or 64-bit (RV64). These registers also have corresponding ABI names commonly used in assembly language programming.

| Register Number | ABI Name | Description            |
|------------|----------|---------------------|
| x0         | zero     | Read-only register, always 0 |
| x1         | ra       | Return address register      |
| x2         | sp       | Stack pointer               |
| x3         | gp       | Global pointer             |
| x4         | tp       | Thread pointer             |
| x5-x7      | t0-t2    | Temporary registers           |
| x8         | s0/fp    | Saved register / Frame pointer    |
| x9         | s1       | Saved register           |
| x10-x17    | a0-a7    | Function arguments / Return values      |
| x18-x27    | s2-s11   | Saved registers           |
| x28-x31    | t3-t6    | Temporary registers           |

Notes:
- `x0` is a special read-only register that always returns `0`; writes to it have no effect.
- Temporary registers (`t0`~`t6`) are not guaranteed to be preserved across function calls.
- Saved registers (`s0`~`s11`) must be saved and restored across function calls.

### (2) CSR Registers

CSRs (Control and Status Registers) are important registers used for controlling, monitoring status, and configuring processor operation. In RISC-V, the CSR address space is 12 bits, supporting a total of 4096 CSR registers.

Common CSR registers include:

| Name        | Address      | Description                               |
|-------------|-----------|----------------------------------------|
| `mstatus`   | 0x300     | Machine-level status register, stores global interrupt enable, privilege level, and other information |
| `misa`      | 0x301     | Indicates the currently supported instruction set architecture               |
| `mie`       | 0x304     | Interrupt enable register                         |
| `mtvec`     | 0x305     | Interrupt vector table address                         |
| `mepc`      | 0x341     | Program counter save register when an exception occurs       |
| `mcause`    | 0x342     | Exception/interrupt cause register                    |
| `mtval`     | 0x343     | Faulting address information                           |
| `mcycle`    | 0xB00     | Cycle counter in machine mode                 |

Notes:
- CSRs can be read and written using instructions belonging to the `zicsr` extension instruction set, such as `csrrw`, `csrrs`, `csrrc`, etc.
- CSRs are classified by privilege level into User-level (u*), Supervisor-level (s*), Machine-level (m*), etc.
- In simplified implementations, supporting only Machine-mode (M-mode) CSRs is a common scenario.

### (3) Floating-Point Registers

When the RISC-V architecture supports floating-point extensions (such as the `F` and `D` extensions), it provides a dedicated set of floating-point registers for floating-point computation and data storage.

RISC-V defines **32 floating-point registers**, numbered `f0` through `f31`, each typically being 32-bit (single precision, RV32F) or 64-bit (double precision, RV64D), depending on the implemented floating-point extension.

| Register Number | ABI Name | Description            |
|------------|----------|---------------------|
| f0-f7      | ft0-ft7  | Temporary registers          |
| f8-f9      | fs0-fs1  | Saved registers          |
| f10-f17    | fa0-fa7  | Function arguments / Return values     |
| f18-f27    | fs2-fs11 | Saved registers          |
| f28-f31    | ft8-ft11 | Temporary registers          |

Notes:
- The ABI naming of floating-point registers is similar to integer registers, but with the prefix `f`.
- According to the ABI convention, function arguments are passed via `fa0`~`fa7`, and return values typically use `fa0`.
- `ft*` are caller-saved, and `fs*` are callee-saved.
- Using floating-point registers requires the processor to support the corresponding floating-point extension instruction set, such as `F` (single precision) or `D` (double precision).

## 0x03 Common Instructions

### (1) Common Integer Instructions (R-Type)

R-type instructions are used for arithmetic or logical operations between registers, format: `rd = rs1 op rs2`.

| Instruction   | Meaning            | Example              | Description                      |
|--------|-----------------|-------------------|---------------------------|
| `add`  | Addition             | `add x5, x1, x2`  | `x5 = x1 + x2`            |
| `sub`  | Subtraction             | `sub x5, x1, x2`  | `x5 = x1 - x2`            |
| `sll`  | Logical left shift         | `sll x5, x1, x2`  | Shift `x1` left by `x2` bits        |
| `srl`  | Logical right shift         | `srl x5, x1, x2`  | Shift `x1` right by `x2` bits        |
| `sra`  | Arithmetic right shift         | `sra x5, x1, x2`  | Right shift preserving sign bit           |
| `and`  | Bitwise AND             | `and x5, x1, x2`  | Bitwise `AND` operation           |
| `or`   | Bitwise OR             | `or x5, x1, x2`   | Bitwise `OR` operation            |
| `xor`  | Bitwise XOR           | `xor x5, x1, x2`  | Bitwise `XOR` operation           |
| `slt`  | Set if less than (signed)| `slt x5, x1, x2`  | If `x1 < x2`, `x5=1`, otherwise 0 |
| `sltu` | Set if less than (unsigned)| `sltu x5, x1, x2` | Same as above but compares unsigned values         |

---

### (2) Common Immediate Instructions (I-Type)

I-type instructions perform operations on a register and an immediate value, suitable for quick computation and constant assignment.

| Instruction     | Meaning           | Example               | Description                            |
|----------|----------------|--------------------|---------------------------------|
| `addi`   | Add immediate        | `addi x5, x1, 10`  | `x5 = x1 + 10`                  |
| `andi`   | AND with immediate        | `andi x5, x1, 0xF` | Bitwise `AND`, commonly used for masking        |
| `ori`    | OR with immediate        | `ori x5, x1, 1`    | Commonly used to set a specific bit                  |
| `xori`   | XOR with immediate      | `xori x5, x1, 0x1` | Invert specific bits                      |
| `slti`   | Set if less than immediate      | `slti x5, x1, 5`   | Signed immediate comparison                |
| `sltiu`  | Set if less than immediate unsigned| `sltiu x5, x1, 5`  | Unsigned immediate comparison                |
| `slli`   | Shift left by immediate      | `slli x5, x1, 3`   | Maximum shift of 31 (RV32)             |
| `srli`   | Shift right by immediate      | `srli x5, x1, 2`   | Unsigned logical right shift                  |
| `srai`   | Arithmetic right shift by immediate  | `srai x5, x1, 2`   | Signed right shift preserving sign bit            |
| `lui`    | Load upper immediate  | `lui x5, 0x12345`  | `x5 = 0x12345000`               |
| `auipc`  | Add upper immediate to PC     | `auipc x5, 0x10`   | `x5 = PC + 0x10000`             |

---

### (3) Load Instructions (I-Type)

Used to read data from a memory address into a register. The address is calculated from `rs1 + offset`.

| Instruction   | Meaning                 | Example                   | Description                           |
|--------|----------------------|------------------------|--------------------------------|
| `lb`   | Load byte (signed)    | `lb x5, 0(x1)`         | Read 1 byte from the address pointed to by `x1`, sign-extend into `x5` |
| `lbu`  | Load byte (unsigned)    | `lbu x5, 0(x1)`        | Zero-extend                         |
| `lh`   | Load halfword (16-bit)     | `lh x5, 2(x1)`         | Sign-extend 16-bit integer             |
| `lhu`  | Load halfword (unsigned)    | `lhu x5, 2(x1)`        | Zero-extend                         |
| `lw`   | Load word (32-bit)       | `lw x5, 4(x1)`         | Commonly used load instruction                   |
| `lwu`  | Load word (unsigned, RV64)| `lwu x5, 4(x1)`        | 32-bit zero-extend for RV64           |
| `ld`   | Load doubleword (64-bit)     | `ld x5, 8(x1)`         | Primary instruction for RV64 architecture           |

### (4) Store Instructions (S-Type)

Used to write values from registers into memory. The address is still calculated from `rs1 + offset`.

| Instruction   | Meaning               | Example                   | Description                       |
|--------|--------------------|------------------------|----------------------------|
| `sb`   | Store byte           | `sb x5, 0(x1)`         | Stores only 1 byte                |
| `sh`   | Store halfword (16-bit)  | `sh x5, 2(x1)`         |                            |
| `sw`   | Store word (32-bit)    | `sw x5, 4(x1)`         | Shared by RV32 and RV64          |
| `sd`   | Store doubleword (64-bit)  | `sd x5, 8(x1)`         | Available only on RV64               |

---

### (5) Control Flow Instructions (B-Type)

Used to implement conditional branches, used with labels. The branch offset is a 12-bit immediate (±4KB).

| Instruction   | Meaning                     | Example                  | Description                            |
|--------|--------------------------|-----------------------|---------------------------------|
| `beq`  | Branch if equal                  | `beq x1, x2, label`   | If `x1 == x2`, branch to `label`   |
| `bne`  | Branch if not equal                  | `bne x1, x2, label`   | If `x1 != x2`, branch             |
| `blt`  | Branch if less than (signed)        | `blt x1, x2, label`   | If `x1 < x2`, branch              |
| `bge`  | Branch if greater or equal (signed)    | `bge x1, x2, label`   | If `x1 >= x2`, branch             |
| `bltu` | Branch if less than (unsigned)        | `bltu x1, x2, label`  | Unsigned less-than comparison branch             |
| `bgeu` | Branch if greater or equal (unsigned)    | `bgeu x1, x2, label`  | Unsigned greater-or-equal comparison branch         |

---

### (6) Jump and Link Instructions (J-Type, I-Type)

Used for procedure calls and function jumps, saving the return address to `ra`.

| Instruction   | Meaning           | Example                 | Description                                |
|--------|----------------|----------------------|-------------------------------------|
| `jal`  | Jump and link (J-type)| `jal x1, label`      | `x1 = PC+4; PC = label` (used for calls) |
| `jalr` | Register indirect jump (I-type)| `jalr x1, x2, 0`    | `x1 = PC+4; PC = x2 + 0` (commonly used for returns, calling function pointers)|

- If the destination register is `x0`, it means **the return address is not saved** (pure jump).
- `jal` is typically used for calling functions, while `jalr` is used for returning from functions.

---

### (7) Special System Call Instructions (Environment Interaction)

| Instruction     | Meaning                    | Example                       | Description                             |
|----------|-------------------------|----------------------------|----------------------------------|
| `ecall`  | Environment call (system call)     | `ecall`                    | Triggers an environment call exception          |
| `ebreak` | Debug breakpoint                 | `ebreak`                   | Triggers a debug exception                       |

## 0x04 Calling Convention

Stack frame structure (RISC-V ABI stack frame)

```text
High Address
│
│  +-----------------------------+
│  | Saved registers              |
│  +-----------------------------+
│  | Arguments beyond register    |
│  | passing limit > a7/fa7       |
│  +-----------------------------+
│  | Return address (ra)          │
│  +-----------------------------+
│  | Local variables              |
│  +-----------------------------+
│  | (16-byte alignment padding)  |
└──► Stack pointer (sp)
```

### 1. **Stack Growth**
- **Grows toward lower addresses**
- `sp` (x2) is always kept **16-byte aligned**
- `sp` is commonly used as a pointer to access stack contents (e.g., `ld t0, 16(sp)`)

### 2. **Saving Register Contents**
Saved by **callee**:
- `s0–s11` (x8–x9, x18–x27) → Used for long-lived data
- `fs0–fs11` (f8–f9, f18–f27) → Floating-point saved registers
- `ra` (x1) → Return address (saved by callee after the call pseudo-instruction modifies ra)

Saved by **caller**:
- `t0–t6` (x5–x7, x28–x31)
- `ft0–ft11` (f0–f7, f28–f31)

### Argument Size Constraints
- Smaller than pointer size (e.g., `char`, `short`): Placed in the lower bits of a register
- Larger than one pointer size (e.g., `long long`): Requires even-aligned registers
  - For example, in RV32:
    ```c
    void foo(int, long long)
    ```
    - `int` → `a0`
    - `long long` → `a2` and `a3` (skipping `a1` for even alignment)

- Larger than two pointer sizes (e.g., `struct { long x[3]; }`) → **Passed by reference**

### Return Value Passing Rules

- Return values use:
  - Integer: `a0`, `a1`
  - Floating-point: `fa0`, `fa1`
- Struct returns:
  - ≤2 pointer-word: Use `a0/a1`
  - >2 pointer-word: **Caller allocates memory** and passes a **hidden pointer as the first argument**

### Register Usage Overview

| Type           | Name       | Purpose                         | Saved By        |
|----------------|------------|------------------------------|-----------------|
| x1             | ra         | Return address                     | Caller |
| x2             | sp         | Stack pointer                       | Callee |
| x8             | s0 / fp    | Frame pointer / Saved register         | Callee        |
| x10–x17        | a0–a7      | Argument / Return value registers          | Caller          |
| x5–x7, x28–x31 | t0–t6      | Temporary variables                     | Caller          |
| x9, x18–x27    | s1, s2–11  | Saved variables                     | Callee        |
| f10–f17        | fa0–fa7    | Floating-point arguments / Return values            | Caller          |
| f0–f7, f28–f31 | ft0–11     | Floating-point temporary variables                 | Caller          |
| f8–f9, f18–f27 | fs0–fs11   | Floating-point saved variables                 | Callee        |

> Note that `sp` and `ra` registers strictly do not belong to the **scratch/preserved** register classification. The handling of these two registers is mostly implemented as callee-saved. Additionally, registers like `gp` do not fall into this category.

## References

- [RISC-V ELF psABI Document (riscv-non-isa/riscv-elf-psabi-doc)](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)  
- [RISC-V Instruction Set Architecture Official Manual (riscv/riscv-isa-manual)](https://github.com/riscv/riscv-isa-manual)
