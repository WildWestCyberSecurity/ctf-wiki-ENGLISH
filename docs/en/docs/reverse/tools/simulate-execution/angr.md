# angr 

## Introduction

[angr](https://github.com/angr/angr) is a cross-platform, open-source binary **concolic** (concrete + symbolic) execution engine written in Python. It provides us with a series of practical binary analysis tools. For more information about angr, you can visit their [official website](https://angr.io), and for the APIs provided by angr, you can refer to the [documentation](https://api.angr.io/).

In CTF reverse engineering challenges, angr's powerful concolic execution engine can help us perform automated analysis more effectively, significantly reducing the time needed to solve challenges.

## Installation

angr itself can be installed directly via pip:

```shell
$ pip3 install angr
```

[angr-management](https://github.com/angr/angr-management) is the graphical interface for angr. After installation, you can launch it directly by typing `angr-management` in the terminal:

```shell
$ pip3 install angr-management
```

[angrop](https://github.com/angr/angrop) is also a project by the angr development team. It can automatically collect ROP gadgets and construct ROP chains:

```shell
$ pip3 install angrop
```

## Basic Usage

This section mainly covers the basic usage of angr and some commonly used APIs.

> Note: [angr_ctf](https://github.com/jakespringer/angr_ctf) is an excellent beginner-level angr practice project. You can use this project to familiarize yourself with angr's basic usage.

### Project

To use angr to analyze a binary file, the first step is to create an `angr.Project` instance — all subsequent operations will be based on this class instance. Here is an example:

```python
>>> import angr
>>> bin_path = './test' # file to be analyzed
>>> proj = angr.Project(bin_path)
WARNING | 2022-11-23 19:25:30,006 | cle.loader | The main binary is a position-independent executable. It is being loaded with a base address of 0x400000.
```

First, we can obtain basic information about the corresponding binary file through a project:

```python
>>> proj.arch     # architecture of the binary file
<Arch AMD64 (LE)>
>>> hex(proj.entry)    # entry point of the binary file
'0x401060'
>>> proj.filename # name of the binary file
'./test'
```

- `arch` is an instance of the `archiinfo.Arch` class, which contains various data such as CPU information for running the file:

  - `arch.bits` & `arch.bytes`: The word size of the CPU (in bits/bytes).

  - `arch.name`: Architecture name, e.g., _X86_.

  - `arch.memory_endness`: Endianness — big-endian is `Endness.BE`, little-endian is `Endness.LE`.

    > There is also a "middle-endian" `Endness.ME` in the source code :)

#### factory - Utility Class Factory

`project.factory` provides us with constructors for some useful classes.

##### block - Basic Block

angr analyzes code in units of basic blocks. We can obtain the **basic block** at a given address — a `Block` class instance — using `project.factory.block(address)`:

```python
>>> block = proj.factory.block(proj.entry) # extract the basic block
>>> block.pp() # pretty-print of disassemble code of the block
        _start:
401060  endbr64
401064  xor     ebp, ebp
401066  mov     r9, rdx
401069  pop     rsi
40106a  mov     rdx, rsp
40106d  and     rsp, 0xfffffffffffffff0
401071  push    rax
401072  push    rsp
401073  lea     r8, [__libc_csu_fini]
40107a  lea     rcx, [__libc_csu_init]
401081  lea     rdi, [main]
401088  call    qword ptr [0x403fe0]
>>> block.instructions # instructions in the block
12
>>> block.instruction_addrs # addr of each instruction
(4198496, 4198500, 4198502, 4198505, 4198506, 4198509, 4198513, 4198514, 4198515, 4198522, 4198529, 4198536)
```

##### state - Simulated Execution State

angr uses the `SimState` class to represent a _simulated program state_. Our various operations are essentially the process of stepping from one state to another.

We use `project.factory.entry_state()` to get the initial execution state of a program, and `project.factory.blank_state(addr)` to get a blank state that starts execution from a specified address:

```python
>>> state = proj.factory.entry_state()
>>> state = proj.factory.blank_state(0xdeadbeef)
```

- `state.regs`: The register state group, where each register is a _bitvector_ (BitVector). We can access the corresponding register by name (e.g., `state.regs.esp -= 12`).
- `state.mem`: The memory access interface for this state. We can perform memory access directly via `state.mem[addr].type` (e.g., `state.mem[0x1000].long = 4`; for reads, you also need to specify `.resolved` or `.concrete` to indicate a bitvector or an actual value, e.g., `state.mem[0x1000].long.concrete`).
- `state.memory`: Another form of memory access interface:
  - `state.memory.load(addr, size_in_bytes)`: Get a bitvector of the specified size at the given address.
  - `state.memory.store(addr, bitvector)`: Store a bitvector at the specified address.
- `state.posix`: POSIX-related environment interface, e.g., `state.posix.dumps(fileno)` gets the stream on the corresponding file descriptor.

In addition to these information retrieval interfaces for the simulated execution state, there are also corresponding solver interfaces `state.solver`, which we will cover in later sections.

##### simulation\_manager - Simulation Manager

angr separates the execution methods of a state into a `SimulationManager` class. The following two ways of writing are equivalent:

```python
>>> proj.factory.simgr(state)
<SimulationManager with 1 active>
>>> proj.factory.simulation_manager(state)
<SimulationManager with 1 active>
```

Two important methods:

- `simgr.step()`: Single-step execution **in units of basic blocks**.
- `simgr.explore()`: Perform path exploration to find states that satisfy the corresponding conditions.

The default parameter of `simgr.explore()` is `find`, i.e., the **desired condition**. When the simulation manager discovers during path exploration that the current state satisfies this condition, that state will be placed in the `simgr.found` list. If no such state is found, the list will be empty.

The desired condition can typically be reaching a specific address:

```python
>>> simgr.explore(find=0x80492F0) # explore to a specific address
WARNING  | 2023-07-17 04:04:28,825 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory with an unspecified value. This could indicate unwanted behavior.
WARNING  | 2023-07-17 04:04:28,825 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING  | 2023-07-17 04:04:28,826 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING  | 2023-07-17 04:04:28,826 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING  | 2023-07-17 04:04:28,826 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING  | 2023-07-17 04:04:28,826 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x7ffeff60 with 4 unconstrained bytes referenced from 0x819af30 (strcmp+0x0 in libc.so.6 (0x9af30))
WARNING  | 2023-07-17 04:04:28,826 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x7ffeff70 with 12 unconstrained bytes referenced from 0x819af30 (strcmp+0x0 in libc.so.6 (0x9af30))
<SimulationManager with 1 active, 16 deadended, 1 found>
```

The desired condition can also be a custom boolean function that takes a _state_ as its parameter. For example, if we want to find a path that outputs a specific string, we can check whether that string is present in the output. We can use `state.posix.dumps(file_descriptor)` to get the character stream on the corresponding file descriptor:

```
>>> def foo(state):
...     return b"Good" in state.posix.dumps(1)
... 
>>> simgr.explore(find=foo)
<SimulationManager with 17 deadended, 1 found>

```

In addition to the `find` parameter, we can also specify the `avoid` parameter — conditions that the simulator should **avoid** during execution. When a state meets such a condition, it will be placed in the `.avoided` list and will not continue execution. Similarly, the `avoid` parameter can be a specific address or a custom boolean function.

Furthermore, we can specify the `num_find` parameter to set the number of qualifying states to find. If not specified, all qualifying states will be stored in the `.found` list.

### Claripy

`Claripy` is angr's **solver engine**, which internally and seamlessly mixes several backends (concrete bitvectors, SAT solvers, etc.). Generally, we don't need to interact with it directly, but we typically use some of the interfaces it provides.

#### bitvector - Bit Vector

A **bitvector** is an important part of angr's solver engine. It represents **a sequence of bits**.

We can create a bitvector value with a concrete value of a specified length using `claripy.BVV(int_value, size_in_bits)` or `claripy.BVV(string_value)`:

```python
>>> bvv = claripy.BVV(b'arttnba3')
>>> bvv
<BV64 0x617274746e626133>
>>> bvv2 = claripy.BVV(0xdeadbeef, 32)
>>> bvv2
<BV32 0xdeadbeef>
```

Bitvectors of the same length can be used in operations. For bitvectors of different lengths, you can use `.zero_extend(extended_bits)` to perform zero-extension before operating on them. Note that bitvector value operations are also subject to overflow:

```python
>>> bvv2 = bvv2.zero_extend(32)
>>> bvv + bvv2
<BV64 0x617274754d102022>
>>> bvv * bvv
<BV64 0x9842ff8e63f3b029>
```

In addition to `bitvector value` which represents a concrete value, bitvectors also have `bitvector symbol` which represents a **symbolic variable**. We can create a named bitvector symbol of a specified length using `claripy.BVS(name, size_in_bits)`:

```python
>>> bvs = claripy.BVS("x", 64)
>>> bvs
<BV64 x_0_64>
>>> bvs2 = claripy.BVS("y", 64)
>>> bvs2
<BV64 y_1_64>
```

Bitvector symbols and bitvector values can also be combined in operations to form more complex expressions:

```python
>>> bvs3 = (bvs * bvs2 + bvv) / bvs
>>> bvs3
<BV64 (x_0_64 * y_1_64 + 0x617274746e626133) / x_0_64>
```

We can use `.op` and `.args` to get the operation type and arguments of a bitvector:

```python
>>> bvv.op
'BVV'
>>> bvs.op
'BVS'
>>> bvs3.op
'__floordiv__'
>>> bvs3.args
(<BV64 x_0_64 * y_1_64 + 0x617274746e626133>, <BV64 x_0_64>)
>>> bvv.args
(7021802812440994099, 64)
```

#### state - Simulated Execution State

##### State Solving

As mentioned earlier, `state.solver` provides some state-based solving interfaces. For example, the solver also has `.BVV()` and `.BVS()` interfaces for creating bitvectors.

When we need to solve for the concrete value of a bitvector symbol, we can first store the bitvector symbol in the state's memory/registers, then use simgr to explore to the corresponding state, and finally use the `state.solver.eval()` member function to get the value of the corresponding bitvector in the current state. Here is a simple example:

```python
bvs_to_solve = claripy.BVS('bvs_to_solve', 64)
init_state = proj.factory.entry_state()
init_state.memory.store(0xdeadbeef, bvs_to_solve)
simgr = proj.factory.simgr(init_state)
simgr.explore(find = 0xbeefdead)

solver_state = simgr.found[0]
print(solver_state.solver.eval(bvs_to_solve))
```

##### Memory Operations

As mentioned earlier, for a state's memory, we can use the corresponding interfaces of `state.memory`:

- `state.memory.load(addr, size_in_bytes)`: Get a bitvector of the specified size at the given address
- `state.memory.store(addr, bitvector)`: Store a bitvector at the specified address

Note that if you want to store a concrete value, you need to specify the endianness via the `endness` parameter.

### Emulated Filesystem

In angr, file system operations are performed through `SimFile` objects. SimFile is an abstract model of _storage_ — a SimFile object can represent a series of bytes, symbols, etc.

We can create a simulated file using `angr.SimFile()`. Examples of creating SimFiles with concrete values and symbolic variables are as follows:

```python
>>> import angr, claripy
>>> sim_file = angr.SimFile('a_file', content = "flag{F4k3_f1@9!}\n")
>>> bvs = claripy.BVS('bvs', 64)
>>> sim_file2 = angr.SimFile('another_file', bvs, size=8) # size in bytes there
```

Simulated files need to be associated with a specific state. We can insert a SimFile into a state's file system using `state.fs.insert(sim_file)` or `sim_file.set_state(state)`:

```python
>>> state.fs.insert('test_file', sim_file)
```

We can also read content from a file:

```python
>>> pos = 0
>>> data, actural_read, pos = sim_file.read(pos, 0x100)
```

For _stream_-type files (such as standard I/O, TCP connections, etc.), we can use `angr.SimPackets()` to create them:

```python
>>> sim_packet = angr.SimPackets('my_packet')
>>> sim_packet
<angr.storage.file.SimPackets object at 0x7f75626a2e80>
```

### Constraints

As mentioned earlier, bitvectors can be used in operations. Similarly, bitvectors can also be used in **comparison operations**, and the result is a `Bool` type object:

```python
>>> bvv = claripy.BVV(0xdeadbeef, 32)
>>> bvv2 = claripy.BVV(0xdeadbeef, 32)
>>> bvv == bvv2
<Bool True>
>>> bvs = claripy.BVS('bvs', 32)
>>> bvs == bvv + bvv2
<Bool bvs_0_32 == 0xbd5b7dde>
>>> bvs2 = claripy.BVS('bvs2', 32)
>>> bvs2 > bvs * bvv + bvv2
<Bool bvs2_1_32 > bvs_0_32 * 0xdeadbeef + 0xdeadbeef>
```

For comparisons involving symbolic values, the `Bool` type object directly represents the corresponding expression, and can therefore be added as a **constraint** to a state. We can add constraints to a corresponding state using `state.solver.add()`:

```python
>>> state.solver.add(bvs == bvv + bvv2)
>>> state.solver.add(bvs2 > bvs * bvv + bvv2)
>>> state.solver.eval(bvs2) # get the concrete value under constraints
```

In addition to the Bool class, Claripy also provides some operations that produce bitvectors as results. Here is an example (for the complete list, please refer to the [documentation](https://docs.angr.io/advanced-topics/claripy)):

```python
>>> claripy.If(bvs == bvs2, bvs, bvs2)
<BV32 if bvs_0_32 == bvs2_1_32 then bvs_0_32 else bvs2_1_32>
```

### Function hook

Sometimes we may need to hook a certain function. In such cases, we can use `project.hook(addr = call_insn_addr, hook = my_function, length = n)` to hook the corresponding call instruction:

- `call_insn_addr`: The address of the call instruction to be hooked
- `my_function`: Our custom Python function
- `length`: The length of the call instruction

Our custom function should be a function that receives `state` as a parameter. angr also provides a decorator syntactic sugar, so both of the following approaches work:

```python
# method 1
@project.hook(0x1234, length=5)
def my_hook_func(state):
    # do something, this is an example
    state.regs.eax = 0xdeadbeef

# method 2
def my_hook_func2(state):
    # do something, this is an example
    state.regs.eax = 0xdeadbeef
proj.hook(addr = 0x5678, hook = my_hook_func2, length = 5)
```

### Simulated Procedure

In angr, the `angr.SimProcedure` class is used to represent **a running procedure on a state** — functions are essentially SimProcedures.

We can represent a custom function by creating a class that inherits from `angr.SimProcedure` and overriding the `run()` method, where the parameters of `run()` are the arguments that the function receives:

```python
class MyProcedure(angr.SimProcedure):
    def run(self, arg1, arg2):
        # do something, this's an example
        return self.state.memory.load(arg1, arg2)
```

Custom function procedures are primarily used to replace existing functions in the file. For example, by default angr uses some built-in SimProcedures to replace certain library functions.

If we already have the symbol table for the binary file, we can directly use `project.hook_symbol(symbol_str, sim_procedure_instance)` to automatically hook all corresponding symbols in the file. The **parameters of the `run()` method are the arguments received by the replaced function**. Here is an example:

```python3
import angr
import claripy

class MyProcedure(angr.SimProcedure):
    def run(self, arg1, arg2):
        # do something, this's an example
        return self.state.memory.load(arg1, arg2)

proj = angr.Project('./test')
proj.hook_symbol('func_to_hook', MyProcedure())
```

Of course, within the `run()` method of a SimProcedure, we can also use some useful member functions:

- `ret(expr)`: Return from the function.
- `jump(addr)`: Jump to the specified address.
- `exit(code)`: Terminate the program.
- `call(addr, args, continue_at)`: Call a function in the file.
- `inline_call(procedure, *args)`: Inline call another SimProcedure.

### stash

In angr, different states are organized into different stashes of the simulation manager. We can step, filter, merge, move, etc. as needed.

#### Stash Types

There are the following types of stashes in angr:

- `simgr.active`: The list of active states. These will be executed by the simulator by default unless an alternative is specified.
- `simgr.deadended`: The list of dead states. When a state can no longer be executed (e.g., no valid instructions, invalid instruction pointer, not all of its successors are satisfiable), it will be placed in this list.
- `simgr.pruned`: The list of pruned states. When `LAZY_SOLVES` is specified, states only check satisfiability when necessary. When a state is found to be unsatisfiable (unsat) with `LAZY_SOLVES` specified, the state hierarchy will be traversed to determine the point in its history where it first became unsatisfiable. That point and all its descendants will be _pruned_ and placed in this list.
- `simgr.unconstrained`: The list of unconstrained states. When `save_unconstrained=True` is specified during creation of the `SimulationManager`, states considered **unconstrained** (i.e., the instruction pointer is controlled by user data or other sources of symbolic data) will be placed in this list.
- `simgr.unsat`: The list of unsatisfiable states. When `save_unsat=True` is specified during creation of the `SimulationManager`, states considered unsatisfiable (i.e., states with **conflicting constraints**, such as requiring the input to be both `"AAAA"` and `"BBBB"` at the same time) will be placed in this list.

There is also a state list that is not a stash — `errored`. If an error occurs during execution, the state and the error it produced will be wrapped in an `ErrorRecord` instance (accessible via `record.state` and `record.error`), and that record will be inserted into `errored`. We can use `record.debug()` to launch a debug window.

#### Stash Operations

We can use `stash.move()` to transfer states between stashes, as follows:

```python
>>> simgr.move(from_stash = 'unconstrained', to_stash = 'active')
```

During the transfer, we can also filter by specifying the `filter_func` parameter:

```python
>>> def filter_func(state):
...     return b'arttnba3' in state.posix.dumps(1)
...
>>> simgr.move(from_stash = 'unconstrained', to_stash = 'active', filter_func = filter_func)
```

A stash is essentially just a list, so during initialization we can specify the initial contents of each stash using a dictionary:

```python
>>> simgr = proj.factory.simgr(init_state,
...     stashes = {
...             'active':[init_state],
...             'found':[],
...     })
```

## REFERENCE

[Github - angr](https://github.com/angr/angr)

[angr documentation](https://api.angr.io/)

[【ANGR.0x00】Getting Started with angr's Basic Usage through angr-CTF](https://arttnba3.cn/2022/11/24/ANGR-0X00-ANGR_CTF/)
