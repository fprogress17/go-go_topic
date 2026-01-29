# Go Context - Deep Dive

## Table of Contents
- [What is Context?](#what-is-context)
- [http.Request and Context](#httprequest-and-context)
- [context.WithTimeout](#contextwithtimeout)
- [Context Types: Background, TODO, WithCancel, WithDeadline](#context-types-background-todo-withcancel-withdeadline)
- [context.WithValue — Request-Scoped Metadata](#contextwithvalue--request-scoped-metadata)
- [Context Propagation (Parent-Child)](#context-propagation-parent-child)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)

---

## What is Context?

In Go, `context` is a built-in package that acts like a **"suitcase"** for a request’s lifecycle. It carries:

1. **Cancellation signals** — Tell functions to stop (user closed tab, timeout, explicit cancel).
2. **Deadlines** — "Finish by this time or stop."
3. **Request-scoped values** — Metadata like user ID, request ID, auth token.

Think of it as a **signal flare**. If a user closes their browser tab, the context sends a signal to your database and background tasks: *"Nobody is listening anymore. Stop and free resources."*

### The Context Interface

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

- **`Deadline()`** — When the context will expire (if any).
- **`Done()`** — Channel that closes when the context is cancelled; workers `select` on this to stop.
- **`Err()`** — `context.Canceled` or `context.DeadlineExceeded` after cancellation.
- **`Value(key)`** — Retrieve request-scoped metadata.

---

## http.Request and Context

### Does `http.Request` have a context?

**Yes.** Every `http.Request` has a built-in context.

- **When is it created?** The `net/http` server creates it when the request starts.
- **When is it cancelled?** It is cancelled when:
  1. The client closes the connection (refresh, stop, navigate away).
  2. The server finishes sending the response.

`r.Context()` returns that request-scoped context. **Always** use it (or a child of it) for any I/O triggered by that request.

### Minimal example

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Request-scoped context
    
    result, err := fetchFromDB(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}
```

---

## context.WithTimeout

`context.WithTimeout` creates a **child context** that cancels when:

1. The given duration elapses, or  
2. The parent context (e.g. `r.Context()`) is cancelled.

So both "user left" and "took too long" stop your work.

### Example: Slow database with timeout

```go
func searchHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()  // Always release timer resources
    
    query := r.URL.Query().Get("q")
    results, err := searchDatabase(ctx, query)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
            return
        }
        if errors.Is(err, context.Canceled) {
            return  // Client gone
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(results)
}

func searchDatabase(ctx context.Context, query string) ([]string, error) {
    resultCh := make(chan []string)
    
    go func() {
        time.Sleep(5 * time.Second)  // Simulates slow DB
        resultCh <- []string{"a", "b", "c"}
    }()
    
    select {
    case res := <-resultCh:
        return res, nil
    case <-ctx.Done():
        return nil, ctx.Err()  // DeadlineExceeded or Canceled
    }
}
```

### Visualizing the flow

| Step | Action | Status |
|------|--------|--------|
| **0s** | Request arrives; `ctx` with 3s timeout starts. | **Running** |
| **0s** | `searchDatabase` starts a goroutine. | **Working** |
| **3s** | `context.WithTimeout` hits its limit. | **DeadlineExceeded** |
| **3s** | `select` in `searchDatabase` sees `<-ctx.Done()`. | **Returns error** |
| **3s** | Handler sends `408 Request Timeout` to the client. | **Done** |
| **5s** | Slow goroutine finishes; no one is listening. | **Cleanup** |

Without context, that goroutine would keep running after the client left, wasting CPU and memory.

---

## Context Types: Background, TODO, WithCancel, WithDeadline

### `context.Background` and `context.TODO`

```go
// Root context, never cancelled. Use as top-level (e.g. main, tests).
ctx := context.Background()

// Placeholder when you're not yet sure which context to use.
ctx := context.TODO()
```

Use `Background` for main, init, tests, or the root of a context tree. Use `TODO` only temporarily.

### `context.WithCancel` — Manual cancellation

You call `cancel()` to signal "stop."

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go worker(ctx)
    
    time.Sleep(2 * time.Second)
    cancel()  // Signal worker to stop
    time.Sleep(500 * time.Millisecond)
}

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Stopping:", ctx.Err())  // context.Canceled
            return
        default:
            fmt.Println("Working...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

### `context.WithDeadline` — Cancel at a specific time

```go
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("Work completed")
case <-ctx.Done():
    fmt.Println("Stopped:", ctx.Err())  // context.DeadlineExceeded
}
```

### Summary

| Function | Use case |
|----------|----------|
| `Background` | Root context (main, tests). |
| `TODO` | Temporary placeholder. |
| `WithCancel` | Manual stop (e.g. user click "Stop"). |
| `WithTimeout` | Stop after a duration. |
| `WithDeadline` | Stop at a specific time. |

---

## context.WithValue — Request-Scoped Metadata

Use `WithValue` to pass request-scoped data (request ID, user ID, auth token) through the call chain. **Avoid** using it for optional function parameters; use explicit arguments instead.

### Custom key type (recommended)

```go
type contextKey string

const (
    keyRequestID contextKey = "requestID"
    keyUserID    contextKey = "userID"
)

func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := fmt.Sprintf("req-%d", time.Now().UnixNano())
        ctx := context.WithValue(r.Context(), keyRequestID, requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    requestID := r.Context().Value(keyRequestID).(string)
    fmt.Fprintf(w, "Request ID: %s\n", requestID)
}
```

### Safe extraction helper

```go
func getRequestID(ctx context.Context) (string, bool) {
    v := ctx.Value(keyRequestID)
    if v == nil {
        return "", false
    }
    id, ok := v.(string)
    return id, ok
}

func handler(w http.ResponseWriter, r *http.Request) {
    if id, ok := getRequestID(r.Context()); ok {
        log.Printf("Request ID: %s", id)
    }
}
```

### Passing through layers

```go
func fetchUser(ctx context.Context, userID string) (*User, error) {
    // Pass ctx to DB, HTTP client, etc.
    return db.GetUser(ctx, userID)
}

func fetchOrders(ctx context.Context, userID string) ([]Order, error) {
    return db.GetOrders(ctx, userID)
}

func dashboardHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := getUserIdFromSession(r)
    
    user, err := fetchUser(ctx, userID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    orders, err := fetchOrders(ctx, userID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]any{"user": user, "orders": orders})
}
```

If the client disconnects, `r.Context()` is cancelled, so both `fetchUser` and `fetchOrders` can stop early when they respect `ctx`.

---

## Context Propagation (Parent-Child)

Contexts form a **tree**. Cancelling a parent cancels all its children; cancelling a child does **not** cancel the parent.

```go
func main() {
    parent, parentCancel := context.WithCancel(context.Background())
    defer parentCancel()
    
    child1, cancel1 := context.WithCancel(parent)
    defer cancel1()
    
    child2, cancel2 := context.WithCancel(parent)
    defer cancel2()
    
    go worker("child1", child1)
    go worker("child2", child2)
    
    time.Sleep(2 * time.Second)
    parentCancel()  // Cancels both child1 and child2
    time.Sleep(500 * time.Millisecond)
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

---

## Deep Examples

### Example 1: HTTP handler with DB + external API and timeout

```go
func orderHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    orderID := r.URL.Query().Get("id")  // or extract from path
    
    order, err := db.GetOrder(ctx, orderID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Enrich with external API, same ctx
    if err := enrichWithInventory(ctx, order); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(order)
}

func enrichWithInventory(ctx context.Context, order *Order) error {
    url := "https://api.example.com/inventory/" + order.ProductID
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    // ... decode and attach to order
    return nil
}
```

If the client disconnects or the 5s timeout fires, both DB and HTTP calls can stop.

### Example 2: Multi-stage pipeline with context

```go
func runPipeline(ctx context.Context, input []int) ([]int, error) {
    stage1 := stageMultiply(ctx, input, 2)
    stage2 := stageFilter(ctx, stage1, func(n int) bool { return n%2 == 0 })
    return stageCollect(ctx, stage2)
}

func stageMultiply(ctx context.Context, in []int, factor int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range in {
            select {
            case out <- n * factor:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func stageFilter(ctx context.Context, in <-chan int, keep func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if !keep(n) {
                continue
            }
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func stageCollect(ctx context.Context, in <-chan int) ([]int, error) {
    var result []int
    for {
        select {
        case n, ok := <-in:
            if !ok {
                return result, nil
            }
            result = append(result, n)
        case <-ctx.Done():
            return result, ctx.Err()
        }
    }
}
```

Cancelling `ctx` (e.g. via timeout) stops all stages.

### Example 3: Graceful shutdown with context

```go
func main() {
    mux := http.NewServeMux()
    // mux.HandleFunc(...)
    srv := &http.Server{Addr: ":8080", Handler: mux}
    
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, os.Interrupt, syscall.SIGTERM)
    <-sig
    
    log.Println("Shutting down...")
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Shutdown:", err)
    }
    log.Println("Done.")
}
```

### Example 4: Concurrent calls with shared timeout

```go
func fetchDashboard(ctx context.Context) (*Dashboard, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    var users []User
    var orders []Order
    var err1, err2 error
    var wg sync.WaitGroup
    
    wg.Add(2)
    go func() {
        defer wg.Done()
        users, err1 = db.GetUsers(ctx)
    }()
    go func() {
        defer wg.Done()
        orders, err2 = db.GetOrders(ctx)
    }()
    
    wg.Wait()
    if err1 != nil {
        return nil, err1
    }
    if err2 != nil {
        return nil, err2
    }
    
    return &Dashboard{Users: users, Orders: orders}, nil
}
```

Both DB calls share the same `ctx`. If the timeout fires or the parent is cancelled, both can stop as soon as `db` respects `ctx`.

---

## Best Practices

1. **First parameter**  
   Use `context.Context` as the first parameter:  
   `func DoSomething(ctx context.Context, name string)`.

2. **Always defer cancel**  
   For `WithCancel`, `WithTimeout`, or `WithDeadline`:
   ```go
   ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
   defer cancel()
   ```

3. **Pass context through**  
   Pass `ctx` into any I/O: DB, HTTP, gRPC, etc.

4. **Check `ctx.Done()` in loops**  
   In long-running loops, `select` on `<-ctx.Done()` and return when it’s closed.

5. **Don’t store context in structs**  
   Pass it explicitly per call instead of keeping it as a field.

6. **Use `WithValue` only for request-scoped data**  
   Not for optional parameters or general configuration.

---

## Common Mistakes

```go
// Don't store context in a struct
type Server struct {
    ctx context.Context  // BAD
}

// Don't pass nil; use context.TODO() if needed
DoSomething(nil, "x")  // BAD

// Don't ignore ctx in I/O
func fetch(id string) (*User, error) {
    return db.GetUser(context.Background(), id)  // BAD: ignores request context
}

// Don't forget defer cancel
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
// ... use ctx ...
cancel()  // BAD: no defer, can leak if early return
```

---

## Key Takeaways

- **Propagation:** Pass `ctx` as the first argument to any function that does I/O.
- **Efficiency:** Context ensures goroutines and I/O stop when the client leaves or a timeout hits, saving CPU and memory.
- **Safety:** Always `defer cancel()` when using `WithCancel`, `WithTimeout`, or `WithDeadline`.
- **Convention:** Use `context.Context` as the first parameter:  
  `func DoSomething(ctx context.Context, name string)`.
