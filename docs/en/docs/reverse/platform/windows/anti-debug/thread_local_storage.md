# Thread Local Storage (TLS)

Thread Local Storage (TLS) is used to initialize specific thread data before a thread starts. Since every process contains at least one thread, data is initialized before the main thread runs. The initialization can be done by specifying a static buffer that has been copied to dynamically allocated memory, and/or by executing code in a callback function array to initialize dynamic memory contents. Problems often arise from the abuse of the callback function array.

At runtime, the contents of the TLS callback function array can be modified or extended. Newly added or modified callback functions will be called using the new addresses. There is no limit on the number of callback functions. The array extension can be accomplished with the following code:

``` asm
l1: mov d [offset cbEnd], offset l2
    ret
l2: ...
```

When the callback at l1 returns, the callback function at l2 will be called next.

> todo: continue to finish it


