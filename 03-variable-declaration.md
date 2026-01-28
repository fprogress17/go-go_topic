# Go Variable Declaration - Complete Guide

## Table of Contents
- [Syntax Options](#syntax-options)
- [var vs := Comparison](#var-vs--comparison)
- [Examples in Context](#examples-in-context)
- [Common Patterns](#common-patterns)
- [Best Practices](#best-practices)

---

## Syntax Options

### ❌ WRONG Syntax

```go
var a :=  // SYNTAX ERROR!
```

**Error:** `unexpected :=, expecting type`

You **cannot** use `var` and `:=` together.

### ✅ CORRECT Syntax Options

#### Option 1: var with type

```go
var a int
var b string
var c bool

// Later assign
a = 42
b = "hello"
c = true
```

#### Option 2: var with initialization

```go
var a = 42        // Type inferred as int
var b = "hello"   // Type inferred as string
var c = true      // Type inferred as bool
```

#### Option 3: var with explicit type + value

```go
var a int = 42
var b string = "hello"
var c bool = true
```

#### Option 4: Short declaration := (most common)

```go
a := 42        // Type inferred
b := "hello"
c := true
```

⚠️ **Important:** `:=` can only be used inside functions, not at package level.

---

## var vs := Comparison

| Feature | `var a int` | `var a = 42` | `a := 42` |
|---------|-------------|--------------|-----------|
| Type specified? | Yes | No (inferred) | No (inferred) |
| Must initialize? | No (zero value) | Yes | Yes |
| Where usable? | Anywhere | Anywhere | Functions only |
| Re-declaration? | No | No | Yes (with := multi-assign) |

---

## Examples in Context

### 1. Package-level (must use var)

```go
package main

import "fmt"

// OK
var globalA int
var globalB = "hello"
var globalC string = "world"

// ERROR: syntax error
// globalD := 42  // ❌ Cannot use := at package level

func main() {
    fmt.Println(globalA, globalB, globalC)
}
```

### 2. Function-level (both work)

```go
func main() {
    // Using var
    var x int = 10
    var y = 20        // Type inferred
    var z int         // Zero value (0)
    
    // Using :=
    a := 10
    b := "test"
    c := true
    
    fmt.Println(x, y, z, a, b, c)
}
```

### 3. Multiple declarations

```go
func main() {
    // var block
    var (
        name string = "Alice"
        age  int    = 30
        active bool = true
    )
    
    // Multiple :=
    x, y := 10, 20
    a, b, c := 1, "hello", true
    
    fmt.Println(name, age, active, x, y, a, b, c)
}
```

### 4. Zero values (var only)

```go
func main() {
    var i int       // 0
    var f float64   // 0.0
    var b bool      // false
    var s string    // "" (empty string)
    var p *int      // nil
    
    // := requires initialization
    // x :=  // ❌ ERROR: cannot use := without value
    
    fmt.Println(i, f, b, s, p)
}
```

**Output:** `0 0 false  <nil>`

### 5. Re-declaration trick with :=

```go
func main() {
    x := 10
    
    // This is OK! (at least one new variable)
    x, y := 20, 30  // x re-assigned, y newly declared
    
    fmt.Println(x, y)  // 20 30
}
```

But this fails:

```go
func main() {
    x := 10
    x := 20  // ❌ ERROR: no new variables on left side of :=
}
```

### 6. Type conversion requirement

```go
func main() {
    // Explicit type needed
    var x int64 = 42
    
    // Short form infers int (not int64)
    y := 42  // Type is int
    
    // Type mismatch
    // var z int64 = y  // ❌ ERROR: cannot use y (type int) as type int64
    
    // Must convert
    var z int64 = int64(y)  // ✅ OK
    
    fmt.Println(x, y, z)
}
```

---

## Common Patterns

### Pattern 1: Declare now, assign later

```go
var result string

if condition {
    result = "yes"
} else {
    result = "no"
}

fmt.Println(result)
```

### Pattern 2: Short and clean (idiomatic Go)

```go
// Instead of
var err error
err = doSomething()

// Prefer
err := doSomething()
```

### Pattern 3: Error handling

```go
// Common idiom
if err := doSomething(); err != nil {
    return err
}

// err is scoped to if block only
```

### Pattern 4: Type assertion

```go
// With var
var val interface{} = "hello"
var str string
str = val.(string)

// With :=
val := interface{}("hello")
str := val.(string)
```

### Pattern 5: Function return values

```go
// Multiple return values
func getValues() (int, string) {
    return 42, "hello"
}

// Using var
var num int
var str string
num, str = getValues()

// Using :=
num, str := getValues()
```

---

## Advanced Examples

### Example 1: Pointer variables

```go
func main() {
    // var with pointer
    var p *int
    x := 42
    p = &x
    
    // := with pointer
    y := 42
    q := &y
    
    fmt.Println(*p, *q)  // 42 42
}
```

### Example 2: Slice and map initialization

```go
func main() {
    // var with zero value
    var s []int        // nil slice
    var m map[string]int  // nil map
    
    // var with initialization
    var s2 = []int{1, 2, 3}
    var m2 = map[string]int{"a": 1}
    
    // := with initialization
    s3 := []int{1, 2, 3}
    m3 := map[string]int{"a": 1}
    
    // := with make
    s4 := make([]int, 0, 10)
    m4 := make(map[string]int)
}
```

### Example 3: Struct initialization

```go
type Person struct {
    Name string
    Age  int
}

func main() {
    // var with zero value
    var p Person  // {"" 0}
    
    // var with initialization
    var p2 = Person{Name: "Alice", Age: 30}
    
    // := with initialization
    p3 := Person{Name: "Bob", Age: 25}
    
    // := with pointer
    p4 := &Person{Name: "Charlie", Age: 35}
}
```

### Example 4: Interface variables

```go
type Writer interface {
    Write([]byte) (int, error)
}

func main() {
    // var with nil interface
    var w Writer  // nil
    
    // := with concrete type
    w2 := &bytes.Buffer{}
    
    // Type assertion
    var w3 interface{} = w2
    w4 := w3.(Writer)
}
```

---

## When to Use What?

### Use `var` when:

1. **Package-level variables**
```go
package main

var (
    ConfigFile = "config.json"
    DebugMode  = false
)
```

2. **Need zero value initialization**
```go
var count int  // Starts at 0
var buffer bytes.Buffer  // Empty buffer
```

3. **Want explicit type declaration**
```go
var port int = 8080  // Clear that port is int
```

4. **Declaring without immediate value**
```go
var result string
// ... later
result = calculate()
```

### Use `:=` when:

1. **Inside functions (most common)**
```go
func process() {
    data := readFile()  // Most idiomatic
    err := validate(data)
}
```

2. **Type inference is clear**
```go
name := "Alice"  // Obviously string
count := 42      // Obviously int
```

3. **Quick, local variables**
```go
if err := doSomething(); err != nil {
    return err
}
```

4. **Idiomatic Go style**
```go
// Preferred in Go
result := calculate()
```

---

## Common Mistakes

### Mistake 1: Using := at package level

```go
package main

// ❌ ERROR
count := 0

// ✅ CORRECT
var count = 0
// or
var count int
```

### Mistake 2: Re-declaring without new variable

```go
func main() {
    x := 10
    x := 20  // ❌ ERROR
    
    // ✅ CORRECT
    x = 20
    // or
    x, y := 20, 30  // At least one new variable
}
```

### Mistake 3: Mixing var and :=

```go
// ❌ ERROR
var x := 10

// ✅ CORRECT
var x = 10
// or
x := 10
```

### Mistake 4: Forgetting := in if statements

```go
// ❌ ERROR
if err = doSomething(); err != nil {
    // err might not be declared
}

// ✅ CORRECT
if err := doSomething(); err != nil {
    // err is properly scoped
}
```

---

## Best Practices

### ✅ DO:

```go
// Use := for local variables
func process() {
    data := readData()
    result := calculate(data)
}

// Use var for package-level
var (
    Config = loadConfig()
    Logger = setupLogger()
)

// Use var when you need zero value
var buffer bytes.Buffer
```

### ❌ DON'T:

```go
// Don't use := at package level
count := 0  // ERROR

// Don't mix var and :=
var x := 10  // ERROR

// Don't forget := in if statements
if err = doSomething() {  // Might be wrong
```

---

## Summary

| Syntax | Use Case | Example |
|--------|----------|---------|
| `var a int` | Package-level, zero value | `var count int` |
| `var a = 42` | Package-level, inferred | `var port = 8080` |
| `var a int = 42` | Explicit type + value | `var port int = 8080` |
| `a := 42` | Function-level (idiomatic) | `count := 0` |

**Golden rule:** Choose based on clarity and idiom, not just syntax!

---

## Quick Reference

```go
// Package level
var GlobalVar = "value"
var globalVar = "value"

// Function level
func example() {
    // All valid:
    var x int
    var y = 10
    var z int = 20
    a := 30
    
    // Multiple
    var (
        name string = "Alice"
        age  int    = 30
    )
    x, y := 1, 2
}
```

Remember: `:=` declares, `=` re-assigns! 🚀
