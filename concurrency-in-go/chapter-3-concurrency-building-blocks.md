# **Go's Concurrency Building Blocks**

## **Goroutines**
*A Goroutine is a function that is running concurrently alongside other code.*

```
Place go keyword before a function

func main() {
    go sayHello()
    // continue doing other things
}

func sayHello() {
    fmt.Println("hello")
}
```
```
Anonymous function

go func() {
    fmt.Println("hello")
}()
// continue doing other things
```
```
Assign function to variable

sayHello := func() {
    fmt.Println("hello")
}
go sayHello()
// continue doing other things

```
----------

### How do Goroutines actually work?
They are not OS threads, and not exactly green threads (threads that are managed by a language's runtime). They are a higher level of abstraction known as *coroutines*
- Coroutines are concurrent subroutines (functions, closures, or methods in Go) that are *nonpreemptive*
  - Cannot be interrupted, but have multiple points throughout which allow for suspension or reentry

### M:N Scheduler
Go's mechanism for hosting goroutines, where it maps M green threads to N OS threads. Goroutines are then scheduled onto green threads.
- When more goroutines than green threads, the scheduler h andles the distribution of the goroutines.

### Fork-Join Model
**Fork**: at any point in the program, it can split off a child branch of execution to be run concurrently with its parent

**Join**: at some point in the future, concurrent branches of execution join back together. 

*go statement in Go is how a fork is performed, forked threads of execution are goroutines.* 

To create a join point, we have to synchronize the main goroutine and the forked goroutine. 
```
var wg sync.WaitGroup
sayHello := func() {
    defer wg.Done()
    fmt.Println("hello")
}
wg.Add(1)
go sayHello()
wg.Wait() <<< join point
```

### Closures
Closures close around the lexical scope they are created in, thereby capturing variables.
```
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(salutation)
    }()
}
wg.Wait()

good day
good day
good day
```
Above example, goroutine is running a closure that has closed over the iteration variable salutation. If loop exits before goroutines begin, this means the salutation variable falls out of scope. Will the goroutine be accessing memory that has potentially been garbage collected?
- Go runtime is observant enough to know that a reference to the salutation variable is still being held, and therefore will transfer the memory to the heap so that the goroutines can continue to access it. 

The proper way to write the loop to pass a copy of salutation into the closure
```
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(salutation string) {
        defer wg.Done()
        fmt.Println(salutation)
    }(salutation)
}
wg.Wait()
```
----------
## **The** sync **Package**
Contains the concurrency primitives that are most useful for low-level memory access synchronization. 

### WaitGroup
Wait for a set of concurrent operations to complete when we don't care about the result of the concurrent operation, or there are other means of collecting their results. If neither are true, usage of channels and select statement is recommended instead.
```
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    fmt.Println("1st goroutine sleeping...")
    time.Sleep(1)
}()

wg.Add(1)
go func() {
    defer wg.Done()
    fmt.Println("2nd goroutine sleeping...")
    time.Sleep(2)
}()

wg.Wait()
fmt.Println("All goroutines complete.")
```

### Mutex and RWMutex
