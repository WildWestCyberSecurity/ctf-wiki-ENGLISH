# TurboFan JIT Compiler

For full code compilation scenarios, most code may only execute once or twice. In that case, using interpreted execution to run JavaScript code is more efficient than traditional JIT compilation. However, when a piece of code needs to be executed repeatedly many times, the overall efficiency of JIT compilation is higher than interpreted execution — this is where V8's JIT compiler comes in: [TurboFan Compiler](https://v8.dev/docs/turbofan). When a piece of code is executed repeatedly, or is manually specified for compilation, Ignition passes the bytecode to TurboFan, which performs JIT compilation to generate machine code for the corresponding architecture.

![](https://s2.loli.net/2025/03/11/rfiMqLce6Qt435h.png)

## TurboFan Internals

> Under construction.

## Reference

https://arttnba3.cn

[What Are Rendering Engines: An In-Depth Guide](https://www.lambdatest.com/learning-hub/rendering-engines)

[RenderingNG deep-dive: BlinkNG](https://developer.chrome.com/docs/chromium/blinkng)

[How Does the Browser Render HTML?](https://component-odyssey.com/tips/02-how-does-the-browser-render-html)

[Browser's Rendering Pipeline](https://www.figma.com/community/file/1327562660128482813/browsers-rendering-pipeline)

[Inside look at modern web browser (part 1) ](https://developer.chrome.com/blog/inside-browser-part1)

[JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics)

[Winty's blog - Modern Browser Architecture Overview](https://github.com/LuckyWinty/blog/blob/master/markdown/Q%26A/%E7%8E%B0%E4%BB%A3%E6%B5%8F%E8%A7%88%E5%99%A8%E6%9E%B6%E6%9E%84%E6%BC%AB%E8%B0%88.md)

[Firing up the Ignition interpreter](https://v8.dev/blog/ignition-interpreter)

[Digging into the TurboFan JIT](https://v8.dev/blog/turbofan-jit)

[Ignition: Jump-starting an Interpreter for V8](https://docs.google.com/presentation/d/1HgDDXBYqCJNasBKBDf9szap1j4q4wnSHhOYpaNy5mHU/edit#slide=id.g1357e6d1a4_0_58)

[Ignition: An Interpreter for V8](https://docs.google.com/presentation/d/1OqjVqRhtwlKeKfvMdX6HaCIu9wpZsrzqpIVIwQSuiXQ/edit#slide=id.g1357e6d1a4_0_58)

[Deoptimization in V8](https://docs.google.com/presentation/d/1Z6oCocRASCfTqGq1GCo1jbULDGS-w-nzxkbVF7Up0u0/htmlpresent) 

[A New Crankshaft for V8](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html)

[TurboFan](https://v8.dev/docs/turbofan)

[Sea of Nodes](https://darksi.de/d.sea-of-nodes/)

[TurboFan: A new code generation architecture for V8](https://docs.google.com/presentation/d/1_eLlVzcj94_G4r9j9d_Lj5HRKFnq6jgpuPJtnmIBs88/edit#slide=id.g2134da681e_0_125)
