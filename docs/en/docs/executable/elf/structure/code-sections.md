# Code Section

## Overview

After the dynamic linker creates the process image and performs relocations, each shared object file has the opportunity to execute some initialization code. All shared object files are initialized before the executable file gains control.

Before calling the initialization code of object file A, the initialization code of all shared object files that A depends on will be called first. For example, if object file A depends on another object file B, then B will be in A's dependency list, which is recorded in the DT_NEEDED entry of the dynamic structure. Circular dependency initialization is undefined.

The initialization of object files is completed by recursively processing each dependency entry. An object file will only execute its initialization code after all the object files it depends on have processed their own dependencies.

The following example explains two correct orderings that can be used to generate the given example. In this example, a.out depends on b, d, and e. b depends on d and f, and d depends on e and g. Based on this information, we can draw the following dependency graph. The algorithm described above will allow us to initialize in the following order.

![](figure/initialization_ordering_example.png)

Similarly, shared object files also have termination functions, which are executed through the atexit mechanism when the process completes its termination sequence. The order in which the dynamic linker calls termination functions is exactly the reverse of the initialization order above. The dynamic linker will ensure that it executes initialization or termination functions at most once.

Shared object files specify their initialization and termination functions through DT_INIT and DT_FINI in the dynamic structure. Typically, these functions are in the .init section and the .fini section.

Note:

>   Although atexit termination handlers are normally executed, they are not guaranteed to be executed when the program dies. More specifically, if the program calls the _exit function or the process dies due to receiving a signal, the corresponding functions will not be executed.

The dynamic linker is not responsible for calling the executable file's .init section or registering the executable file's .fini section using atexit. Termination functions specified by the user through the atexit mechanism must be executed before all shared object file termination functions.

## .init & .init_array

This section contains executable instructions and is part of the process initialization code. When the program begins execution, the system executes this code before calling the main program entry point (usually the C language main function).

## .text

This section contains the executable instructions of the program.

## .fini & .fini_array

This section contains executable instructions and is part of the process termination code. When the program exits normally, the system executes the code here.
