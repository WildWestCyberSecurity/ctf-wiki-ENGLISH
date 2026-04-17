# Chromium

## Overview

[Chromium](https://www.chromium.org/Home/) is an open-source browser project under [The Chromium Projects](https://www.chromium.org/chromium-projects/), licensed under multiple licenses including BSD 3-clause. It is primarily written in C++, and the source code can be obtained from [https://chromium.googlesource.com/chromium/src/+/HEAD/docs/get_the_code.md](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/get_the_code.md).

[Google Chrome](https://www.google.com/chrome/) is a browser developed by Google based on Chromium. Google's developers have added proprietary Google code (Google account, tab sync, media decoders, etc.), making it a proprietary Google project.

Currently, the vast majority of actively maintained browsers in the world are developed based on Chromium (using Chromium as their engine), including [Microsoft Edge](https://www.microsoft.com/en-us/edge/) which abandoned its own browser engine. Only [Firefox](https://www.mozilla.org/en-US/firefox/new/) and [Safari](https://www.apple.com/safari/) still persist in using other engines.

![](https://s2.loli.net/2025/03/08/3u7rOs6j4gQJzRa.png)

——But in reality, a browser is not something that can be summarized in just a few words. Chromium itself is a complete browser project, and its rendering engine is [Blink](https://www.chromium.org/blink/), which is also what Microsoft primarily relies on from Chromium (because their own EdgeHTML engine, derived from the IE-era Trident engine, was not quite up to par). Correspondingly, Safari's rendering engine is [the open-source WebKit](https://webkit.org/), and Firefox's rendering engine is [the open-source Gecko](https://firefox-source-docs.mozilla.org/overview/gecko.html). However, beyond that, Microsoft's JavaScript engine is the self-developed [open-source Chakra](https://github.com/chakra-core/ChakraCore) rather than Chromium's [V8](https://v8.dev/). Similarly, Safari's JS engine is its own [Nitro](), and Firefox's JS engine is the self-developed [open-source SpiderMonkey](https://spidermonkey.dev/)......

> This is also why when we talk about self-developed browsers, we usually only recognize 4. The various other so-called "self-developed" browsers that are just reskinned versions of Chromium have neither their own layout engine, nor their own JS engine, nor their own network stack... In short, none of the core components are self-developed, so one might wonder...

So how does a browser work? Some people might think "isn't it just receiving an HTML file, parsing it, and drawing it on the screen?" — **Only** speaking in terms of the rendering engine, that does seem to be a fair summary? The rendering engine consumes the HTML, CSS, JS files sent by the server, parses them, and hands them to the GPU to draw on the screen:

![](https://s2.loli.net/2025/03/08/6ObUMxFAh5Tkvw1.png)

Expanding a bit, it looks like this (the HTTP Client is the browser):

![](https://s2.loli.net/2025/03/08/OLtXs5rTjk2i7pG.png)

Parsing HTML produces a DOM Tree representing the content, parsing CSS produces a CSSOM Tree representing the layout, and combining them gives a Render Tree containing specific rendering information. Add the JS Engine for dynamic content adjustment and various behind-the-scenes work, and the prototype of a modern browser seems to take shape:

![](https://s2.loli.net/2025/03/08/6agfHueKnjmJCB1.png)

Although it may seem like even an undergraduate just learning compiler principles could hand-craft a simple parsing engine using recursive descent, in reality, implementing modern Web standards with sufficiently high performance requires far more work than most people imagine. Whether it's the network protocol stack behind the browser, memory management and multi-process architecture, or the implementation details of the rendering engine and JS engine — any small piece taken individually could fill many papers. Theoretically, no single developer could independently explain all the parts clearly, and the focus of this article is not on how the entire browser works, so we won't go into depth here :)

![](https://s2.loli.net/2025/03/08/AxTcI3XVnUyZ68l.png)

## Chromium Render Process

The runtime architecture of Chromium is roughly as shown in the diagram below (communication between these processes is typically done through [Mojo](https://chromium.googlesource.com/chromium/src/+/lkgr/mojo/README.md), an IPC Engine):

![Why so cartoonish? Because the author couldn't find a more suitable diagram...](https://s2.loli.net/2025/03/08/63bwqkTr1WAXm5g.png)

The Browser Process is responsible for network request handling, storage management, UI, etc. (the left, middle, and right threads in the diagram above). After completing the reception of network request content, it passes the page data to the Render Process for rendering:

![](https://s2.loli.net/2025/03/08/wj6VW8IsRpQlG4Y.png)

The Render Process first parses the HTML, CSS, and JavaScript content. As mentioned earlier, parsing HTML ultimately produces a DOM Tree, and JavaScript at this point **blocks HTML parsing and is parsed and executed first** — this is because JS code may modify the document content (e.g., using `document.write()`).

> If your inline JS code in HTML does not use `document.write()`, you can use the [async](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-async) or [defer](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-defer) attributes on the `<script>` tag to run JS asynchronously without blocking the parsing process.

![](https://s2.loli.net/2025/03/08/hcvo3XOuCgVNs1k.png)

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
