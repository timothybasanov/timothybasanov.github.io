---
title:  "Guice Singletons Parallelization"
---

Guice is a Java library for dependency injection. It's a very widespread one and a lot of servers written in Java use it. Surprisingly until recently *all* singleton objects created by Guice were created sequentially.

{{ excerpt_separator }}

It had two downsides:

  * When several tests are running within the same JVM all their respective singletons are created sequentially. One slow (DB-related as an example) singleton could bring a whole test suite to a halt.
  * Guice itself can create deadlocks in such a way that it could not be prevented from the library's client's code.

[Reimplementing Guice singleton scope](https://github.com/google/guice/commits?author=timofeyb) with a support for a proper multi-threading, turned out to be surprisingly hard.

> Getting a deadlock with Guice's singletons: Thread `TA` creates singleton 
  object `A`, using injector `IA`. Thread `T2` takes lock `L` and tries to 
  create a singleton object `B` using injector `IB`. Thread `TA` tries to
  take lock `L`. Now both threads wait for each other, we have a deadlock.
  This is a *very* puzzling behaviour as threads use their individual 
  injectors and even create different objects.


# Naïve way to fix it with lock per injector

Giving each injector its own lock for singletons provides a simple way
to parallelize common scenario when different injectors are used for
different tests running in parallel.

[Making Singleton's creation lock less coarse.](https://github.com/google/guice/commit/d7aa953d088f4955789051414bcd6134437afa17)

As it turns out Guice supports parent-child relations for injectors,
one lock per such a tree is used.


# Proper way to fix it with a lock per singleton

Giving each singleton its own lock allows full parallelization.

> Reimplementing Guice's singleton scope with a support for a proper
  multi-threading, turned out to be surprisingly hard.

As it turns out Guice supports dependency cycles and resolves it automatically using dynamic proxy objects when a flag is on for a specific injector. This increases complexity of a solution dramatically as dependency cycle can span several threads and it *will* guarantee a deadlock.

Solving this problem turned out to be very nontrivial. Guice guarantees that no proxy object would be created when no dependency cycle for the object under creation is detected. Therefore we have to detect lock cycles ourselves. It's tricky as there are no common Java library that does it: even Google Commons (Guava) detect only static locks.

[Implement more granular locks for a Singleton scope in Guice](https://github.com/google/guice/commit/5e6c93348c4250012801b6e41753789d760f06e4)

As a side effect [CycleDetectingLock](https://github.com/google/guice/blob/master/core/src/com/google/inject/internal/CycleDetectingLock.java) was written. It allows dynamic lock cycle detection with `O(N)` time where `N` is number of threads that are part of dependency tree.

It worked well and negatively affected less than 0.01% projects in terms of performance or functionality. It was surprising considering ~1K lines of heavily multi-threaded code in this change.
