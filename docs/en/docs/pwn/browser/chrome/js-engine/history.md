# V8 Design History

This section gives a general overview of V8's development history, intended for extended reading.

### 2008: Full-code Generation + Semi-optimization

At this time V8 had just been released. In short, it used a full code generation approach: **all JS code was JIT-compiled to generate machine code for the corresponding architecture**, followed by a small amount of optimization.

![](https://s2.loli.net/2025/03/11/LeisDWScb2ZVUC1.png)

### 2010: Full-code Generation + Crankshaft JIT Compiler & Optimizer

After two years of patches and improvements, the V8 development team split the original code optimization portion into a standalone JIT compiler called [Crankshaft](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html). The JS code compilation pipeline gained an additional branch to Crankshaft, and V8 also began introducing the Deoptimization mechanism:

![](https://s2.loli.net/2025/03/11/cd1xWp8gA5wnyhD.png)

> When does the AST generated from JS code go to the original full code compilation engine? When does it go to Crankshaft? This is mainly determined by the configuration options enabled in Chromium. At this point, Chromium's execution mode could either first pass JS to the original full code generation engine and then to Crankshaft for optimization, or directly to Crankshaft.

### 2014: Full-code Generation + Crankshaft / TurboFan JIT Compiler & Optimizer

Four more years passed, and a brand new JavaScript JIT Compiler + Optimizer named [TurboFan](https://v8.dev/docs/turbofan) emerged. The compilation pipeline became as shown below — the original pipeline gained an additional fork pointing to TurboFan, with TurboFan and Crankshaft existing in parallel:

![](https://s2.loli.net/2025/03/11/7Ptx93koOHsIi12.png)

> When does the AST generated from JS code go to the original full code compilation engine? When does it go to Crankshaft? When does it go to TurboFan? Same as before, this is also determined by specific configuration options.

### 2016: Ignition Interpreter + Full-code Generation + Crankshaft / TurboFan JIT Compiler & Optimizer

Two more years passed, and the JS interpreter named [Ignition](https://v8.dev/blog/ignition-interpreter) emerged. The V8 engine **introduced the concept of interpreted execution for the first time**. The reason was that full code generation was still too time-consuming and space-consuming, dragging down execution efficiency — even with the powerful assistance of TurboFan, it wasn't as fast as direct interpretation. Therefore, from this year onward, interpreted execution became the primary way V8 executes JS code:

![](https://s2.loli.net/2025/03/11/CocnM4vmT8qpXwx.png)

Ignition at this point was an _optional feature_. For V8 with Ignition enabled, the JS AST would be given to Ignition to generate bytecode for interpreted execution, and frequently executed bytecode would be passed to TurboFan for JIT compilation to generate machine code, or to the original full code generation + Crankshaft optimization path. For V8 without Ignition enabled, it followed the original full code generation + Crankshaft optimization path (back to 2010, essentially).

> At this point the entire compilation pipeline had become large and complex, so a new round of architectural optimization was inevitable——

### 2017: Ignition Interpreter + TurboFan JIT Compiler

Another year passed and everyone realized that with the two powerful components Ignition and TurboFan, _the full code generation JIT Compiler and the outdated Crankshaft seemed no longer necessary, so these two components were directly removed_. **The compilation pipeline became the streamlined Ignition Interpreter + TurboFan JIT Compiler**. The V8 team was so excited about this that they published an article titled [Launching Ignition and TurboFan](https://v8.dev/blog/launching-ignition-and-turbofan), and interpreted execution ultimately became the default way V8 Engine executes JS code:

![](https://s2.loli.net/2025/03/11/CX81DoyTBaeV46n.png)

> After this, the compilation pipeline has remained largely unchanged. The developers have simply continued to make incremental improvements on top of these two pillars......
