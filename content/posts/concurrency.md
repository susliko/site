---
title: "Concurrent approaches to concurrency"
date: 2020-07-01T15:17:42+03:00
draft: false
tags: ["concurrency", "languages"]
---

Coroutine, fiber, goroutine, green thread, virtual thread, continuation, future, actor, generator, async/await.  
What do all of these buzzwords mean?  
What problems do they solve?
How do they relate?  
Which approaches do various languages take?  
Let's shed a light on this! :bulb:
<!--more-->

The reader is invited to read the broad-picture Part 1 and intersting for him examples from the Part 2.
- [Part 1. Concepts](#part-1-concepts)
  - [Reasons for concurrent programming](#reasons-for-concurrent-programming)
    - [Waiting optimization](#waiting-optimization)
    - [Multitasking](#multitasking)
  - [Units of execution](#units-of-execution)
    - [System processes and threads](#system-processes-and-threads)
    - [Green (lightweight) threads](#green-lightweight-threads)
    - [Coroutines and Fibers](#coroutines-and-fibers)
    - [Actors](#actors)
    - [Bonus: continuations](#bonus-continuations)
  - [Syntax](#syntax)
    - [Callbacks](#callbacks)
    - [Futures and async/await](#futures-and-asyncawait)
    - [Communicating sequential processes (CSP)](#communicating-sequential-processes-csp)
    - [Message passing](#message-passing)
    - [Monad comprehensions](#monad-comprehensions)
  - [Points of comparison](#points-of-comparison)
    - [Types of multitasking](#types-of-multitasking)
    - [Cancellation](#cancellation)
    - [Structured concurrency](#structured-concurrency)
- [Part 2. Implementations](#part-2-implementations)
  - [Erlang/Elixir](#erlangelixir)
  - [C](#c)
  - [Haskell](#haskell)
  - [JVM and Loom](#jvm-and-loom)
  - [Scala](#scala)
  - [Kotlin](#kotlin)
  - [Clojure](#clojure)
  - [Go](#go)
  - [JavaScript](#javascript)
  - [Python](#python)
  - [C++](#c-1)
  - [Rust](#rust)
- [Summary](#summary)

# Part 1. Concepts
## Reasons for concurrent programming
Every program consists of a bunch of CPU *computations* and syncronous *waiting* for something to become available (e.g. result of a network request, some lock).  
Instead of keeping CPU idle while waiting, we could start another computation *concurrently* (read "concurrent" as "with overlapping execution time bounds"): this would allow to accomplish more in less time.  
A set of techniques which enables concurrent execution of computations is called **concurrent programming**. 

### Waiting optimization
All waiting operations might be split into two categories: synchronisation (language-level) and I/O (system-level).  
*Synchronisation*:
* locks 
* blocking queues

*I/O*:
* network socket reads/writes
* disk reads/writes 
* DNS lookups
* other system calls (see [types of system calls](https://www.tutorialspoint.com/different-types-of-system-calls))

There are two ways to deal with waiting operations: either constantly poll for some completion condition or "subscribe" to the completion event via a callback which will resume a computation when called. The second approach gives the desired opportunity for CPU to work on other computations while previous computation might be paused until the waiting finishes.  
Most of the listed types of blocking might be implemented efficiently with that idea in mind:
* efficient implementations of synchronisation primitives (locks, blocking queues) are handled by the language designers
* asynchronous socket operations: [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) (Linux), [kqueue](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2) (FreeBSD), [iocp](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports) (Windows)
* asynchronous disk I/O: [ui_uring](https://kernel.dk/io_uring.pdf) (Linux), [aio](https://www.freebsd.org/cgi/man.cgi?query=aio&sektion=4&manpath=freebsd-release-ports) (FreeBSD), [overlapped I/O](https://docs.microsoft.com/en-us/windows/win32/sync/synchronization-and-overlapped-input-and-output?redirectedfrom=MSDN) (Windows)
* there is also a bunch of solutions for asynchronous DNS-lookups (e.g. Linux [qdns](https://linux.die.net/man/3/qdns))

### Multitasking
Besides waiting optimisation concurrent programming ships an opportunity to progress in execution of multiple CPU-bound concurrent tasks independently of the number of involved system threads.  
This approach is called **multitasking**.  
There are two types of it:
* **Preemptive multitasking**. In this model each computation might be paused by the scheduler and resumed at some time later.  
  Examples: system threads. Goroutines in Go, sparks in Haskell and actors in Erlang are roughly preemptive, nuances will be explored in the corresponding sections.
* **Cooperative multitasking**. This model allows only voluntary pauses, which allow other computations to get their CPU time. Voluntary pausing is often called **yielding**.  
  Examples: fibers in Lua, Python, Rust, Kotlin.

While preemptive approach does not allow long-running tasks to occupy all the CPU time, those programs are more prone to bugs like data races and race conditions, since the execution might be preempted at an unexpected point; therefore a very careful synchronisation is required.  
Cooperative approach is widespread in many languages since it is sufficient for most of applications (non latency-critical ones) and allows smaller amount of inappropriate execution scenarios to happen.

The next part will cover the known ways to implement multitasking.

*Useful links*:
* [C10k problem](https://en.wikipedia.org/wiki/C10k_problem): handling large amount of concurrent clients
* [list](https://gist.github.com/jboner/2841832) of latency numbers for RAM/CPU/disk/network operations
* [race condition vs data race](https://blog.regehr.org/archives/490)
## Units of execution
### System processes and threads
The lowest abstraction representing an executing program is a **system  (native) process**.  
Each process has its *address space* (some allocated memory to store all possible addresses to devices, files on disk, main memory locations, remote hosts). Processes are scheduled by a system scheduler. The scheduling is preemptive, in order to guarantee that all running processes will have a chance to progress.

Some programs require parallel execution (e.g. browser should be able to browse several pages in time). For this purpose a process can spawn multiple **system (native) threads** -- some objects, that share the address space of their process, but have own call stacks. To schedule a process means to schedule all of its threads.  
The goal of parallel execution might also be achieved with process spawning, but inter-process communication (via pipes or sockets) is slower (TODO reasons and links) than communication between threads sharing the same address space.

However, if a program needs to run up to millions of concurrent tasks, spawning system threads on per-task basis is not affordable because of the overhead in the thread start/stop time (TODO what time) and the amount of memory it occupies: the default size of Linux thread call stack is 2 megabytes. It turns out, that a system thread is a relatively precious resource, as one cannot spawn more than a couple of thousands of them on a single machine due to the memory limit. In case of single-threaded languages (e.g. JavaScript, Python, Lua) spawning system threads is not even an option for multitasking.  

We need abstractions, which enable multitasking, regardless of the amount of system threads, backing our program at the moment.

*Useful links*:
* [Linux processes and threads](https://www.slashroot.in/difference-between-process-and-thread-linux) 
* [Completely Fair Scheduler](https://opensource.com/article/19/2/fair-scheduling-linux): default Linux scheduler
* [Thread pooling](https://softwareengineering.stackexchange.com/a/173581): keeping a few long-living "worker" threads, which recieve tasks through some "executor" interface. Allows to avoid system thread spawning for every new task by scheduling these tasks onto existing threads from the pool. Each multi-thread language has its means for thread pooling, either language-level (e.g. in Java) or runtime-level (e.g. in Go).

### Green (lightweight) threads
Basically, each programming language has a choice either to rely on the system thread scheduling or implement custom units of execution and schedule them on top of the system threads.  
The latter approach is called **green (or lightweight) threading**.

The word "green" originated from the name of the team (The Green Team), which designed a language-level threading model for [Java 1.1](https://docs.oracle.com/cd/E19455-01/806-3461/ch2mt-41/index.html).  
Now this terminology does not define anything specific. Green threads are referred to as some language-level units of execution, which are scheduled by the the language runtime and have lesser context switch and creation overhead than system threads. For example, fibers can be named green threads, actors also imply the existence of some green thread implementation.

### Coroutines and Fibers
In order to implement multitasking we need to represent our computations as some objects, which can be launched, paused and resumed. This is where a coroutine concept becomes really handy.  
**Coroutine** is a line of execution with its local variables and instruction pointer, that has the ability to voluntarily pause (persisting values of its local variables and giving the control to another coroutine) and resume by another coroutine sometime in the future. Coroutines can be used to implement *generators* -- expressions, which can produce multiple values (see Python [generators](https://wiki.python.org/moin/Generators) for an example) and language-level multitasking. Although being a building block for multitasking, coroutines should not be associated with concurrency: this is just one application of the powerful concept.  
Coroutines are classified across multiple axes:  
* Control-transfer mechanism
  - *Asymmetric* (also semi-symmetric) implies two control-transfer operations, which induce the caller-callee relationships. One for calling another coroutine; another for yielding current coroutine with passing control to its caller.  
  Examples: [Lua](https://www.lua.org/pil/9.1.html) (native support), [Kotlin](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html) (library support).
  - *Symmetric* provides the only control-transfer operation, that allows to explicitly pass control to any coroutine. All coroutines operate at the same level, without caller-callee hierarchy. Examples: old-school [Modula-2](https://freepages.modula2.org/gintro/A%20Gentle%20Introduction%20to%20Modula-2.htm#Coroutines) (native support), [C++](http://www.crystalclearsoftware.com/soc/coroutine/coroutine/symmetric_coroutines.html) (library support; both symmetric and asymmetric).  

  Symmetric coroutines, acting like *goto* statement, seem to be mostly abandoned by modern languages in favor to their asymmetric counterparts, which behave more like functions.
* *Stackfullness*. Stackfull coroutines maintain a callstack and have an ability to yield from nested function calls. Stackless coroutines lack this ability, but have lower memory footprint.
* *First-class/constrained* constructs.  
  First-class coroutines can be manipulated by programmer explicitly (e.g. in Lua or Kotlin).   
  Constrained implementations of coroutines are baked into some language-level constructs (e.g. generators in Python, goroutines in Go) and cannot be used by programmer for other purposes.


Implementations of coroutines, which pass control to some *scheduler* (instead of passing to other coroutines), are called **fibers**. This is an implementation of green threading with *cooperative scheduling* (due to the properties of coroutines). Fibers are also often referred to as *cooperative light-weight threads*.

*Useful links*:
* [More on coroutines classificaiton](http://www.inf.puc-rio.br/~roberto/docs/MCC15-04.pdf): the paper provides detailed coroutine overview and claims coroutines to be as expressive as one-shot continuations.
### Actors

### Bonus: continuations

## Syntax
### Callbacks
### Futures and async/await
### Communicating sequential processes (CSP)
### Message passing
### Monad comprehensions 

## Points of comparison
### Types of multitasking
### Cancellation
### Structured concurrency

# Part 2. Implementations
## Erlang/Elixir
## C#
## Haskell
## JVM and Loom
## Scala
## Kotlin
## Clojure
## Go
## JavaScript
## Python
## C++
## Rust

# Summary