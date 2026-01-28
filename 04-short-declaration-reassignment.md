# Go Short Declaration (:=) Re-assignment Guide

## Table of Contents
- [Basic Re-assignment](#basic-re-assignment)
- [The Rule](#the-rule)
- [Multiple Variables](#multiple-variables)
- [Deep Examples](#deep-examples)
- [Scope and Shadowing](#scope-and-shadowing)
- [Best Practices](#best-practices)

---

## Basic Re-assignment

### Short Answer

**YES**, variables declared with `:=` are fully re-assignable (mutable).

### Basic Re-assignment

```go
func main() {
    x := 10        // Declare and initialize
    fmt.Println(x) // 10
    
    x = 20         // Re-assign (use = not :=)
    fmt.Println(x) // 20
    
    x = 30         // Re-assign again
    fmt.Println(x) // 30
}
```

✅ **Works perfectly!**

---

## Common Confusion

### ❌ WRONG: Using := for re-assignment

```go
func main() {
    x := 10
    x := 20  // ❌ ERROR: no new variables on left side of :=
}
```

**Error:** `no new variables on left side of :=`

### ✅ CORRECT: Use = for re-assignment

```go
func main() {
    x := 10  // := for declaration
    x = 20   // = for re-assignment
}
```

---

## The Rule

| Operation | Syntax | When to use |
|-----------|--------|-------------|
| Declaration + Initialization | `x := 10` | First time creating variable |
| Re-assignment | `x = 20` | Changing existing variable |

---

## Multiple Variables

### Case 1: All new variables

```go
func main() {
    x, y := 10, 20  // Both new - OK with :=
    fmt.Println(x, y)
}
```

### Case 2: All existing variables

```go
func main() {
    x, y := 10, 20
    x, y = 30, 40   // Both exist - use =
    fmt.Println(x, y)  // 30 40
}
```

### Case 3: Mixed (at least one new)

```go
func main() {
    x := 10
    
    // x exists, but y is new - := is OK!
    x, y := 20, 30  // x re-assigned, y declared
    
    fmt.Println(x, y)  // 20 30
}
```

**Rule:** `:=` is allowed if at least one variable on the left is new.

---

## Deep Examples

### Example 1: Loop re-assignment

```go
func main() {
    sum := 0  // Declare with :=
    
    for i := 0; i < 5; i++ {
        sum = sum + i  // Re-assign with =
    }
    
    fmt.Println(sum)  // 10
}
```

### Example 2: Conditional re-assignment

```go
func main() {
    status := "pending"  // Declare
    
    if someCondition {
        status = "approved"  // Re-assign
    } else {
        status = "rejected"  // Re-assign
    }
    
    fmt.Println(status)
}
```

### Example 3: Error handling pattern

```go
func main() {
    data, err := readFile("a.txt")  // Declare both
    if err != nil {
        return
    }
    
    // Re-assign err, declare moreData
    moreData, err := readFile("b.txt")  // Mixed: OK!
    if err != nil {
        return
    }
    
    fmt.Println(data, moreData)
}
```

### Example 4: Multiple re-assignments

```go
func main() {
    x := 1
    fmt.Println(x)  // 1
    
    x = 2
    fmt.Println(x)  // 2
    
    x = 3
    fmt.Println(x)  // 3
    
    x = 4
    fmt.Println(x)  // 4
    
    // Can re-assign as many times as needed
}
```

### Example 5: Re-assignment in different scopes

```go
func main() {
    x := 10
    
    {
        x = 20  // Re-assigns outer x
        fmt.Println(x)  // 20
    }
    
    fmt.Println(x)  // 20 (changed!)
}
```

---

## Scope and Shadowing

### Be careful with scope!

```go
func main() {
    x := 10  // Outer x
    
    if true {
        x := 20  // NEW variable (shadows outer x)
        fmt.Println(x)  // 20
    }
    
    fmt.Println(x)  // 10 (outer x unchanged!)
}
```

This is **NOT** re-assignment, it's **shadowing** (new variable with same name in inner scope).

### Actual re-assignment in scope

```go
func main() {
    x := 10  // Outer x
    
    if true {
        x = 20   // Re-assign outer x (use =)
        fmt.Println(x)  // 20
    }
    
    fmt.Println(x)  // 20 (outer x changed!)
}
```

### Function parameters

```go
func process(x int) {
    x = 100  // Re-assigns parameter (local copy)
    fmt.Println(x)  // 100
}

func main() {
    x := 10
    process(x)
    fmt.Println(x)  // 10 (unchanged - passed by value)
}
```

---

## Pointer Re-assignment

### Re-assigning what pointer points to

```go
func main() {
    x := 10
    p := &x    // p points to x
    
    fmt.Println(*p)  // 10
    
    // Re-assign what p points to
    *p = 20
    fmt.Println(x)   // 20 (x changed!)
    
    y := 30
    p = &y     // Re-assign p itself to point to y
    fmt.Println(*p)  // 30
    fmt.Println(x)   // 20 (x unchanged)
}
```

### Re-assigning pointer value

```go
func main() {
    x := 10
    y := 20
    
    p := &x
    fmt.Println(*p)  // 10
    
    p = &y  // Re-assign pointer
    fmt.Println(*p)  // 20
}
```

---

## Struct Field Re-assignment

```go
type Person struct {
    Name string
    Age  int
}

func main() {
    p := Person{Name: "Alice", Age: 25}
    
    // Re-assign fields
    p.Name = "Bob"
    p.Age = 30
    
    fmt.Println(p)  // {Bob 30}
}
```

### Pointer to struct

```go
func main() {
    p := &Person{Name: "Alice", Age: 25}
    
    // Re-assign through pointer
    p.Name = "Bob"
    (*p).Age = 30  // Same as p.Age = 30
    
    fmt.Println(p)  // &{Bob 30}
}
```

---

## Slice/Map Re-assignment

### Slice

```go
func main() {
    // Slice
    nums := []int{1, 2, 3}
    nums[0] = 10        // Modify element
    nums = []int{4, 5}  // Re-assign entire slice
    
    fmt.Println(nums)  // [4 5]
}
```

### Map

```go
func main() {
    // Map
    m := map[string]int{"a": 1}
    m["a"] = 10         // Modify value
    m = map[string]int{"b": 2}  // Re-assign entire map
    
    fmt.Println(m)  // map[b:2]
}
```

### Array (not slice)

```go
func main() {
    arr := [3]int{1, 2, 3}
    arr[0] = 10  // Modify element
    arr = [3]int{4, 5, 6}  // Re-assign entire array
    
    fmt.Println(arr)  // [4 5 6]
}
```

---

## Comparison: Go vs Other Languages

### JavaScript (let/const)

```javascript
let x = 10;    // Mutable
x = 20;        // OK

const y = 10;  // Immutable
y = 20;        // ERROR
```

### Go (no const for variables)

```go
x := 10   // Always mutable
x = 20    // Always OK

// No "const" for variables (only for compile-time constants)
const PI = 3.14  // Compile-time constant
```

---

## Constants vs Variables

```go
func main() {
    // Variable (mutable)
    x := 10
    x = 20  // OK
    
    // Constant (immutable)
    const y = 10
    y = 20  // ❌ ERROR: cannot assign to y
}
```

### Iota for constants

```go
const (
    Sunday = iota  // 0
    Monday         // 1
    Tuesday        // 2
)

// Cannot re-assign
Sunday = 10  // ❌ ERROR
```

---

## Advanced Patterns

### Pattern 1: Re-assignment in switch

```go
func main() {
    result := "unknown"
    
    switch value := 42; {
    case value < 10:
        result = "small"
    case value < 100:
        result = "medium"
    default:
        result = "large"
    }
    
    fmt.Println(result)  // medium
}
```

### Pattern 2: Re-assignment with type assertion

```go
func main() {
    var i interface{} = "hello"
    
    // Re-assign with type assertion
    s := i.(string)
    s = "world"  // Re-assign
    
    fmt.Println(s)  // world
}
```

### Pattern 3: Re-assignment in defer

```go
func main() {
    x := 10
    
    defer func() {
        fmt.Println("Defer:", x)  // 20 (captures final value)
    }()
    
    x = 20  // Re-assign before defer executes
}
```

---

## Summary Table

| Syntax | Meaning | Re-assignable? |
|--------|---------|----------------|
| `x := 10` | Declare + initialize | ✅ YES (use `x = 20`) |
| `var x = 10` | Declare + initialize | ✅ YES (use `x = 20`) |
| `var x int` | Declare (zero value) | ✅ YES (use `x = 20`) |
| `const x = 10` | Constant | ❌ NO (immutable) |

---

## Key Takeaway

```go
// Declaration (first time only)
x := 10

// Re-assignment (as many times as you want)
x = 20
x = 30
x = 40
// ... forever
```

**`:=` declares, `=` re-assigns. The variable itself is always mutable unless it's a `const`!**

---

## Common Mistakes

### Mistake 1: Using := for re-assignment

```go
x := 10
x := 20  // ❌ ERROR
```

### Mistake 2: Forgetting = in re-assignment

```go
x := 10
x := 20  // Wrong - should be x = 20
```

### Mistake 3: Confusing shadowing with re-assignment

```go
x := 10
{
    x := 20  // Shadowing, not re-assignment
}
// x is still 10
```

---

## Best Practices

### ✅ DO:

```go
// Use := for declaration
x := 10

// Use = for re-assignment
x = 20

// Re-assign as needed
x = 30
```

### ❌ DON'T:

```go
// Don't use := for re-assignment
x := 10
x := 20  // ERROR

// Don't confuse shadowing
x := 10
{
    x := 20  // Creates new variable
}
```

Remember: **`:=` declares, `=` re-assigns!** 🚀
