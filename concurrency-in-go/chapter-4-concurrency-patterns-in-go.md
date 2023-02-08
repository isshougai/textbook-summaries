# Concurrency Patterns in Go

## Confinement
Different options exists for safe operations when dealing with concurrent code. Some include
- Synchronization primitives for sharing memory (e.g., sync.Mutex)
- Synchronization via communicating (e.g., channels)

However, there are options that are implicitly safe within multiple concurrent processes:
- Immutable data
- *Data protected by confinement*

Confinement is ensuring information is only ever available from *one* concurrent process. If achieved, a concurrent program is implicitly safe and no synchronization is needed

### Ad hoc Confinement
Confinement achieved through convention
```go
data := make([]int, 4)

loopData := func(handleData chan<- int) {
    defer close(handleData)
    for i := range data {
        handleData <- data[i]
    }
}

handleData := make(chan int)
go loopData(handleData)

for num := range handleData {
    fmt.Println(num)
}
```

### Lexical Confinement
Confinement achieved using lexical scope to expose only the correct data and concurrency primitives for multiple concurrent processes to use. 
```go
chanOwner := func() <-chan int {
    results := make(chan int, 5)
    go func() {
        defer close(results)
        for i := 0; i <= 5; i++ {
            results <- i // Confines the write aspect of this channel to prevent other goroutines from writing to it
        }
    }()
    return results
}

consumer := func(results <-chan int) { // By declaring read only, confine the usage to only reads
    for result := range results {
        fmt.Printf("Received: %d\n", result)
    }
    fmt.Println("Done receiving!")
}

results := chanOwner()
consumer(results)
```
Above uses channels, which are concurrent-safe. Below uses a data structure that is not concurrent safe
```go
printData := func(wg *sync.WaitGroup, data []byte) { // doesn't close around data slice
    defer wg.Done()

    var buff bytes.Buffer
    for _, b := range data { // takes in slice of byte to operate on.
        fmt.Fprintf(&buff, "%c", b)
    }
    fmt.Println(buff.String())
}

var wg sync.WaitGroup
wg.Add(2)
data := []byte("golang")
go printData(&wg, data[:3]) //because of lexical scope, impossible to do the wrong thing
go printData(&wg, data[3:])

wg.Wait()
```
Confinement improves performance and reduces cognitive load on developers as compared to synchronization. However, it may sometimes be difficult to establish confinement, and thus have to fallback on Go concurrency primitives

----------

## The for-select Loop
```go
for { // Either loop infinitely or range over something
    select {
    // Do some work with channels
}
}
```
Scenarios where the above pattern is useful
1. Sending iteration variables out on a channel
```go
for _, s := range []string{"a", "b", "c"} {
    select {
    case <-done:
        return
    case stringStream <- s:
    }
}
```
2. Looping infinitely waiting to be stopped
```go
for {
    select {
    case <-done:
        return
    default:
    }
    // Do non-preemptable work
}

for {
    select {
    case <-done:
        return
    default:
        // Do non-preemptable work
    }
}
```

----------

## Preventing Goroutine Leaks
