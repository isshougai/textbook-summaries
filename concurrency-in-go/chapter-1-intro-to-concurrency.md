# **An Introduction to Concurrency**

## Moore's Law, Web Scale, and the Mess We're In
- Companies began to investigate alternative ways to increase computing power in anticipation of the slowdown in the rate of Moore's law prediction
- Soon faced the limits of Amdahl's law: gains are bounded by how much of the program must be written in a sequential manner
- [Spigot algorithms](https://mathworld.wolfram.com/SpigotAlgorithm.html) reduces a problem to *embarrassingly parallel*, which means to easily divide problems into parallel tasks

### *Embarrassingly parallel problems should scale horizontally*
- Cloud computing makes horizontal scaling extremely accessible
- Introduces new problems
  1. Resource provision
  2. Communication between instances
  3. Aggregating and storing results
  4. **Modeling code concurrently**

### *Web Scale Software*
- Rolling upgrades
- Elastic horizontally scalable architecture
- Geographic distribution
- Increased complexity in comprehension and fault tolerance

----------


## Why is Concurrency Hard?

### ***Race Conditions***
When two or more operations must execute in the correct order, but the program has not been written so that this order is guaranteed to be maintained. Occurs most of the time as a *data race*, where one concurrent operation attempts to read a variable, while at some undetermined time another concurrent operation is attempting to write to the same variable.
- Introducing sleeps can be helpful in debugging, but it is not the solution. Should always target logical correctness

### ***Atomicity***
Within the context that something is operating, it is indivisible, or uninterruptible. The atomicity of an operation can change depending on the currently defined scope. When thinking about atomicity, the context, or scope in which the operation needs to be considered in needs to be defined. 
- i++ as an example
  - Retrieve, increment, and store are each atomic alone, but the combination may not be atomic
  - *Combining atomic operations does not necessarily produce a larger atomic operation*
- Something atomic is implicitly safe within concurrent contexts
- Techniques exists to force atomicity, the art becomes determining which areas of code needs to be atomic, and at what level of granularity.

### ***Memory Access Synchronization***
A *critical section* of a program is a section that needs exclusive access to a shared resource. One way to synchronize access to memory between critical sections is to use *locks*.
- Synchronizing access doesn't solve data races or logical correctness, a non deterministic program will still be non deterministic. 
- Calls to locks slows down programs, as program pauses for a period of time

### ***Deadlocks, Livelocks, and Starvation***
These issues concern ensuring program has something useful to do at all times.

#### *Deadlock*
All concurrent processes are waiting on one another. A deadlocked program will never recover without outside intervention.
- *Coffman Conditions* - conditions that must be present for deadlocks to arise
  1. *Mutual Exclusion*: A concurrent process holds exclusive rights to a resource at any one time
  2. *Wait For Condition*: A concurrent process must simultaneously hold a resource and be waiting for an additional resource
  3. *No Preemtion*: A resource held by a concurrent process can only be released by that process
  4. *Circular Wait*: A concurrent process (P1) must be waiting on a chain of other concurrent processes (P2), which in turn are waiting on it (P1)

#### *Livelock*
Programs that are actively performing concurrent operations, but these operations do nothing to move the state of the program forward. Livelocks are a subset of a *starvation*

#### *Starvation*
Any situation where a concurrent process cannot get all the resources it needs to perform work. Starvation usually implies that there are one or more greedy concurrent process that are unfairly preventing one or more concurrent processes from accomplishing work as efficiently as possible, or maybe at all. 
- Recording and sampling metrics (logging when work is accomplished) to determine if rate of work is as high as expected is a good way to identify starvation.

### ***Determining Concurrency Safety***
Developers themselves pose the biggest challenge of developing concurrent code. It can be difficult to know how to utilize concurrent code without comments. Comments should cover:
  1. Who is responsible for the concurrency?
  2. How is the problem space mapped onto concurrency primitives?
  3. Who is responsible for the synchronization?

```
// CalculatePi calculates digits of Pi between the begin and end
// place.
//
// Internally, CalculatePi will create FLOOR((end-begin)/2) concurrent
// processes which recursively call CalculatePi. Synchronization of
// writes to pi are handled internally by the Pi struct.
func CalculatePi(begin, end int64, pi *Pi)
```
----------

## **Simplicity in the Face of Complexity**
Go as a language has made concurrency significantly easier.

### ***Garbage Collector***
Go's GC is concurrent and low-latency. Go's GC pauses are generally between 10 - 100 microseconds

### ***Runtime***
Go's runtime automatically handles multiplexing concurrent operations onto operating system threads (go-routines).

### ***Concurrency Primitives***
Go's channel primitive provides a composable, concurrent-safe way to communicate between concurrent processes.