# Go `defer`, `panic`, `recover` — Complete Guide

## Table of Contents
- [Overview](#overview)
- [defer](#defer)
- [panic](#panic)
- [recover](#recover)
- [Order of Execution](#order-of-execution)
- [defer + return](#defer--return)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)

---

## Overview

- **`defer`** — Schedule a function call when the surrounding function returns (normal return, panic, or runtime exit). Used for cleanup (close files, unlock mutexes, etc.).
- **`panic`** — Stop normal execution and begin unwinding the stack, running defers. Use sparingly for “impossible” or unreachable states.
- **`recover`** — Capture a panic **only inside a deferred function**. Returns the value passed to `panic`; otherwise `nil`. Used to turn panics into errors or to log and re‑panic.

---

## defer

### Syntax

```go
defer f(args)        // f runs when surrounding function returns
defer func() { ... }()
```

- Arguments are **evaluated immediately**; the **call runs** at function exit.
- Multiple defers run in **LIFO** order (last deferred, first run).

### Basic usage

```go
func main() {
    defer fmt.Println("third")
    defer fmt.Println("second")
    defer fmt.Println("first")
    fmt.Println("body")
}
// body
// first
// second
// third
```

### Cleanup patterns

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()
// use f...
```

```go
mu.Lock()
defer mu.Unlock()
// use shared state...
```

### Arguments evaluated at defer site

```go
func main() {
    i := 0
    defer fmt.Println(i)  // i evaluated now → 0
    i = 1
}
// 0
```

```go
func main() {
    i := 0
    defer func() { fmt.Println(i) }()  // closure over i
    i = 1
}
// 1
```

---

## panic

### Syntax

```go
panic(v)
```

- **`v`** is any value (often `string` or `error`). It becomes the value `recover` returns.
- Panic **unwinds the stack**, running defers. If nothing recovers, the program exits with a stack trace.

### When it runs

```go
func a() {
    defer fmt.Println("a defer")
    b()
    fmt.Println("a after b")
}

func b() {
    defer fmt.Println("b defer")
    panic("oops")
    fmt.Println("b after panic")  // never runs
}

func main() {
    a()
}
// b defer
// a defer
// panic: oops  (+ stack trace)
```

### Typical use

- **Unreachable code** (e.g. after exhaustive switch).
- **Programming bugs** you don’t want to handle as normal errors (e.g. nil pointer, failed invariant).
- **Library** signaling “this must not happen” to the caller.

Prefer **returning `error`** for expected failures (I/O, validation, etc.).

---

## recover

### Syntax

```go
func() {
    if x := recover(); x != nil {
        // handle panic: x is value passed to panic
    }
}()
```

- **`recover()`** is valid **only inside a deferred function**. Elsewhere it always returns `nil` and has no effect.
- Returns the value passed to `panic`; returns `nil` if no panic or panic was already recovered.

### Basic recovery

```go
func main() {
    defer func() {
        if v := recover(); v != nil {
            fmt.Println("recovered:", v)
        }
    }()
    panic("boom")
}
// recovered: boom
```

### Recover then re‑panic

```go
defer func() {
    if v := recover(); v != nil {
        log.Printf("panic: %v", v)
        panic(v)  // re-panic after logging
    }
}()
```

### Recover only in the right place

```go
func bad() {
    recover()       // no effect: not in defer
}

func good() {
    defer func() { recover() }()
    panic(1)
}
```

---

## Order of Execution

1. Function runs normally (or until `panic`).
2. **Defers** run in LIFO order.
3. If a defer calls **`recover`** and there was a panic, `recover` returns the panic value and unwinding stops there.
4. If no recover (or panic again), unwinding continues to the caller, and so on.

---

## defer + return

Return values are **set** before defers run. Defers can **change** named return values by modifying them.

### Named returns

```go
func f() (n int) {
    defer func() { n++ }()
    return 0
}
// returns 1: defer runs after "return 0" sets n, then n++
```

### Unnamed returns

```go
func f() int {
    n := 0
    defer func() { n++ }()
    return n  // 0 is copied to return slot; defer mutates local n only
}
// returns 0
```

### Practical pattern

```go
func do() (err error) {
    f, err := os.Open("f")
    if err != nil {
        return err
    }
    defer func() {
        if closeErr := f.Close(); closeErr != nil && err == nil {
            err = closeErr
        }
    }()
    // use f...
    return nil
}
```

---

## Deep Examples

### Example 1: Mutex + defer

```go
func (s *Service) Do() error {
    s.mu.Lock()
    defer s.mu.Unlock()
    // ... work ...
    return nil
}
```

### Example 2: HTTP handler with panic recovery

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if v := recover(); v != nil {
                log.Printf("panic: %v\n%s", v, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Example 3: Unreachable default

```go
func must(err error) {
    if err != nil {
        panic(err)
    }
}

f, err := os.Open("file")
must(err)
defer f.Close()
```

### Example 4: Defer and loop

```go
for _, path := range paths {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()  // all defers run at end of function
    // use f...
}
```

Prefer a **helper** so each file is closed before the next is opened:

```go
for _, path := range paths {
    err := func() error {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        // use f...
        return nil
    }()
    if err != nil {
        return err
    }
}
```

---

## Best Practices

- **Use `defer` for cleanup** (Close, Unlock, etc.) so it runs even on panic or early returns.
- **`defer` mutex unlock** immediately after lock: `defer mu.Unlock()`.
- **Remember**: defer arguments are evaluated at the defer line; use a **closure** if you need the value at exit.
- **Use `panic` rarely** — for bugs and unreachable code, not for expected errors.
- **`recover` only in deferred functions**; often at the edge (e.g. HTTP middleware, main goroutine).
- Prefer **returning `error`** over panic for recoverable failure paths.
- Avoid **defer in hot loops** if you care about allocation; use explicit cleanup or a helper.

---

## Summary

| Keyword | Purpose |
|--------|---------|
| `defer` | Run function at surrounding return; LIFO; args evaluated at defer. |
| `panic` | Unwind stack; defers run; `recover` can capture. |
| `recover` | Capture panic value only inside defer; else `nil`. |

Use **defer** for cleanup, **panic** sparingly, and **recover** at boundaries to turn panics into logs or errors.
