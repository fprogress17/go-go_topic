# Go nil - Complete Guide

## Table of Contents
- [What Can Be nil?](#what-can-be-nil)
- [Pointer nil](#pointer-nil)
- [Slice nil](#slice-nil)
- [Map nil](#map-nil)
- [Channel nil](#channel-nil)
- [Interface nil](#interface-nil)
- [Function nil](#function-nil)
- [Error nil](#error-nil)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## What Can Be nil?

`nil` is Go's zero value for pointers, interfaces, slices, maps, channels, and function types.

```go
var p *int           // nil pointer
var s []int          // nil slice
var m map[string]int // nil map
var ch chan int      // nil channel
var f func()         // nil function
var i interface{}    // nil interface
var err error        // nil interface (error is interface)
```

### What CANNOT be nil:

```go
var x int      // 0 (not nil)
var b bool     // false (not nil)
var str string // "" (not nil)
var arr [3]int // [0 0 0] (not nil, arrays have fixed size)
```

---

## Pointer nil

### Basic Pointer

```go
func main() {
    var p *int
    fmt.Println(p == nil)  // true
    
    // Dereferencing nil pointer = PANIC!
    // fmt.Println(*p)  // panic: runtime error: invalid memory address
    
    // Safe check
    if p != nil {
        fmt.Println(*p)
    } else {
        fmt.Println("Pointer is nil")
    }
}
```

### Creating Non-nil Pointer

```go
func main() {
    x := 42
    p := &x
    fmt.Println(p == nil)   // false
    fmt.Println(*p)         // 42
    
    // Using new()
    p2 := new(int)
    fmt.Println(p2 == nil)  // false
    fmt.Println(*p2)        // 0 (zero value)
}
```

---

## Slice nil

### Nil Slice vs Empty Slice

```go
func main() {
    var s1 []int           // nil slice
    s2 := []int{}          // empty slice (not nil!)
    s3 := make([]int, 0)   // empty slice (not nil!)
    
    fmt.Println(s1 == nil) // true
    fmt.Println(s2 == nil) // false
    fmt.Println(s3 == nil) // false
    
    fmt.Println(len(s1))   // 0
    fmt.Println(len(s2))   // 0
    fmt.Println(len(s3))   // 0
}
```

### Working with Nil Slices

```go
func main() {
    var nums []int  // nil
    
    // Safe operations on nil slice
    fmt.Println(len(nums))     // 0 (no panic!)
    fmt.Println(cap(nums))     // 0
    
    // Can append to nil slice
    nums = append(nums, 1, 2, 3)
    fmt.Println(nums)          // [1 2 3]
    
    // Can range over nil slice
    for _, n := range nums {
        fmt.Println(n)
    }
}
```

### Nil Slice in JSON

```go
import (
    "encoding/json"
    "fmt"
)

type Response struct {
    Data []string `json:"data"`
}

func main() {
    // nil slice
    r1 := Response{Data: nil}
    json1, _ := json.Marshal(r1)
    fmt.Println(string(json1))  // {"data":null}
    
    // empty slice
    r2 := Response{Data: []string{}}
    json2, _ := json.Marshal(r2)
    fmt.Println(string(json2))  // {"data":[]}
}
```

---

## Map nil

### Nil Map Behavior

```go
func main() {
    var m map[string]int  // nil map
    
    fmt.Println(m == nil)     // true
    fmt.Println(len(m))       // 0
    
    // Reading from nil map is OK
    val := m["key"]
    fmt.Println(val)          // 0 (zero value)
    
    val, ok := m["key"]
    fmt.Println(val, ok)      // 0 false
    
    // Writing to nil map = PANIC!
    // m["key"] = 42  // panic: assignment to entry in nil map
}
```

### Creating Non-nil Map

```go
func main() {
    // Method 1: make()
    m1 := make(map[string]int)
    m1["key"] = 42
    fmt.Println(m1)  // map[key:42]
    
    // Method 2: literal
    m2 := map[string]int{}
    m2["key"] = 42
    fmt.Println(m2)  // map[key:42]
    
    // Method 3: initialized
    m3 := map[string]int{
        "a": 1,
        "b": 2,
    }
    fmt.Println(m3)  // map[a:1 b:2]
}
```

---

## Channel nil

### Nil Channel Behavior

```go
func main() {
    var ch chan int  // nil channel
    
    fmt.Println(ch == nil)  // true
    
    // Operations on nil channel BLOCK FOREVER!
    
    // This blocks forever:
    // ch <- 42      // send blocks
    // <-ch          // receive blocks
    // close(ch)     // panic: close of nil channel
}
```

### Useful Pattern: Disabling Channel in Select

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    go func() {
        ch1 <- 1
        close(ch1)
    }()
    
    go func() {
        ch2 <- 2
        close(ch2)
    }()
    
    for i := 0; i < 2; i++ {
        select {
        case val, ok := <-ch1:
            if ok {
                fmt.Println("ch1:", val)
            }
            ch1 = nil  // Disable this case
        case val, ok := <-ch2:
            if ok {
                fmt.Println("ch2:", val)
            }
            ch2 = nil  // Disable this case
        }
    }
}
```

---

## Interface nil

### Interface Nil Complexity

```go
func main() {
    var i interface{}
    fmt.Println(i == nil)  // true
    
    var p *int
    i = p
    fmt.Println(i == nil)  // false! (has type info)
    
    fmt.Printf("%T, %v\n", i, i)  // *int, <nil>
}
```

**Why?** Interface holds two things:
- Type information
- Value

Interface is nil only if **both** are nil.

### The Nil Interface Trap

```go
type MyError struct {
    Msg string
}

func (e *MyError) Error() string {
    return e.Msg
}

func doSomething() error {
    var err *MyError  // nil pointer
    // ... some logic
    return err        // Returns interface with nil value but non-nil type!
}

func main() {
    err := doSomething()
    
    // This looks wrong!
    if err != nil {
        fmt.Println("Error occurred:", err)  // Prints!
    }
    
    // Correct check
    if err != nil && err.Error() != "" {
        fmt.Println("Real error:", err)
    }
}
```

**Fix:**

```go
func doSomething() error {
    var err *MyError
    if someCondition {
        err = &MyError{Msg: "something went wrong"}
    }
    
    if err != nil {
        return err
    }
    return nil  // Return typed nil (interface nil)
}
```

---

## Function nil

```go
func main() {
    var f func()
    
    fmt.Println(f == nil)  // true
    
    // Calling nil function = PANIC!
    // f()  // panic: runtime error: invalid memory address
    
    // Safe check
    if f != nil {
        f()
    }
}
```

---

## Error nil (Special Case)

### Error Interface

```go
type error interface {
    Error() string
}
```

### Common Patterns

```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil  // nil = success
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)  // 5
}
```

---

## Nil Comparison Table

| Type | Can be nil? | Zero value | Safe operations |
|------|-------------|------------|-----------------|
| `*T` (pointer) | ✅ Yes | `nil` | Compare, check `!= nil` |
| `[]T` (slice) | ✅ Yes | `nil` | `len()`, `cap()`, `range`, `append()` |
| `map[K]V` | ✅ Yes | `nil` | Read (returns zero), `len()` |
| `chan T` | ✅ Yes | `nil` | Compare (blocks on send/receive) |
| `func()` | ✅ Yes | `nil` | Compare only |
| `interface{}` | ✅ Yes | `nil` | Compare, type assertion |
| `error` | ✅ Yes | `nil` | Compare |
| `int`, `bool`, `string` | ❌ No | `0`, `false`, `""` | N/A |
| `[N]T` (array) | ❌ No | Zero array | N/A |
| `struct{}` | ❌ No | Zero struct | N/A |

---

## Real-World Examples

### Example 1: Optional Fields with Pointers

```go
type User struct {
    Name  string
    Email *string  // Optional
    Age   *int     // Optional
}

func main() {
    email := "user@example.com"
    age := 30
    
    u1 := User{
        Name:  "Alice",
        Email: &email,
        Age:   &age,
    }
    
    u2 := User{
        Name: "Bob",
        // Email and Age are nil
    }
    
    if u1.Email != nil {
        fmt.Println("Email:", *u1.Email)
    }
    
    if u2.Email != nil {
        fmt.Println("Email:", *u2.Email)
    } else {
        fmt.Println("No email provided")
    }
}
```

### Example 2: Lazy Initialization

```go
type Cache struct {
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    if c.data == nil {
        c.data = make(map[string]string)
    }
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key, value string) {
    if c.data == nil {
        c.data = make(map[string]string)
    }
    c.data[key] = value
}

func main() {
    cache := &Cache{}  // data is nil initially
    
    cache.Set("name", "Alice")
    val, ok := cache.Get("name")
    fmt.Println(val, ok)  // Alice true
}
```

### Example 3: Nil Receiver Pattern

```go
type Tree struct {
    Value int
    Left  *Tree
    Right *Tree
}

// Method works even if receiver is nil!
func (t *Tree) Sum() int {
    if t == nil {
        return 0
    }
    return t.Value + t.Left.Sum() + t.Right.Sum()
}

func main() {
    tree := &Tree{
        Value: 1,
        Left: &Tree{
            Value: 2,
            Left:  nil,
            Right: nil,
        },
        Right: &Tree{
            Value: 3,
        },
    }
    
    fmt.Println(tree.Sum())       // 6
    fmt.Println(tree.Left.Sum())  // 2
    
    var empty *Tree
    fmt.Println(empty.Sum())      // 0 (no panic!)
}
```

---

## Best Practices

### ✅ DO:

```go
// Check nil before dereferencing
if ptr != nil {
    value := *ptr
}

// Return typed nil for interfaces
func DoWork() error {
    // ...
    return nil  // Not: return (*MyError)(nil)
}

// Use nil slices for "no data"
var results []string  // nil = no results

// Initialize maps before use
m := make(map[string]int)
```

### ❌ DON'T:

```go
// Don't compare slice to nil for emptiness
if len(slice) == 0 {  // Better than: if slice == nil
    // ...
}

// Don't return concrete nil as interface
func Bad() error {
    var err *MyError  // nil
    return err        // BAD: interface with type!
}

// Don't forget to check nil
*ptr = 42  // DANGER: might panic
```

---

## Summary

- `nil` is the zero value for reference types
- **Pointers:** Must check before dereferencing
- **Slices:** Can safely use `len()`, `append()`, `range`
- **Maps:** Can read but NOT write
- **Channels:** Operations block forever
- **Interfaces:** `nil` only if both type and value are `nil`
- **Functions:** Must check before calling

**Golden Rule:** Always check `!= nil` before using pointers, maps, and functions!
