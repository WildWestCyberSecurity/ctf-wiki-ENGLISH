# Race Condition

## Overview

A race condition occurs when the outcome of a system depends on the sequence of uncontrollable events. When these uncontrollable events do not occur in the order the developer intended, bugs may arise. The term originally comes from two electrical signals racing each other to influence the output.

![](./figure/race_condition.png)

Race conditions mainly occur in the following fields:

- Electronic systems, especially logic circuits
- Computers, especially multithreaded programs and distributed programs.

Since modern systems heavily employ concurrent programming and frequently share resources, race condition vulnerabilities are common.

Here we mainly consider race conditions in computer programs. When the execution result of software depends on the order of processes or threads, a race condition may occur. Upon simple consideration, we can see that race conditions require the following **conditions**:

- Concurrency, meaning there are at least two concurrent execution flows. The execution flows here include threads, processes, tasks, and other levels of execution flows.
- Shared objects, meaning multiple concurrent flows access the same object. **Common shared objects include shared memory, file systems, and signals. Generally, these shared objects are used to enable communication between multiple execution flows.** Additionally, the code that accesses shared objects is called the **critical section**. When writing code normally, this section should be locked.
- Changing the object, meaning at least one execution flow will change the state of the contested object. If the program only reads the object, no race condition will occur.

Due to the high uncertainty of execution flows during concurrency, race conditions are relatively **hard to detect** and **difficult to reproduce and debug**. This also makes fixing race conditions quite challenging.

The impact of race conditions is diverse — in mild cases the program executes abnormally, and in severe cases the program crashes. If a race condition vulnerability is exploited by an attacker, it is very likely that the attacker will gain privileges on the corresponding system.

Here is a simple example.

```c
#include <pthread.h>
#include <stdio.h>

int counter;
void *IncreaseCounter(void *args) {
  counter += 1;
  sleep(0.1);
  printf("Thread %d has counter value %d\n", (unsigned int)pthread_self(),
         counter);
}

int main() {
  pthread_t p[10];
  for (int i = 0; i < 10; ++i) {
    pthread_create(&p[i], NULL, IncreaseCounter, NULL);
  }
  for (int i = 0; i < 10; ++i) {
    pthread_join(p[i], NULL);
  }
  return 0;
}

```

Generally, we might expect the output to be as follows:

```shell
➜  005race_condition ./example1
Thread 1859024640 has counter value 1
Thread 1841583872 has counter value 2
Thread 1832863488 has counter value 3
Thread 1824143104 has counter value 4
Thread 1744828160 has counter value 5
Thread 1736107776 has counter value 6
Thread 1727387392 has counter value 7
Thread 1850304256 has counter value 8
Thread 1709946624 has counter value 9
Thread 1718667008 has counter value 10
```

However, due to the existence of race conditions, the actual output is often unsatisfactory:

```c
➜  005race_condition ./example1
Thread 1417475840 has counter value 2
Thread 1408755456 has counter value 2
Thread 1391314688 has counter value 8
Thread 1356433152 has counter value 8
Thread 1365153536 has counter value 8
Thread 1373873920 has counter value 8
Thread 1382594304 has counter value 8
Thread 1400035072 has counter value 8
Thread 1275066112 has counter value 9
Thread 1266345728 has counter value 10
```

Let's think carefully about why a race condition might occur, using the following concrete example:

- The program first executes action1, then executes action2. The actions may be at the application level or the operating system level. Normally, we expect the conditions produced by action1 to still hold when action2 is executed.
- However, due to program concurrency, an attacker may be able to break the conditions produced by action1 during the brief time window before action2 is executed. At this point, the attacker's operation races with action2, which may affect the program's execution outcome.

![](./figure/time_interval.png)

So I believe the root of the problem is that although the programmer assumes a certain condition should be satisfied during a given time period, the condition may actually be modified within this very small time window. **Although this time interval may be very small, attackers can still potentially slow down the victim machine's processing speed by executing certain operations (such as computation-intensive operations or DoS attacks).**

## Forms

Common race conditions come in the following forms.

### CWE-367: TOCTOU Race Condition

#### Description

TOCTOU (Time-of-check Time-of-use) refers to when a program checks a resource (variable, memory, file) before using it, but the resource is modified before the program actually uses it.

![](./figure/toctou.png)

Below are some more specific examples.

#### CWE-365: Race Condition in Switch

When a program is executing a switch statement, if the value of the switch variable is changed, it may cause unpredictable behavior. Especially in code where break statements are not written after case statements, once the switch variable changes, the original program logic is very likely to be altered.

#### CWE-363: Race Condition Enabling Link Following

We know that Linux provides two ways of naming files:

- File path names
- File descriptors

However, the methods of resolving these two naming schemes to the corresponding objects differ:

- File path names are resolved **indirectly** through the supplied path (filename, hard link, symbolic link), and the parameter passed is not the actual address (inode) of the corresponding file.
- File descriptors resolve by accessing a pointer that directly points to the file.

It is precisely this indirection that creates the time window mentioned above.

Taking the following code as an example, the program checks whether a file exists before accessing it, and then opens the file and performs operations. However, if an attacker modifies the file to a symbolic link after the check but before the actual use of the file, the program will access the wrong file.

![](./figure/race_condition_file.png)

The root cause of this type of race condition lies in the name-to-object binding problem in the file system. The following functions all use filenames as parameters: access(), open(), creat(), mkdir(), unlink(), rmdir(), chown(), symlink(), link(), rename(), chroot(),…

How can we avoid this problem? We can use the fstat function to read file information and store it in a stat structure, then compare this information with what we already know to determine whether we have read the correct information. In the stat structure, the `st_ino` and `st_dev` variables can uniquely identify a file:

- `st_ino`, contains the file's serial number, i.e., `i-node`
- `st_dev`, contains the device corresponding to the file.

![](./figure/race_condition_identify_file.png)

### CWE-364: Signal Handler Race Condition

#### Overview

Race conditions frequently occur in signal handlers because signal handlers support asynchronous operations. Especially when signal handlers are **non-reentrant** or state-sensitive, an attacker may exploit race conditions in signal handlers to potentially achieve denial-of-service attacks and code execution. For example, if a free operation is performed in a signal handler and another signal arrives, the signal handler will execute another free operation, resulting in a double free situation. With a bit more manipulation, it may be possible to achieve arbitrary address write.

Generally speaking, common race condition scenarios related to signal handlers include:

- Signal handlers and normal code segments share global variables and data segments.
- Sharing state between different signal handlers.
- Signal handlers themselves use non-reentrant functions, such as malloc and free.
- A single signal handler handles multiple signals, which may further lead to use-after-free and double free vulnerabilities.
- Using mechanisms such as setjmp or longjmp that prevent the signal handler from returning to the original program execution flow.

#### Thread Safety and Reentrancy

Here we explain the relationship between thread safety and reentrancy.

-   Thread safety
    -   Means that the function can be called by multiple threads without any problems.
    -   Conditions
        -   Has no shared resources of its own
        -   Has shared resources, which need to be locked.
-   Reentrancy
    -   A function where multiple instances can run simultaneously in the same address space.
    -   A reentrant function can be interrupted, and when other code enters the same function, data integrity is not lost. Therefore, a reentrant function is necessarily thread-safe.
    -   Reentrancy emphasizes that when a single thread is executing and re-enters the same subroutine, it is still safe.
    -   Conditions that violate reentrancy
        -   The function body uses static data structures that are not constants
        -   The function body uses malloc or free functions
        -   The function uses standard I/O functions.
        -   Called functions are not reentrant.
    -   All variables used by reentrant functions are stored on the current [function frame](https://en.wikipedia.org/wiki/Call_stack) of the [call stack](https://en.wikipedia.org/wiki/Call_stack).

## Prevention

To eliminate race conditions, the primary goal is to find the race windows.

A race window is the code section that accesses the contested object, which gives attackers the opportunity to modify the corresponding contested object.

Generally speaking, if we can make conflicting race windows mutually exclusive, we can eliminate race conditions.

### Synchronization Primitives

Generally, we use synchronization primitives to eliminate race conditions. Common ones include:

-   Lock variables
    -   Typically mutex locks, which yield the CPU during waiting, entering an idle state, and automatically retry after a period of time.
    -   Spinlocks, which do not yield the CPU during waiting and continuously retry.
-   Condition variables
    -   **Condition variables are used for waiting, not for locking. Condition variables are used to automatically block a thread until a special condition occurs. Condition variables are usually used together with mutex locks.**
-   Critical section objects, CRITICAL_SECTION

-   Semaphores, which control the number of threads that can access a critical section, generally greater than 1.
-   Pipes, which are shared files used to connect a read process and a write process to enable communication between them. Their lifetime does not exceed that of the process that created the pipe.
-   Named pipes, whose lifetime can be as long as the operating system's runtime.

```
# Create a pipe
mkfifo my_pipe
# gzip reads data from the given pipe and compresses it to out.gz
gzip -9 -c < my_pipe > out.gz &
# Send data to the pipe
cat file > my_pipe
```

### Deadlock

#### Overview

When synchronization primitives are used improperly, processes may encounter deadlocks. A deadlock occurs when two or more execution flows block each other, preventing all of them from continuing. In fact, deadlocks are mainly caused by circular waiting among conflicting execution flows, where each execution flow in the circular wait holds a resource while trying to acquire the next one. As shown in the figure below, processes P1 and P2 both need resources to continue running. P1 holds resource R2 and needs additional resource R1 to run; P2 holds resource R1 and needs additional resource R2 to run. Both are waiting for each other and neither can proceed.

![](./figure/process_deadlock.png)

Generally speaking, deadlocks have four necessary conditions:

- Mutual exclusion — resources are mutually exclusive.
- Hold and wait — holding existing resources while waiting for the next resource.
- No preemption — resources obtained by a process cannot be forcibly taken from the resource holder by the requester before they are fully used; they can only be voluntarily released by the holding process.
- Circular wait — circular waiting for resources.

To eliminate deadlocks, one must break one of the above four necessary conditions.

Additionally, deadlocks may arise from the following causes:

- Processor speed
- Changes in process or thread scheduling algorithms
- Different memory constraints during execution.
- Any asynchronous event that can interrupt program execution.

#### Impact

Deadlocks generally cause denial-of-service attacks.

## Detection

So, at this point, is it possible to detect race condition vulnerabilities? There is indeed research in this area, mainly from two perspectives: static analysis and dynamic analysis.

### Static Detection

Currently known static detection tools include:

-   [Flawfinder](http://www.dwheeler.com/flawfinder/)
    -   Target: C/C++ source code
    -   Steps
        -   Build a vulnerability database
        -   Perform simple text pattern matching, without any data flow or control flow analysis
-   [ThreadSanitizer](https://github.com/google/sanitizers)
    -   Target: C++ and Go
    -   Implementation: LLVM

### Dynamic Detection

- [Intel Inspector](https://en.wikipedia.org/wiki/Intel_Inspector)
- [Valgrind](https://en.wikipedia.org/wiki/Valgrind)

# References

- http://www.teraits.com/pitagoras/marcio/segapp/05.ppt
- http://repository.root-me.org/Programmation/C%20-%20C++/EN%20-%20Secure%20Coding%20in%20C%20and%20C++%20Race%20Conditions.pdf
- https://www.blackhat.com/presentations/bh-europe-04/bh-eu-04-tsyrklevich/bh-eu-04-tsyrklevich.pdf
- https://xinhuang.github.io/posts/2014-09-23-detect-race-condition-using-clang-thread-sanitizer.html
- https://llvm.org/devmtg/2011-11/Hutchins_ThreadSafety.pdf
- http://www.cnblogs.com/biyeymyhjob/archive/2012/07/20/2601655.html
- http://www.cnblogs.com/huxiao-tee/p/4660352.html
- https://github.com/dirtycow/dirtycow.github.io
