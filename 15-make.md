# Go `make` Built-in — Complete Guide

## Table of Contents
- [Overview](#overview)
- [Syntax](#syntax)
- [make vs new vs Composite Literals](#make-vs-new-vs-composite-literals)
- [Slices](#slices)
- [Maps](#maps)
- [Channels](#channels)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)

---

## Overview

`make` is a built-in function that **allocates and initializes** slices, maps, and channels. It returns an **initialized value** (not a pointer). Memory is zeroed where applicable.

- **Slices:** Pre-allocate length and/or capacity.
- **Maps:** Create an empty, ready-to-use map (optional capacity hint).
- **Channels:** Create unbuffered or buffered channels; **required** for channels (no literal alternative).

---

## Syntax

| Type | Form | Meaning |
|------|------|--------|
| Slice | `make([]T, n)` | Slice of `T`, length `n`, capacity `n` |
| Slice | `make([]T, n, c)` | Slice of `T`, length `n`, capacity `c` (`n ≤ c`) |
| Map | `make(map[K]V)` | Map, no capacity hint |
| Map | `make(map[K]V, hint)` | Map with space for ~`hint` elements |
| Channel | `make(chan T)` | Unbuffered channel of `T` |
| Channel | `make(chan T, n)` | Buffered channel of `T`, buffer size `n` |

**Rules:**

- `n` and `c` must be non-negative integers.
- For slices, if both `n` and `c` are used, **`n` must not exceed `c`**.
- `make` works only with slices, maps, and channels.

---

## make vs new vs Composite Literals

### `make` vs `new`

```go
// make: allocates + initializes. Use for slices, maps, channels.
s := make([]int, 5)      // []int, len 5, cap 5, zeroed
m := make(map[string]int) // empty map, ready to use
c := make(chan int)      // unbuffered channel

// new: allocates, zeroes, returns *T. Use for pointers to any type.
p := new(int)            // *int, points to 0
sp := new([]int)         // *[]int, points to nil slice (not useful for typical slice use)
```

- **Slices, maps, channels:** use `make` (or composite literals where applicable).
- **Pointers to structs, primitives, etc.:** use `new` or `&T{}`.

### `make` vs composite literals

```go
// Slice — both valid
a := make([]int, 0, 10)  // len 0, cap 10, pre-allocated
b := []int{}             // len 0, cap 0, no pre-allocation

// Map — both valid
m1 := make(map[string]int)
m2 := map[string]int{}

// Channel — make only (no literal)
ch := make(chan int)
```

Use `make` when you need **length**, **capacity**, or **buffer size**. Use literals for ad-hoc, small slices/maps.

---

## Slices

### Forms

```go
make([]T, n)     // length n, capacity n. Elements are zero value of T.
make([]T, n, c)  // length n, capacity c. n must be ≤ c.
```

### `make([]T, n)`

Creates a slice of **length** `n` and **capacity** `n`. Elements are zeroed.

```go
s := make([]int, 5)
fmt.Println(s)           // [0 0 0 0 0]
fmt.Println(len(s), cap(s)) // 5 5

s[0] = 10
s[4] = 40
fmt.Println(s)           // [10 0 0 0 40]
```

Use when you know the length and will index directly (e.g. `s[i] = x`).

### `make([]T, n, c)`

Length `n`, capacity `c` (`n ≤ c`). First `n` elements zeroed; extra capacity reserved for `append` without reallocation.

```go
s := make([]int, 0, 10)
fmt.Println(len(s), cap(s)) // 0 10

s = append(s, 1, 2, 3)
fmt.Println(s, len(s), cap(s)) // [1 2 3] 3 10
```

Use when you'll `append` and have a rough upper bound on size.

### Zero values

```go
type Item struct {
    ID   int
    Name string
}

items := make([]Item, 3)
// items[0], items[1], items[2] are Item zero value: {0 ""}
```

### Length vs capacity

```go
s := make([]int, 2, 5)
// len=2, cap=5, s = [0 0]

s = append(s, 10)        // [0 0 10], len=3, cap=5
s = append(s, 20, 30)    // [0 0 10 20 30], len=5, cap=5
s = append(s, 40)        // realloc; len=6, cap typically 10
```

### Invalid uses

```go
make([]int, 5, 3)   // invalid: 5 > 3
make([]int, -1)     // invalid: negative length
make([]int, 0, -1)  // invalid: negative capacity
```

---

## Maps

### Forms

```go
make(map[K]V)      // empty map, default capacity
make(map[K]V, n)   // empty map, hint for ~n elements (avoid early rehashing)
```

### Basic usage

```go
m := make(map[string]int)
m["a"] = 1
m["b"] = 2
fmt.Println(m["a"], len(m)) // 1 2

m2 := make(map[string]int, 100)  // hint: ~100 entries
```

### Why use a hint?

The hint **does not** fix the size; it helps the runtime avoid repeated rehashing as you add elements. Use it when you know approximate size.

```go
// Good: you'll add ~100 keys
users := make(map[int]User, 100)

// Fine: small or unknown size
config := make(map[string]string)
```

### Zero value

A map zero value is `nil`. You **cannot** add keys to a `nil` map.

```go
var m map[string]int  // nil
m["k"] = 1            // panic: assignment to entry in nil map

m = make(map[string]int)  // ok
m["k"] = 1
```

---

## Channels

### Forms

```go
make(chan T)    // unbuffered: send blocks until receive
make(chan T, n) // buffered: send blocks only when buffer full
```

### Unbuffered `make(chan T)`

- Send blocks until another goroutine receives.
- Receive blocks until another goroutine sends.
- Synchronization and “handoff” semantics.

```go
ch := make(chan int)

go func() {
    ch <- 42
}()

v := <-ch
fmt.Println(v) // 42
```

### Buffered `make(chan T, n)`

- Sends non-blocking while buffer has space.
- Receives non-blocking while buffer has values.
- Decouples producer and consumer briefly.

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
// next send would block until someone receives

fmt.Println(<-ch, <-ch, <-ch) // 1 2 3
```

### Channel direction (optional)

```go
var sendOnly chan<- int = make(chan int, 1)
var recvOnly <-chan int = make(chan int, 1)
```

---

## Deep Examples

### Example 1: Batch processing with pre-allocated slice

```go
func processBatch(input []int, batchSize int) [][]int {
    n := (len(input) + batchSize - 1) / batchSize
    batches := make([][]int, 0, n)  // avoid reallocation

    for i := 0; i < len(input); i += batchSize {
        end := i + batchSize
        if end > len(input) {
            end = len(input)
        }
        batches = append(batches, input[i:end])
    }
    return batches
}

func main() {
    data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    batches := processBatch(data, 3)
    fmt.Println(batches) // [[1 2 3] [4 5 6] [7 8 9] [10]]
}
```

### Example 2: Slice with known length, fill by index

```go
func squares(n int) []int {
    out := make([]int, n)
    for i := 0; i < n; i++ {
        out[i] = i * i
    }
    return out
}

func main() {
    fmt.Println(squares(5)) // [0 1 4 9 16]
}
```

### Example 3: Map with capacity hint (known key set)

```go
func countWords(words []string) map[string]int {
    counts := make(map[string]int, len(words))
    for _, w := range words {
        counts[w]++
    }
    return counts
}

func main() {
    words := []string{"a", "b", "a", "c", "b", "a"}
    fmt.Println(countWords(words)) // map[a:3 b:2 c:1]
}
```

### Example 4: Worker pool with buffered channels

```go
func workerPool(numWorkers, numJobs int) {
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    for w := 0; w < numWorkers; w++ {
        go func() {
            for j := range jobs {
                results <- j * 2
            }
        }()
    }

    for j := 0; j < numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    for i := 0; i < numJobs; i++ {
        fmt.Println(<-results)
    }
}

func main() {
    workerPool(3, 9)
}
```

### Example 5: Matrix (2D slice) with `make`

```go
func newMatrix(rows, cols int) [][]float64 {
    m := make([][]float64, rows)
    for i := range m {
        m[i] = make([]float64, cols)
    }
    return m
}

func main() {
    m := newMatrix(2, 3)
    m[0][1] = 1.5
    m[1][2] = 2.5
    fmt.Println(m) // [[0 1.5 0] [0 0 2.5]]
}
```

### Example 6: Object pool (reusable slice buffer)

```go
var bufferPool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 4096)
    },
}

func getBuffer() []byte {
    return bufferPool.Get().([]byte)
}

func putBuffer(b []byte) {
    b = b[:0]
    bufferPool.Put(b)
}
```

### Example 7: Unbuffered channel as mutex / signal

```go
func main() {
    done := make(chan struct{})

    go func() {
        time.Sleep(time.Second)
        close(done)
    }()

    <-done
    fmt.Println("done")
}
```

### Example 8: Semaphore (concurrency limit) via buffered channel

```go
func main() {
    maxConcurrent := 3
    sem := make(chan struct{}, maxConcurrent)
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        sem <- struct{}{}  // acquire (blocks if 3 already in use)
        go func(id int) {
            defer wg.Done()
            defer func() { <-sem }()  // release
            fmt.Printf("worker %d\n", id)
            time.Sleep(time.Second)
        }(i)
    }

    wg.Wait()
}
```

The buffered channel holds at most `maxConcurrent` tokens; goroutines acquire before work and release when done. At most 3 run at a time.

---

## Best Practices

### Slices

- Use `make([]T, 0, cap)` when you'll `append` and know an approximate max size.
- Use `make([]T, n)` when you have a fixed length and assign by index.
- Avoid `make([]T, n)` then only appending — you get `n` zeroed elements you may not use.

### Maps

- Use `make(map[K]V, hint)` when you know approximate size to reduce rehashing.
- Always use `make` (or a literal) before writing; never write to a `nil` map.

### Channels

- Use **unbuffered** channels for strict handoff and synchronization.
- Use **buffered** channels when you want to decouple send/receive or batch work (e.g. worker pools).
- `make` is required for channels; there is no literal form.

### General

- Prefer `make` over `new` for slices, maps, and channels.
- Use composite literals for small, fixed slices/maps when you don’t need pre-allocation.

---

## Summary

| Target | Typical use |
|--------|-------------|
| `make([]T, n)` | Fixed-length slice, assign by index |
| `make([]T, 0, c)` | Append-heavy slice, reserve capacity |
| `make(map[K]V)` | Map, unknown or small size |
| `make(map[K]V, n)` | Map, ~`n` elements expected |
| `make(chan T)` | Unbuffered channel, sync |
| `make(chan T, n)` | Buffered channel, producer–consumer |

`make` initializes slices, maps, and channels so they are ready to use; for channels it is the only way to create them.
