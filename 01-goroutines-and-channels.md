# Go Goroutines and Channels - Complete Guide

## Table of Contents
- [Goroutines (`go func()`)](#goroutines-go-func)
- [Channels (`<-Done()`)](#channels--done)
- [Complete Examples](#complete-examples)
- [Best Practices](#best-practices)

---

## Goroutines (`go func()`)

### Basic Syntax

```go
go func() {
    // code runs concurrently
}()
```

**Breaking it down:**
- `go` - keyword that launches a goroutine (lightweight thread)
- `func() { ... }` - anonymous function definition
- `()` - immediately invokes the function (in a new goroutine)

### What Happens

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    fmt.Println("1")
    go func() {
        fmt.Println("2")
    }()
    fmt.Println("3")
    time.Sleep(100 * time.Millisecond)
    
    // Output might be: 1, 3, 2
    // The goroutine runs concurrently!
}
```

### With Parameters

```go
package main

import "fmt"

func main() {
    name := "Alice"
    go func(n string) {
        fmt.Println("Hello", n)
    }(name)  // Pass variables as arguments
    
    time.Sleep(100 * time.Millisecond)
}
```

**Why pass parameters?** Avoids closure issues with variable capture.

### Advanced: Goroutine with Return Values

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    resultChan := make(chan int)
    
    go func() {
        time.Sleep(1 * time.Second)
        resultChan <- 42
    }()
    
    result := <-resultChan
    fmt.Println("Result:", result)  // Result: 42
}
```

### Multiple Goroutines

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Duration(id) * 100 * time.Millisecond)
            fmt.Printf("Goroutine %d finished\n", id)
        }(i)
    }
    
    wg.Wait()
    fmt.Println("All goroutines finished")
}
```

---

## Channels (`<-Done()`)

### Channel Basics

```go
ch := make(chan int)

// Send to channel
ch <- 42

// Receive from channel
value := <-ch

// Receive and discard value
<-ch
```

### `<-Done()` Specifically

```go
<-cycle.Done()
```

This means:
- **Wait (block)** until the `Done()` channel receives a value or closes
- `Done()` typically returns a `<-chan struct{}` (receive-only channel)
- Used for signaling that something is complete

### Common Pattern (context)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        // do work
        time.Sleep(2 * time.Second)
        cancel()  // signals done
    }()
    
    <-ctx.Done()  // waits here until cancel() is called
    fmt.Println("Context cancelled!")
}
```

### Your Code Explained

```go
go func(){
    err := sec.ListenAndServe()  // Start server (blocks)
}()

<-cycle.Done()  // Main goroutine waits here
```

**What's happening:**

1. **Main goroutine** starts a **new goroutine** to run the server
2. Server runs **concurrently** in background
3. **Main goroutine** blocks at `<-cycle.Done()`, waiting for signal
4. When `cycle` is done (cancelled/completed), main goroutine continues
5. Program can then shut down gracefully

### Visualization

```
Main goroutine          Server goroutine
     |                        |
     | --- go func() -------> |
     |                        | ListenAndServe()
     | <-cycle.Done()         | (running...)
     | (blocked/waiting)      |
     |                        |
     | <-- signal received    |
     | (continues)            |
```

---

## Complete Examples

### Example 1: HTTP Server with Graceful Shutdown

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

func main() {
    // Create a context that we control
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // WaitGroup to track all goroutines
    var wg sync.WaitGroup

    // Channel to catch OS signals (Ctrl+C, kill, etc.)
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    // Create HTTP server
    server := &http.Server{
        Addr:    ":8080",
        Handler: createHandler(),
    }

    // Goroutine 1: Run the HTTP server
    wg.Add(1)
    go func() {
        defer wg.Done()
        log.Println("🚀 Server starting on :8080")
        
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("❌ Server error: %v", err)
        }
        log.Println("✅ Server stopped accepting connections")
    }()

    // Goroutine 2: Background worker (simulates periodic tasks)
    wg.Add(1)
    go func() {
        defer wg.Done()
        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                log.Println("⚙️  Background task running...")
            case <-ctx.Done():
                log.Println("🛑 Background worker shutting down")
                return
            }
        }
    }()

    // Goroutine 3: Monitor server health
    wg.Add(1)
    go func() {
        defer wg.Done()
        healthCheck := time.NewTicker(5 * time.Second)
        defer healthCheck.Stop()

        for {
            select {
            case <-healthCheck.C:
                resp, err := http.Get("http://localhost:8080/health")
                if err != nil {
                    log.Printf("⚠️  Health check failed: %v", err)
                } else {
                    resp.Body.Close()
                    log.Println("💚 Health check passed")
                }
            case <-ctx.Done():
                log.Println("🛑 Health monitor shutting down")
                return
            }
        }
    }()

    // Main goroutine: Wait for shutdown signal
    log.Println("⏳ Press Ctrl+C to shutdown...")
    <-sigChan  // BLOCKS here until signal received
    log.Println("\n🔔 Shutdown signal received!")

    // Cancel context (signals all goroutines to stop)
    cancel()

    // Give server time to finish existing requests
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer shutdownCancel()

    log.Println("⏳ Gracefully shutting down server...")
    if err := server.Shutdown(shutdownCtx); err != nil {
        log.Printf("❌ Shutdown error: %v", err)
    }

    // Wait for all goroutines to finish
    wg.Wait()
    log.Println("✅ All goroutines finished. Goodbye!")
}

func createHandler() http.Handler {
    mux := http.NewServeMux()

    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        log.Printf("📨 Request: %s %s", r.Method, r.URL.Path)
        time.Sleep(1 * time.Second) // Simulate work
        fmt.Fprintf(w, "Hello! Time: %s\n", time.Now().Format(time.RFC3339))
    })

    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "OK")
    })

    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        log.Println("⏰ Slow request started...")
        time.Sleep(5 * time.Second)
        fmt.Fprint(w, "Done!")
        log.Println("✅ Slow request completed")
    })

    return mux
}
```

### Example 2: Worker Pool Pattern

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // Start 3 workers
    var wg sync.WaitGroup
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }
    
    // Send 9 jobs
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)  // No more jobs
    
    // Wait for workers to finish
    go func() {
        wg.Wait()
        close(results)  // Close results when all done
    }()
    
    // Collect results
    for result := range results {
        fmt.Println("Result:", result)
    }
}

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for job := range jobs {
        fmt.Printf("Worker %d started job %d\n", id, job)
        time.Sleep(time.Second)
        fmt.Printf("Worker %d finished job %d\n", id, job)
        results <- job * 2
    }
}
```

**Output:**
```
Worker 1 started job 1
Worker 2 started job 2
Worker 3 started job 3
Worker 1 finished job 1
Result: 2
Worker 1 started job 4
...
```

### Example 3: Pipeline Pattern

```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    // Stage 1: Generate numbers
    numbers := generate(1, 2, 3, 4, 5)
    
    // Stage 2: Square them
    squared := square(numbers)
    
    // Stage 3: Filter even numbers
    evens := filter(squared)
    
    // Consume results
    for result := range evens {
        fmt.Println("Result:", result)
    }
    
    fmt.Println("Done!")
}

func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
            time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
            time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
        }
    }()
    return out
}

func filter(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}
```

---

## Channel Operations Deep Dive

### 1. Buffered vs Unbuffered Channels

```go
// Unbuffered - blocks until receiver ready
ch1 := make(chan int)
ch1 <- 42  // BLOCKS until someone receives

// Buffered - can hold values
ch2 := make(chan int, 3)
ch2 <- 1  // OK
ch2 <- 2  // OK
ch2 <- 3  // OK
ch2 <- 4  // BLOCKS (buffer full)
```

### 2. Channel Direction

```go
// Send-only channel
func sendOnly(ch chan<- int) {
    ch <- 42
    // <-ch  // ERROR: cannot receive from send-only channel
}

// Receive-only channel
func receiveOnly(ch <-chan int) {
    val := <-ch
    // ch <- 42  // ERROR: cannot send to receive-only channel
}

// Bidirectional (default)
func bidirectional(ch chan int) {
    ch <- 42
    val := <-ch
}
```

### 3. Close & Range

```go
jobs := make(chan int, 5)

// Producer
go func() {
    for i := 0; i < 5; i++ {
        jobs <- i
    }
    close(jobs)  // Signal "no more values"
}()

// Consumer
for job := range jobs {  // Exits when channel closed
    fmt.Println("Processing:", job)
}
```

### 4. Select Statement

```go
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case ch2 <- 42:
    fmt.Println("Sent to ch2")
case <-time.After(1 * time.Second):
    fmt.Println("Timeout!")
case <-ctx.Done():
    fmt.Println("Context cancelled")
default:
    fmt.Println("No channel ready")
}
```

---

## Best Practices

### ✅ DO:

1. **Always use WaitGroup for multiple goroutines**
```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // work
}()
wg.Wait()
```

2. **Use context for cancellation**
```go
ctx, cancel := context.WithCancel(context.Background())
go func() {
    select {
    case <-ctx.Done():
        return
    default:
        // work
    }
}()
```

3. **Close channels when done**
```go
defer close(ch)
```

4. **Use buffered channels when appropriate**
```go
ch := make(chan int, 100)  // Prevents blocking
```

### ❌ DON'T:

1. **Don't leak goroutines**
```go
// BAD: No way to stop
go func() {
    for {
        // infinite loop
    }
}()

// GOOD: Use context
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // work
        }
    }
}()
```

2. **Don't send on closed channel**
```go
close(ch)
ch <- 42  // PANIC!
```

3. **Don't forget to close channels**
```go
// BAD: Receiver waits forever
go func() {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    // Missing close(ch)
}()

// GOOD
defer close(ch)
```

---

## Key Takeaways

- `go func()` creates lightweight concurrent execution
- Channels synchronize and communicate between goroutines
- `<-ch` blocks until data available (or channel closed)
- `context.Context` provides cancellation signals
- `sync.WaitGroup` waits for multiple goroutines
- `select` multiplexes multiple channel operations
- This is the foundation of Go's concurrency model!

---

## Common Patterns

### Pattern 1: Fan-Out/Fan-In

```go
func fanOut(in <-chan int, n int) []<-chan int {
    outs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        out := make(chan int)
        go func() {
            defer close(out)
            for val := range in {
                out <- val * 2
            }
        }()
        outs[i] = out
    }
    return outs
}

func fanIn(ins ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, in := range ins {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for val := range ch {
                out <- val
            }
        }(in)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Pattern 2: Rate Limiting

```go
func rateLimit(requests <-chan int, limit time.Duration) <-chan int {
    out := make(chan int)
    ticker := time.NewTicker(limit)
    defer ticker.Stop()
    
    go func() {
        defer close(out)
        for req := range requests {
            <-ticker.C
            out <- req
        }
    }()
    
    return out
}
```

### Pattern 3: Timeout Pattern

```go
func withTimeout(fn func() error, timeout time.Duration) error {
    done := make(chan error, 1)
    
    go func() {
        done <- fn()
    }()
    
    select {
    case err := <-done:
        return err
    case <-time.After(timeout):
        return fmt.Errorf("operation timed out")
    }
}
```

---

This is the foundation of Go's powerful concurrency model! 🚀
