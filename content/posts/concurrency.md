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
- [Part 1. Concepts](#part-1-concepts)
  - [Reasons for concurrent programming](#reasons-for-concurrent-programming)
    - [Types of blocking](#types-of-blocking)
    - [How to optimize waiting?](#how-to-optimize-waiting)
  - [Processing units: bottom to top](#processing-units-bottom-to-top)
    - [System threads and processes](#system-threads-and-processes)
    - [Continuations, Generators, Coroutines](#continuations-generators-coroutines)
    - [Coroutine + Sheduler = Fiber](#coroutine--sheduler--fiber)
    - [Actors](#actors)
    - [MISC: synonyms](#misc-synonyms)
  - [Syntax](#syntax)
    - [Callbacks](#callbacks)
    - [Futures and async/await](#futures-and-asyncawait)
    - [Communicating sequential processes (CSP)](#communicating-sequential-processes-csp)
    - [Monad comprehensions](#monad-comprehensions)
  - [Points of comparison](#points-of-comparison)
    - [Types of multitasking](#types-of-multitasking)
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
### Types of blocking
### How to optimize waiting?
## Processing units: bottom to top
### System threads and processes
### Continuations, Generators, Coroutines
### Coroutine + Sheduler = Fiber
### Actors
### MISC: synonyms
## Syntax
### Callbacks
### Futures and async/await
### Communicating sequential processes (CSP)
### Monad comprehensions 
## Points of comparison
### Types of multitasking
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
