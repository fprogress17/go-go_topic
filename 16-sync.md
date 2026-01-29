# Go `sync` Package — Complete Guide

## Table of Contents
- [Overview](#overview)
- [WaitGroup](#waitgroup)
- [Mutex & RWMutex](#mutex--rwmutex)
- [Once](#once)
- [sync.Map](#syncmap)
- [sync.Pool](#syncpool)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)

---

## Overview

The `sync` package provides **low-level primitives** for synchronizing goroutines. Use them when **shared mutable state** or **waiting for goroutines** is involved. Channels communicate; mutexes protect.

| Primitive | Use case |
|-----------|----------|
| `WaitGroup` | Wait for N goroutines to finish |
| `Mutex` | Exclusive access to shared data |
| `RWMutex` | Many readers OR one writer |
| `Once` | Run init exactly once |
| `sync.Map` | Concurrent map (specific use cases) |
| `sync.Pool` | Reuse expensive objects |

---

## WaitGroup

Use when you spawn **multiple goroutines** and need to **wait for all** before continuing.

### API

```go
var wg sync.WaitGroup

wg.Add(n)   // add n outstanding goroutines (often 1 per goroutine)
wg.Done()   // decrement by 1 (call when goroutine finishes)
wg.Wait()   // block until counter is 0
```

### Basic usage

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
            fmt.Printf("goroutine %d done\n", id)
        }(i)
    }

    wg.Wait()
    fmt.Println("all done")
}
```

### Rules

- **Call `Add` before starting the goroutine** (or at least before `Wait`), typically from the same goroutine that waits.
- **Call `Done` exactly once per `Add(1)`** — use `defer wg.Done()` inside the goroutine to avoid leaks.
- **Don’t reuse a WaitGroup** until `Wait` has returned. Create a new one for each “round.”

---

## Mutex & RWMutex

Use when **multiple goroutines read/write shared data**. Mutex gives exclusive access; RWMutex allows many readers or one writer.

### Mutex

```go
var mu sync.Mutex
var count int

func inc() {
    mu.Lock()
    defer mu.Unlock()
    count++
}

func get() int {
    mu.Lock()
    defer mu.Unlock()
    return count
}
```

- **`Lock`** / **`Unlock`**: exclusive access. Always pair them; `defer mu.Unlock()` right after `Lock` to avoid deadlocks on panic.

### RWMutex

```go
var rw sync.RWMutex
var cache map[string]int

func read(key string) (int, bool) {
    rw.RLock()
    defer rw.RUnlock()
    v, ok := cache[key]
    return v, ok
}

func write(key string, val int) {
    rw.Lock()
    defer rw.Unlock()
    if cache == nil {
        cache = make(map[string]int)
    }
    cache[key] = val
}
```

- **`RLock`** / **`RUnlock`**: shared read access. Multiple readers can hold it; writers block.
- **`Lock`** / **`Unlock`**: exclusive write. Use for updates.

Use **RWMutex** when reads are frequent and writes rare; otherwise **Mutex** is enough.

---

## Once

Ensures a function runs **exactly once**, across all goroutines.

```go
var (
    once sync.Once
    db   *Database
)

func getDB() *Database {
    once.Do(func() {
        db = connectDB()
    })
    return db
}
```

- **`Do(f)`**: runs `f` once; other callers block until it finishes. Good for lazy init, config loading, etc.

---

## sync.Map

A **concurrent map** optimized when:

- Keys are **stable** (few overwrites/deletes), **or**
- Per-goroutine keys (e.g. goroutine-local caches).

For general shared maps, a **plain `map` + `Mutex` (or `RWMutex`)** is usually simpler and enough.

### API

```go
var m sync.Map

m.Store(key, value)       // set
v, ok := m.Load(key)      // get
v, ok := m.LoadOrStore(key, value)  // get or set
m.Delete(key)
m.Range(func(k, v any) bool {
    // iterate; return false to stop
    return true
})
```

### Example

```go
var cache sync.Map

func set(id string, val int) {
    cache.Store(id, val)
}

func get(id string) (int, bool) {
    v, ok := cache.Load(id)
    if !ok {
        return 0, false
    }
    return v.(int), true
}

func main() {
    set("a", 1)
    set("b", 2)
    if v, ok := get("a"); ok {
        fmt.Println(v) // 1
    }
    cache.Range(func(k, v any) bool {
        fmt.Println(k, v)
        return true
    })
}
```

---

## sync.Pool

**Reuses** temporary objects to reduce allocations. Good for **short‑lived, per‑request** buffers (e.g. `[]byte`, parsed structs).

### API

```go
var pool = sync.Pool{
    New: func() any { return make([]byte, 0, 4096) },
}

buf := pool.Get().([]byte)
defer pool.Put(buf)
buf = buf[:0]
// use buf...
```

- **`Get()`**: take from pool (or run `New` if empty).
- **`Put(x)`**: return `x` to pool. Don’t hold references after `Put`.

### Example

```go
var bufferPool = sync.Pool{
    New: func() any { return make([]byte, 0, 4096) },
}

func getBuffer() []byte {
    return bufferPool.Get().([]byte)
}

func putBuffer(b []byte) {
    b = b[:0]
    bufferPool.Put(b)
}
```

---

## Deep Examples

### Example 1: Parallel fetch with WaitGroup

```go
func fetchAll(urls []string) ([]string, error) {
    var wg sync.WaitGroup
    var mu sync.Mutex
    var errs []error
    results := make([]string, len(urls))

    for i, u := range urls {
        wg.Add(1)
        go func(i int, u string) {
            defer wg.Done()
            resp, err := http.Get(u)
            if err != nil {
                mu.Lock()
                errs = append(errs, err)
                mu.Unlock()
                return
            }
            defer resp.Body.Close()
            b, _ := io.ReadAll(resp.Body)
            mu.Lock()
            results[i] = string(b)
            mu.Unlock()
        }(i, u)
    }

    wg.Wait()
    if len(errs) > 0 {
        return nil, errs[0]
    }
    return results, nil
}
```

### Example 2: Cache with RWMutex

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]interface{}
}

func (c *Cache) Get(k string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[k]
    return v, ok
}

func (c *Cache) Set(k string, v interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.items == nil {
        c.items = make(map[string]interface{})
    }
    c.items[k] = v
}
```

### Example 3: Lazy singleton with Once

```go
var (
    configOnce sync.Once
    config     *Config
)

func LoadConfig() *Config {
    configOnce.Do(func() {
        config = readConfigFromEnv()
    })
    return config
}
```

### Example 4: Worker pool (WaitGroup + channels)

```go
func runWorkers(jobs <-chan int, n int) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := range jobs {
                process(j)
            }
        }()
    }
    wg.Wait()
}
```

---

## Best Practices

- **Prefer channels** for ownership transfer and explicit communication; use **mutexes** when you need to protect shared in‑memory state.
- **Always `defer Unlock`** (or `RUnlock`) right after `Lock` (or `RLock`).
- Use **RWMutex** only when reads dominate; otherwise **Mutex** is simpler.
- **WaitGroup**: `Add` before starting goroutines; `defer Done()` inside each; don’t reuse before `Wait` returns.
- **Once**: use for one‑time init; keep `Do` side effects minimal.
- **sync.Map**: use only when it matches the “stable keys” or “per‑goroutine” pattern; otherwise `map` + mutex.
- **sync.Pool**: use for frequently allocated, short‑lived objects; don’t hold references after `Put`.

---

## Summary

| Primitive | Purpose |
|-----------|---------|
| `WaitGroup` | Wait for N goroutines |
| `Mutex` | Exclusive access to shared data |
| `RWMutex` | Many readers or one writer |
| `Once` | One‑time init |
| `sync.Map` | Concurrent map (special cases) |
| `sync.Pool` | Reuse temporary objects |

Together with **goroutines** and **channels**, `sync` completes the core concurrency toolkit in Go.
