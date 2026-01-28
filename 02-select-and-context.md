# Go Select & Context - Deep Dive

## Table of Contents
- [SELECT Statement](#select-statement)
- [CONTEXT Package](#context-package)
- [Advanced Patterns](#advanced-patterns)
- [Best Practices](#best-practices)

---

## SELECT Statement

### Basic Concept

`select` lets a goroutine wait on multiple channel operations simultaneously. It's like a switch statement for channels.

### 1. Simple Select

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    // Goroutine 1: sends to ch1 after 1 second
    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "from channel 1"
    }()

    // Goroutine 2: sends to ch2 after 2 seconds
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from channel 2"
    }()

    // Select: whichever channel is ready first
    select {
    case msg1 := <-ch1:
        fmt.Println("Received:", msg1)  // This executes (faster)
    case msg2 := <-ch2:
        fmt.Println("Received:", msg2)
    }
    
    // Output: "Received: from channel 1"
}
```

**Key points:**
- Only one case executes (the first ready)
- If multiple cases are ready, random selection
- Blocks until at least one case can proceed

### 2. Select with Default (Non-blocking)

```go
func main() {
    ch := make(chan string)
    
    select {
    case msg := <-ch:
        fmt.Println("Received:", msg)
    default:
        fmt.Println("No data available")  // Executes immediately
    }
    
    // Output: "No data available"
}
```

**Use case:** Check channel without blocking

### 3. Select in Loop (Common Pattern)

```go
func worker(jobs <-chan int, done <-chan bool) {
    for {
        select {
        case job := <-jobs:
            fmt.Println("Processing job:", job)
            time.Sleep(1 * time.Second)
        case <-done:
            fmt.Println("Worker stopping")
            return  // Exit goroutine
        }
    }
}

func main() {
    jobs := make(chan int, 5)
    done := make(chan bool)
    
    go worker(jobs, done)
    
    // Send some jobs
    for i := 1; i <= 3; i++ {
        jobs <- i
    }
    
    time.Sleep(4 * time.Second)
    done <- true  // Signal shutdown
    time.Sleep(100 * time.Millisecond)
}
```

### 4. Timeout Pattern

```go
func main() {
    ch := make(chan string)
    
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()
    
    select {
    case res := <-ch:
        fmt.Println("Got:", res)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout!")  // Executes (1s < 2s)
    }
}
```

`time.After(d)` returns a channel that sends a value after duration `d`.

### 5. Multiple Operations in Select

```go
func main() {
    send := make(chan int)
    receive := make(chan int)
    
    go func() {
        receive <- 42  // Send to receive channel
    }()
    
    select {
    case send <- 1:
        fmt.Println("Sent to send")
    case val := <-receive:
        fmt.Println("Received:", val)  // This executes
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout")
    }
}
```

### 6. Advanced: Select with Multiple Receives (Fan-In)

```go
func fanIn(ch1, ch2 <-chan string) <-chan string {
    out := make(chan string)
    
    go func() {
        defer close(out)
        for {
            select {
            case msg, ok := <-ch1:
                if !ok {
                    ch1 = nil  // Disable this case
                    continue
                }
                out <- msg
            case msg, ok := <-ch2:
                if !ok {
                    ch2 = nil  // Disable this case
                    continue
                }
                out <- msg
            }
            
            // Exit when both channels closed
            if ch1 == nil && ch2 == nil {
                return
            }
        }
    }()
    
    return out
}

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        for i := 0; i < 3; i++ {
            ch1 <- fmt.Sprintf("ch1-%d", i)
            time.Sleep(100 * time.Millisecond)
        }
        close(ch1)
    }()
    
    go func() {
        for i := 0; i < 3; i++ {
            ch2 <- fmt.Sprintf("ch2-%d", i)
            time.Sleep(150 * time.Millisecond)
        }
        close(ch2)
    }()
    
    merged := fanIn(ch1, ch2)
    for msg := range merged {
        fmt.Println(msg)
    }
}
```

### 7. Select with Send Operations

```go
func main() {
    ch := make(chan int, 1)
    
    select {
    case ch <- 42:
        fmt.Println("Sent 42")
    case val := <-ch:
        fmt.Println("Received:", val)
    default:
        fmt.Println("No operation ready")
    }
}
```

---

## CONTEXT Package

### What is Context?

Context carries:
- **Cancellation signals** (stop work)
- **Deadlines** (time limits)
- **Request-scoped values** (metadata)

### 1. Basic Context Types

```go
// Background: root context (never cancelled)
ctx := context.Background()

// TODO: placeholder when unsure which context to use
ctx := context.TODO()
```

### 2. WithCancel - Manual Cancellation

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
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Goroutine cancelled:", ctx.Err())
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()
    
    time.Sleep(2 * time.Second)
    cancel()  // Trigger cancellation
    time.Sleep(1 * time.Second)
}
```

**Output:**
```
Working...
Working...
Working...
Working...
Goroutine cancelled: context canceled
```

### 3. WithTimeout - Automatic Cancellation

```go
func main() {
    // Automatically cancelled after 2 seconds
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()  // Always call cancel to free resources
    
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Task completed")
    case <-ctx.Done():
        fmt.Println("Timeout:", ctx.Err())  // This executes
    }
}
```

### 4. WithDeadline - Cancel at Specific Time

```go
func main() {
    deadline := time.Now().Add(2 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Completed")
    case <-ctx.Done():
        fmt.Println("Deadline exceeded:", ctx.Err())
    }
}
```

### 5. Context Propagation (Parent-Child)

```go
func main() {
    // Parent context
    parentCtx, parentCancel := context.WithCancel(context.Background())
    
    // Child contexts
    childCtx1, cancel1 := context.WithCancel(parentCtx)
    childCtx2, cancel2 := context.WithCancel(parentCtx)
    defer cancel1()
    defer cancel2()
    
    go worker("Worker 1", childCtx1)
    go worker("Worker 2", childCtx2)
    
    time.Sleep(2 * time.Second)
    
    parentCancel()  // Cancels BOTH children!
    
    time.Sleep(1 * time.Second)
}

func worker(name string, ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s stopped: %v\n", name, ctx.Err())
            return
        default:
            fmt.Printf("%s working...\n", name)
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

**Key:** Cancelling parent cancels all children, but not vice versa.

### 6. Context with Values (Metadata)

```go
type key string

func main() {
    ctx := context.Background()
    
    // Add values
    ctx = context.WithValue(ctx, key("userID"), "12345")
    ctx = context.WithValue(ctx, key("requestID"), "abc-def")
    
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    userID := ctx.Value(key("userID")).(string)
    requestID := ctx.Value(key("requestID")).(string)
    
    fmt.Printf("Processing request %s for user %s\n", requestID, userID)
}
```

⚠️ **Warning:** Only use for request-scoped data, not for passing optional parameters!

### 7. Real-World: HTTP Server with Context

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/search", searchHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()
    
    query := r.URL.Query().Get("q")
    
    // Simulate database query
    results, err := searchDatabase(ctx, query)
    if err != nil {
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    fmt.Fprintf(w, "Results: %v\n", results)
}

func searchDatabase(ctx context.Context, query string) ([]string, error) {
    // Simulate slow query
    resultChan := make(chan []string)
    
    go func() {
        time.Sleep(5 * time.Second)  // Simulates slow DB
        resultChan <- []string{"result1", "result2"}
    }()
    
    select {
    case results := <-resultChan:
        return results, nil
    case <-ctx.Done():
        return nil, ctx.Err()  // Returns context.DeadlineExceeded
    }
}
```

**Test:**
```bash
# This will timeout after 3 seconds
curl "http://localhost:8080/search?q=golang"
```

### 8. Complex Example: Multi-stage Pipeline with Context

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Stage 1: Generate numbers
    numbers := generate(ctx, 1, 2, 3, 4, 5)
    
    // Stage 2: Square them
    squared := square(ctx, numbers)
    
    // Stage 3: Filter even numbers
    evens := filter(ctx, squared)
    
    // Consume results
    for result := range evens {
        fmt.Println("Result:", result)
    }
    
    fmt.Println("Done!")
}

func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
                time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
            case <-ctx.Done():
                fmt.Println("Generate cancelled")
                return
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
                time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
            case <-ctx.Done():
                fmt.Println("Square cancelled")
                return
            }
        }
    }()
    return out
}

func filter(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                select {
                case out <- n:
                case <-ctx.Done():
                    fmt.Println("Filter cancelled")
                    return
                }
            }
        }
    }()
    return out
}
```

---

## Advanced Patterns

### Pattern 1: Context with Request ID

```go
type requestIDKey struct{}

func withRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey{}, id)
}

func getRequestID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey{}).(string)
    return id, ok
}

func handler(ctx context.Context) {
    id, _ := getRequestID(ctx)
    fmt.Printf("Request ID: %s\n", id)
}
```

### Pattern 2: Context with Timeout and Retry

```go
func doWithRetry(ctx context.Context, fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            err := fn()
            if err == nil {
                return nil
            }
            if i < maxRetries-1 {
                time.Sleep(time.Duration(i+1) * time.Second)
            }
        }
    }
    return fmt.Errorf("max retries exceeded")
}
```

### Pattern 3: Context Tree Visualization

```go
func printContextTree(ctx context.Context, indent string) {
    fmt.Printf("%sContext Type: %T\n", indent, ctx)
    
    if deadline, ok := ctx.Deadline(); ok {
        fmt.Printf("%sDeadline: %v\n", indent, deadline)
    }
    
    if err := ctx.Err(); err != nil {
        fmt.Printf("%sError: %v\n", indent, err)
    }
    
    // Check for parent context
    if parent := getParent(ctx); parent != nil {
        printContextTree(parent, indent+"  ")
    }
}
```

---

## Best Practices

### ✅ DO:

```go
// Pass context as first parameter
func DoSomething(ctx context.Context, arg string) error

// Always call cancel() to free resources
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()

// Check ctx.Done() in loops
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // do work
    }
}

// Use context in HTTP handlers
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // use ctx
}
```

### ❌ DON'T:

```go
// Don't store context in struct
type Server struct {
    ctx context.Context  // BAD!
}

// Don't pass nil context
DoSomething(nil, "arg")  // Use context.TODO() instead

// Don't use context.Value for function parameters
// Use explicit function arguments

// Don't ignore context cancellation
func bad(ctx context.Context) {
    // Missing ctx.Done() check
    time.Sleep(10 * time.Second)  // BAD!
}
```

---

## Summary Table

| Feature | Use Case | Example |
|---------|----------|---------|
| `select` | Wait on multiple channels | Network + timeout |
| `select` with `default` | Non-blocking check | Try receive without waiting |
| `context.WithCancel` | Manual cancellation | User clicks "Stop" |
| `context.WithTimeout` | Auto-cancel after duration | API calls with timeout |
| `context.WithDeadline` | Cancel at specific time | Batch job must finish by midnight |
| `context.WithValue` | Request-scoped metadata | User ID, trace ID |

---

## Common Use Cases

### Use Case 1: Database Query with Timeout

```go
func queryWithTimeout(ctx context.Context, query string) ([]Row, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    return db.QueryContext(ctx, query)
}
```

### Use Case 2: Multiple API Calls with Cancellation

```go
func fetchMultiple(ctx context.Context, urls []string) ([]Response, error) {
    results := make([]Response, len(urls))
    errChan := make(chan error, len(urls))
    
    for i, url := range urls {
        go func(idx int, u string) {
            resp, err := http.Get(u)
            if err != nil {
                errChan <- err
                return
            }
            results[idx] = resp
        }(i, url)
    }
    
    for i := 0; i < len(urls); i++ {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case err := <-errChan:
            if err != nil {
                return nil, err
            }
        }
    }
    
    return results, nil
}
```

### Use Case 3: Graceful Shutdown

```go
func gracefulShutdown(server *http.Server) {
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    
    <-sigChan
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    server.Shutdown(ctx)
}
```

---

Both `select` and `context` are fundamental to writing robust, cancellable, concurrent Go code! 🚀
