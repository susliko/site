---
title: "Concurrent approaches to concurrency"
date: 2020-06-01T15:17:42+03:00
draft: true
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
    - [Continuations, Coroutines, Generators](#continuations-coroutines-generators)
    - [Coroutines + Scheduling = Fibers](#coroutines--scheduling--fibers)
    - [Actors](#actors)
    - [MISC: green and lightweight threads](#misc-green-and-lightweight-threads)
  - [Syntax](#syntax)
    - [Callbacks](#callbacks)
    - [Futures and async/await](#futures-and-asyncawait)
    - [Communicating sequential processes (CSP)](#communicating-sequential-processes-csp)
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
Every program consists of a bunch of CPU *computations* and syncronous *waiting* for something to become available (e.g. lock, result of a network request).  
Instead of keeping CPU idle while waiting, we could start another computation *concurrently* (read "concurrent" as "with overlapping execution time bounds"): this would allow to accomplish more in less time.  
A set of techniques which enables concurrent execution of computations is called *concurrent programming*. 

### Waiting optimization
All waiting operations might be split into two categories: synchronisation and I/O.  
*Synchronisation* (language-level):
* locks 
* blocking queues

*I/O (OS-level)*:
* network socket reads/writes
* disk reads/writes 
* DNS lookups
* other system calls (see [types of system calls](https://www.tutorialspoint.com/different-types-of-system-calls))

There are two ways to deal with waiting operations: either constantly poll for some completion condition or "subscribe" to that event via a callback which will resume a computation when called. The second approach gives the desired opportunity for CPU to work on other computations while previous computation might be paused until the waiting finishes.  
Most of the listed types of blocking might be implemented efficiently with that idea in mind:
* efficient implementations of synchronisation primitives (locks, blocking queues) are handled by the language designers
* asynchronous socket operations: [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) (Linux), [kqueue](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2) (FreeBSD), [iocp](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports) (Windows)
* asynchronous disk I/O: [ui_uring](https://kernel.dk/io_uring.pdf) (Linux), [aio](https://www.freebsd.org/cgi/man.cgi?query=aio&sektion=4&manpath=freebsd-release-ports) (FreeBSD), [overlapped I/O](https://docs.microsoft.com/en-us/windows/win32/sync/synchronization-and-overlapped-input-and-output?redirectedfrom=MSDN) (Windows)
* there is also a bunch of solutions for asynchronous DNS-lookups (e.g. Linux [qdns](https://linux.die.net/man/3/qdns))

### Multitasking
Besides waiting optimisation concurrent programming ships an opportunity to progress in execution of multiple CPU-bound concurrent tasks independently of the number of involved system threads.  
This approach is called *multitasking*.  
There are two types of it:
* *Preemptive multitasking*. In this model each computation might be paused by the scheduler and resumed at some time later.  
  Examples: system threads, goroutines (Go), IO monad (Haskell).
* *Cooperative multitasking*. This model allows only voluntary pauses, which allow other computations to get their CPU time. Voluntary pausing is often called *yielding*.  
  Examples: coroutines (Lua, Kotlin)

While preemptive approach doesn't allow long-running tasks to occupy all the CPU time, those programs are more prone to bugs, since the execution might be preempted at any point. Therefore a very careful synchronisation is required to prevent various bugs like data races and race conditions.  
Cooperative approach is widespread in many languages since it is sufficient for most of applications (non latency-critical ones) and allows smaller amount of inappropriate execution scenarios to happen.

The next part will cover the known ways to implement multitasking.

Useful links:
* [list](https://gist.github.com/jboner/2841832) of latency numbers for RAM/CPU/disk/network operations
* [race condition vs data race](https://blog.regehr.org/archives/490)
## Units of execution
### System processes and threads
The lowest abstraction representing an executing program is a *system process*.  
Each process has its *address space* (some allocated memory to store all possible addresses to devices, files on disk, main memory locations, remote hosts). Processes are scheduled by a system scheduler. The scheduling is preemptive, in order to guarantee that all running processes will have a chance to progress.

Many programs require parallel execution (e.g. browser should be able to brouse several pages in time). For this purpose a process can spawn multiple *system threads* -- some objects that share the address space of their process, but have own call stacks. To schedule a process means to schedule all of its threads.  
The goal of parallel execution might also be achieved via creating multiple processes, but inter-process communication (via pipes or sockets) is slower than communication between threads sharing the same address space.

Why not to stop here? Why do we keep hearing about "lightweight threads" implementations in different languages?  
The reason for this is the time which takes to start a system thread and the amount of memory it occupies: the default size of Linux thread call stack is 2 megabytes. It turns out that a system thread is a relatively precious resource, as one cannot spawn more than a couple of thousands of them on a single machine.  
But at the same time one might want to have millions of small concurrent tasks accomplished, hence spawning one system thread per one task is not affordable. 

Useful links:
* [Linux processes and threads](https://www.slashroot.in/difference-between-process-and-thread-linux) 
* [Completely Fair Scheduler -- default Linux scheduler](https://opensource.com/article/19/2/fair-scheduling-linux)
### Continuations, Coroutines, Generators 
### Coroutines + Scheduling = Fibers
### Actors
### MISC: green and lightweight threads

## Syntax
### Callbacks
### Futures and async/await
### Communicating sequential processes (CSP)
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