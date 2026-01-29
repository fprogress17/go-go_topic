# Go Empty Interface & Empty Struct — Complete Guide

## Table of Contents
- [Overview](#overview)
- [Empty Interface (`any` / `interface{}`)](#empty-interface-any--interface)
- [Empty Struct (`struct{}`)](#empty-struct-struct)
- [Deep Examples](#deep-examples)
- [When to Use What](#when-to-use-what)
- [Best Practices](#best-practices)

---

## Overview

| Concept | What it is | Typical use |
|--------|------------|-------------|
| **Empty interface** | `interface{}` or `any` — no methods; **any type** satisfies it | “Unknown” or heterogeneous values, JSON, generic containers |
| **Empty struct** | `struct{}` — no fields; **zero size** | Signaling, set-as-map-keys, “no data” placeholders |

They solve different problems: **empty interface** is about **accepting any type**; **empty struct** is about **minimal values** (signals, set membership, etc.).

---

## Empty Interface (`any` / `interface{}`)

### What it is

```go
type any = interface{}  // built-in alias since Go 1.18
```

An interface with **no methods**. Every type implements it, so a value of type `any` (or `interface{}`) can hold **any** Go value.

```go
var x any
x = 42
x = "hello"
x = []int{1, 2, 3}
x = struct{ A int }{A: 1}
```

### What it isn’t

- **Not** “no type” — it’s a concrete interface type. Variables have a type; `any` is that type.
- **Not** “generic” by itself — it erases type info at the type level. You recover it via **type assertion** or **type switch**.
- **Not** a replacement for **generics** when you have a uniform type. Use `any` when you genuinely need “unknown” or mixed types.

### Type assertion

Extract the concrete type from an `any`:

```go
var v any = "hello"

s := v.(string)           // panic if v is not string
s, ok := v.(string)       // safe: ok false if not string
if ok {
    fmt.Println(s)
}
```

### Type switch

Branch on the **runtime type**:

```go
func describe(v any) {
    switch x := v.(type) {
    case nil:
        fmt.Println("nil")
    case int:
        fmt.Println("int:", x)
    case string:
        fmt.Println("string:", x)
    case []byte:
        fmt.Println("[]byte, len:", len(x))
    case error:
        fmt.Println("error:", x.Error())
    default:
        fmt.Printf("other: %T %v\n", x, x)
    }
}

describe(42)                    // int: 42
describe("hi")                  // string: hi
describe([]byte{1, 2, 3})       // []byte, len: 3
describe(errors.New("oops"))    // error: oops
```

### JSON and `any`

Unmarshal into `map[string]any` or `any` when structure is dynamic:

```go
var raw map[string]any
err := json.Unmarshal([]byte(`{"a":1,"b":"x","c":[1,2,3]}`), &raw)
if err != nil {
    log.Fatal(err)
}

fmt.Println(raw["a"])   // 1
fmt.Println(raw["b"])   // "x"
fmt.Println(raw["c"])   // []interface {} [1 2 3]
```

```go
var v any
json.Unmarshal([]byte(`[1, "two", {"k": 3}]`), &v)
// v is []interface {} with int, string, map[string]interface {}
```

### “Accept anything” helpers

```go
func logJSON(key string, val any) {
    b, _ := json.Marshal(map[string]any{key: val})
    log.Println(string(b))
}

logJSON("user", user)
logJSON("count", 42)
logJSON("ids", []int{1, 2, 3})
```

### Slice / map of `any`

```go
var rows []any
rows = append(rows, 1, "two", true, []int{1, 2})

m := map[string]any{
    "id":   1,
    "name": "alice",
    "tags": []string{"a", "b"},
}
```

Use when you **truly** have mixed types (e.g. table rows, JSON-like structures). Otherwise prefer typed slices/maps or generics.

### `any` vs generics

- **Generics** — single, known type per use; type-safe; no runtime type checks.
- **`any`** — unknown or mixed types; type info recovered via assertion/switch.

Prefer **generics** when you have a uniform type (e.g. `[]T`, `map[K]V`). Use **`any`** when you need heterogeneous data (JSON, logging, plugins, etc.).

---

## Empty Struct (`struct{}`)

### What it is

A struct type with **no fields**. Its size is **0**. There is effectively a single value of that type; all `struct{}` values are equal.

```go
var x struct{}
fmt.Println(unsafe.Sizeof(x))  // 0
fmt.Println(struct{}{} == struct{}{})  // true
```

### What it isn’t

- **Not** “no value” — it’s a real type with a value. You use it when you need a **value that carries no data**.
- **Not** “nil” — `struct{}` is a value type; you use `struct{}{}` as a literal.
- **Not** interchangeable with `bool` or `int` for “present/absent” when you want **zero allocation** — `struct{}` is the smallest representation.

### Use case 1: Set (map keys only)

You care only about **membership**, not a value. Use `map[T]struct{}`:

```go
set := make(map[string]struct{})
set["a"] = struct{}{}
set["b"] = struct{}{}

if _, ok := set["a"]; ok {
    fmt.Println("a in set")
}

delete(set, "b")
```

`struct{}` uses **no extra memory** per key beyond the map entry itself. `map[T]bool` allocates a bool per key.

### Use case 2: Signal / done channel

Channels often carry “something happened” without payload. `chan struct{}` is the idiom:

```go
done := make(chan struct{})

go func() {
    time.Sleep(time.Second)
    close(done)  // or: done <- struct{}{}
}()

<-done
fmt.Println("done")
```

`close(done)` broadcasts to all receivers; no allocations for events.

### Use case 3: Method receiver (no state)

When a type exists only to attach **behavior** (methods), not state:

```go
type NullHandler struct{}

func (NullHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {}

var _ http.Handler = NullHandler{}
```

All `NullHandler` values are equivalent; no fields needed.

### Use case 4: No-op or sentinel type

```go
type Nop struct{}

func (Nop) Write(p []byte) (n int, err error) { return len(p), nil }

var _ io.Writer = Nop{}
```

### Use case 5: Map that only tracks keys (no value type)

You already use `map[K]struct{}` for sets; the “value” is “key exists.” No other type gives you zero-size values.

---

## Deep Examples

### Example 1: Generic “printer” with `any`

```go
func Print(v any) {
    switch x := v.(type) {
    case string:
        fmt.Println("string:", x)
    case int:
        fmt.Println("int:", x)
    case []int:
        fmt.Println("slice:", x)
    default:
        fmt.Printf("default: %T %v\n", x, x)
    }
}

Print(42)
Print("hello")
Print([]int{1, 2, 3})
```

### Example 2: Table / row as `[]any`

```go
func printRow(row []any) {
    for i, v := range row {
        if i > 0 {
            fmt.Print("\t")
        }
        fmt.Print(v)
    }
    fmt.Println()
}

row1 := []any{1, "alice", true}
row2 := []any{2, "bob", false}
printRow(row1)
printRow(row2)
```

### Example 3: Set operations with `map[T]struct{}`

```go
func toSet(ss []string) map[string]struct{} {
    s := make(map[string]struct{}, len(ss))
    for _, k := range ss {
        s[k] = struct{}{}
    }
    return s
}

func intersect(a, b map[string]struct{}) map[string]struct{} {
    out := make(map[string]struct{})
    for k := range a {
        if _, ok := b[k]; ok {
            out[k] = struct{}{}
        }
    }
    return out
}

func main() {
    a := toSet([]string{"a", "b", "c"})
    b := toSet([]string{"b", "c", "d"})
    fmt.Println(intersect(a, b))  // map[b:{} c:{}]
}
```

### Example 4: Worker pool with `chan struct{}` semaphore

```go
func main() {
    limit := 3
    sem := make(chan struct{}, limit)
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        sem <- struct{}{}
        go func(id int) {
            defer wg.Done()
            defer func() { <-sem }()
            fmt.Println("work", id)
            time.Sleep(100 * time.Millisecond)
        }(i)
    }

    wg.Wait()
}
```

### Example 5: Optional config with `*struct{}`

Sometimes “present” vs “absent” is enough. A pointer can distinguish:

```go
type Config struct {
    Addr  string
    Debug *struct{}  // present = true, nil = false
}

func main() {
    on := &struct{}{}
    c := Config{Addr: ":8080", Debug: on}
    if c.Debug != nil {
        fmt.Println("debug on")
    }
}
```

### Example 6: JSON patch / arbitrary structure

```go
var patch map[string]any
json.Unmarshal([]byte(`{"name":"alice","age":30}`), &patch)

for k, v := range patch {
    fmt.Printf("%s: %v (%T)\n", k, v, v)
}
```

### Example 7: `http.Handler` that does nothing

```go
// struct{}{} alone does NOT implement http.Handler (no ServeHTTP).
// Use a type with an empty struct receiver:

type nopHandler struct{}

func (nopHandler) ServeHTTP(http.ResponseWriter, *http.Request) {}

var _ http.Handler = nopHandler{}
```

The **pattern** is “empty struct + methods” for stateless handlers; the struct carries no data.

---

## When to Use What

| Need | Use |
|------|-----|
| Value of “any” type | `any` / `interface{}` |
| JSON with unknown shape | `map[string]any`, `any` |
| Type switch / assertion on runtime type | `any` |
| Set (membership only) | `map[T]struct{}` |
| Signal / done channel | `chan struct{}` |
| Stateless type (only behavior) | `struct{}` as receiver |
| Semaphore / “token” with zero payload | `chan struct{}` or `map[T]struct{}` |

---

## Best Practices

### Empty interface

- Use **`any`** (clearer than `interface{}`) when you need “any type.”
- Prefer **generics** over `any` when you have a single, known type.
- Use **type switches** or **safe assertions** (`v, ok := x.(T)`) before using values.
- Avoid `any` “everywhere” — use it where heterogeneity or true unknowns are required (JSON, logging, plugins, etc.).

### Empty struct

- Use **`map[K]struct{}`** for sets; avoid `map[K]bool` if you care about allocation.
- Use **`chan struct{}`** for signals / done; `close` for broadcast.
- Use **`struct{}`** as a receiver when a type has only methods and no state.
- Use **`struct{}{}`** as a concrete value; no need for variables if you only need the literal.

---

## Summary

| Concept | Purpose |
|--------|---------|
| **`any` / `interface{}`** | Hold any type; use with type assertion, type switch, JSON, heterogeneous data. |
| **`struct{}`** | Zero-size value; use for sets (`map[K]struct{}`), signals (`chan struct{}`), stateless types. |

**Empty interface** = “any type”; **empty struct** = “no data, smallest value.” Both are simple building blocks used throughout Go code and the standard library.
