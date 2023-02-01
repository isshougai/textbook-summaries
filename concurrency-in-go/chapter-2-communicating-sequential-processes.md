# **Modeling Your Code: Communicating Sequential Processes**

## The Difference Between Concurrency and Parallelism
*Concurrency is a property of the code; parallelism is a property of the running program*
- Code on a single machine will always be sequential, on multi core *may* be parallel
  - *We do not write parallel code, only concurrent code that we hope will be run in parallel*.
  - Parallelism is a property of the *runtime* of the program, not the code.
- It is possible, even desirable, to be ignorant of whether concurrent code is actually running in parallel.
  - Made possible by the layers of abstraction laying beneath the program's model
- Parallelism is a function of time, or context.
  - As we move down the stack of abstraction, the problem of modeling things concurrently becomes more difficult.


## Communicating Sequential Processes
*Hoarse* asserted that inputs and outputs needed to be considered language primitives. Most popular languages favor sharing and synchronizing access to the memory to CSP's message-passing style. 

Goroutines free us from having to think about our problem space in terms of parallelism. No need to worry about threads or workers. Advancements in parallelism will also improve the performance of Go programs.

## Go's Philosophy on Concurrency
CSP influenced a large part of Go's design, but Go also supports more traditional means of writing concurrent code through memory access synchronization and primitives that follow that technique. Deciding between CSP style or primitives should be to use whichever is most expressive and/or most simple.

![decision tree](https://pegasuswang.readthedocs.io/zh/latest/golang/concurrency-in-go/decision_tree.png)

