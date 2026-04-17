# Mutable Heap Numbers

The **Mutable Heap Numbers** mechanism is an optimization mechanism publicly disclosed by the V8 team in 2025. The idea originated from a performance bottleneck of `Math.random` in `async-fs`, a JS filesystem implementation. For example, consider the following code:

```javascript
let seed;
Math.random = (function() {
  return function () {
    seed = ((seed + 0x7ed55d16) + (seed << 12))  & 0xffffffff;
    seed = ((seed ^ 0xc761c23c) ^ (seed >>> 19)) & 0xffffffff;
    seed = ((seed + 0x165667b1) + (seed << 5))   & 0xffffffff;
    seed = ((seed + 0xd3a2646c) ^ (seed << 9))   & 0xffffffff;
    seed = ((seed + 0xfd7046c5) + (seed << 3))   & 0xffffffff;
    seed = ((seed ^ 0xb55a4f09) ^ (seed >>> 16)) & 0xffffffff;
    return (seed & 0xfffffff) / 0x10000000;
  };
})();
```

The variable `seed` changes with every call to `Math.random`, generating a pseudo-random sequence. The key point is that `seed` is stored in a `ScriptContext`, a structure used to represent the storage location of accessible values within a specific script (in plain terms, **the context of a JS script**). Internally, this structure is an array of V8 tagged values:

- ScopeInfo: context metadata.
- NativeContext: global object.
- Slot 0 ~ Slot n: various other values.

The values of these Slots are typically 32 bits, with the least significant bit of each value used as a tag, so when used, the value is right-shifted by 1 bit:

- 0: 31-bit small integer (SMI).
- 1: 31-bit pointer.

Correspondingly, data larger than 31 bits is stored on the heap, with the ScriptContext's Slot storing pointers to these objects. For simple numeric types, a `HeapNumber` object is used for storage, while complex objects use structures like `JSObject`.

![](https://s2.loli.net/2025/03/11/BEhSayrC24XJ6Tq.png)

This is where the bottleneck arises:

- **HeapNumber allocation**: In the JS code above, the `seed` variable is stored in a `HeapNumber` object, so memory allocation and deallocation occurs with every `Math.random` function call.
- **Floating-point arithmetic**: Although `Math.random` uses integer operations throughout, `seed` is stored in a generic `HeapNumber`, causing the compiler to generate slower floating-point operation instructions, as well as requiring potential 64-bit floating-point to 32-bit integer conversions and precision loss checks even when the compiler knows it might be a 32-bit integer.

How to break through? The V8 team adopted a two-part optimization:

- **Slot type tracking / mutable heap number slots**: The V8 team extended [script context const value tracking](https://issues.chromium.org/u/2/issues/42203515) to include type information, tracking whether a slot value is a constant, `SMI`, `HeapNumber`, or generic tagged value. They introduced the concept of **mutable heap numbers** in script contexts, similar to [mutable heap number fields](https://v8.dev/blog/react-cliff#smi-heapnumber-mutableheapnumber) of `JSObjects`: the slot value changed from pointing to a changing `HeapNumber` to **holding a HeapNumber** — thereby eliminating heap reallocation during code optimization updates.
- **Mutable heap Int32**: The V8 team enhanced script context slot types to track whether a numeric value is within the Int32 range. If so, the `HeapNumber` stores it as a pure `Int32`; if it needs to be extended to `double`, no reallocation of `HeapNumber` space is needed. In this case, the `Math.random` in the example above, under compiler observation, is known to be a continuously updated integer value, so the corresponding slot is marked as a mutable `Int32`.

Note that such optimization introduces code dependencies on the context slot's value. If the slot's type changes (e.g., becomes a string), the optimization will be rolled back. Therefore, ensuring slot type stability is crucial (~~in other words, don't keep changing the type of a JS variable~~).

Ultimately, the optimization results for `Math.random` are as follows:

- **No allocation / fast in-place updates**: Updating `seed` does not require repeatedly allocating new heap objects.
- **Integer operations**: The compiler knows the type is `Int32`, thus avoiding generating inefficient floating-point operations.

In the end, the `async-fs` benchmark speed improved by 2.5x, and the overall score on [JetStream2](https://browserbench.org/JetStream2.1/) improved by approximately `1.6%` — quite an effective result.
