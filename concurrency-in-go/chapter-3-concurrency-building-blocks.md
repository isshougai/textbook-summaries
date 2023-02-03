# **Go's Concurrency Building Blocks**

## **Goroutines**
*A Goroutine is a function that is running concurrently alongside other code.*

```go
Place go keyword before a function

func main() {
    go sayHello()
    // continue doing other things
}

func sayHello() {
    fmt.Println("hello")
}
```
```go
Anonymous function

go func() {
    fmt.Println("hello")
}()
// continue doing other things
```
```go
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
```go
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
```go
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
```go
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

## WaitGroup
Wait for a set of concurrent operations to complete when we don't care about the result of the concurrent operation, or there are other means of collecting their results. If neither are true, usage of channels and select statement is recommended instead.
```go
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

## Mutex and RWMutex
*Mutual Exclusion* is a way to guard critical sections of your program that requires exclusive access to a shared resource. Whereas channels share memory by communicating, Mutex shares memory by creating a convention developers must follow to synchronize access to the memory. 
```go
var count int
var lock sync.Mutex

increment := func() {
    lock.Lock()
    defer lock.Unlock()
    count++
    fmt.Printf("Incrementing: %d\n", count)
}

decrement := func() {
    lock.Lock()
    defer lock.Unlock()
    count--
    fmt.Printf("Decrementing: %d\n", count)
}

// Increment
var arithmetic sync.WaitGroup
for i := 0; i <= 5; i++ {
    arithmetic.Add(1)
    go func() {
        defer arithmetic.Done()
        increment()
    }()
}

// Decrement
for i := 0; i <= 5; i++ {
    arithmetic.Add(1)
    go func() {
        defer arithmetic.Done()
        decrement()
    }()
}

arithmetic.Wait()
fmt.Println("Arithmetic complete.")
```
To reduce bottleneck of program, one strategy is to reduce the cross-section of the critical section. When not all concurrent processes need to read *and* write to this memory, can take advantage of ***sync.RWMutex***. An arbitrary number of readers can hold a reader lock so long as nothing else is holding a writer lock.
```go
producer := func(wg *sync.WaitGroup, l sync.Locker) {
    defer wg.Done()
    for i := 5; i > 0; i-- {
        l.Lock()
        l.Unlock()
        time.Sleep(1)
    }
}

observer := func(wg *sync.WaitGroup, l sync.Locker) {
    defer wg.Done()
    l.Lock()
    defer l.Unlock()
}

test := func(count int, mutex, rwMutex sync.Locker) time.Duration {
    var wg sync.WaitGroup
    wg.Add(count+1)
    beginTestTime := time.Now()
    go producer(&wg, mutex)
    for i := count; i > 0; i-- {
        go observer(&wg, rwMutex)
    }

    wg.Wait()
    return time.Since(beginTestTime)
}

tw := tabwriter.NewWriter(os.Stdout, 0, 1, 2, ' ', 0)
defer tw.Flush()

var m sync.RWMutex
fmt.Fprintf(tw, "Readers\tRWMutext\tMutex\n")
for i := 0; i < 20; i++ {
    count := int(math.Pow(2, float64(i)))
    fmt.Fprintf(
        tw,
        "%d\t%v\t%v\n",
        count,
        test(count, &m, m.RLocker()),
        test(count, &m, &m),
    )
}
```
## Cond
*"...a rendezvous point for goroutines waiting for or announcing the occurrence of an event."*

An "event" is any arbitrary signal between two or more goroutines that carries no information other than the fact that it has occured. 
```go
c := sync.NewCond(&sync.Mutex{})
c.L.Lock()
for conditionTrue() == false {
    c.Wait()
}
c.L.Unlock()
```
Call to Wait() doesn't just block, it suspends the current goroutine, allowing other goroutines to run on OS thread. 
```go
c := sync.NewCond(&sync.Mutex{})
queue := make([]interface{}, 0, 10)

removeFromQueue := func(delay time.Duration) {
    time.Sleep(delay)
    c.L.Lock()
    queue = queue[1:]
    fmt.Println("Removed from queue")
    c.L.Unlock()
    c.Signal()
}

for i := 0; i < 10; i++{
    c.L.Lock()
    for len(queue) == 2 {
        c.Wait()
    }
    fmt.Println("Adding to queue")
    queue = append(queue, struct{}{})
    go removeFromQueue(1*time.Second)
    c.L.Unlock()
}
```
Signal is one of two methods that the Cond type provides for notifying goroutines blocked on a Wait call that the condition has been triggered. The other method is Broadcast. Internally, runtime maintains FIFO list of goroutines waiting to be signaled; Signal finds the one waiting for the longest and notifies. Broadcast sends a signal to *all* goroutines. 
```go
type Button struct {
    Clicked *sync.Cond
}
button := Button{ Clicked: sync.NewCond(&sync.Mutex{}) }

subscribe := func(c *sync.Cond, fn func()) {
    var goroutineRunning sync.WaitGroup
    goroutineRunning.Add(1)
    go func() {
        goroutineRunning.Done()
        c.L.Lock()
        defer c.L.Unlock()
        c.Wait()
        fn()
    }()
    goroutineRunning.Wait()
}

var clickRegistered sync.WaitGroup
clickRegistered.Add(3)
subscribe(button.Clicked, func() {
    fmt.Println("Maximizing window.")
    clickRegistered.Done()
})
subscribe(button.Clicked, func() {
    fmt.Println("Displaying annoying dialog box!")
    clickRegistered.Done()
})
subscribe(button.Clicked, func() {
    fmt.Println("Mouse clicked.")
    clickRegistered.Done()
})

button.Clicked.Broadcast()

clickRegistered.Wait()
```