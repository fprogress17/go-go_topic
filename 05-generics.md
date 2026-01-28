# Go Generics (Type Parameters) - Complete Guide

## Table of Contents
- [Introduction](#introduction)
- [Basic Syntax](#basic-syntax)
- [Type Constraints](#type-constraints)
- [Generic Data Structures](#generic-data-structures)
- [Multiple Type Parameters](#multiple-type-parameters)
- [Advanced Constraints](#advanced-constraints)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## Introduction

Generics were introduced in **Go 1.18** (March 2022), allowing you to write functions and data structures that work with any type.

### Before Generics (Type-Specific)

```go
func PrintInt(s []int) {
    for _, v := range s {
        fmt.Println(v)
    }
}

func PrintString(s []string) {
    for _, v := range s {
        fmt.Println(v)
    }
}

// Need separate function for each type!
```

### With Generics (Type Parameter)

```go
func Print[T any](s []T) {
    for _, v := range s {
        fmt.Println(v)
    }
}

func main() {
    Print([]int{1, 2, 3})           // T = int
    Print([]string{"a", "b", "c"})  // T = string
    Print([]float64{1.1, 2.2})      // T = float64
}
```

**Syntax breakdown:**
- `[T any]` - Type parameter declaration
- `T` - Type parameter name (can be any name)
- `any` - Constraint (what types T can be)

---

## Basic Syntax

### Function with Type Parameter

```go
func Identity[T any](value T) T {
    return value
}

func main() {
    fmt.Println(Identity(42))        // 42
    fmt.Println(Identity("hello"))   // hello
    fmt.Println(Identity(3.14))      // 3.14
}
```

### Multiple Type Parameters

```go
func Swap[T, U any](a T, b U) (U, T) {
    return b, a
}

func main() {
    x, y := Swap(10, "hello")
    fmt.Println(x, y)  // hello 10
}
```

### Type Inference

```go
// Explicit type
Print[int]([]int{1, 2, 3})

// Type inference (Go can infer)
Print([]int{1, 2, 3})  // Same as above
```

---

## Type Constraints

### 1. `any` - Any type

```go
func First[T any](slice []T) T {
    return slice[0]
}

func main() {
    fmt.Println(First([]int{1, 2, 3}))        // 1
    fmt.Println(First([]string{"a", "b"}))    // "a"
}
```

### 2. `comparable` - Types that support == and !=

```go
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {  // Needs == operator
            return true
        }
    }
    return false
}

func main() {
    fmt.Println(Contains([]int{1, 2, 3}, 2))        // true
    fmt.Println(Contains([]string{"a", "b"}, "c"))  // false
}
```

### 3. Interface constraints

```go
// Define constraint
type Number interface {
    int | int64 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(Sum([]int{1, 2, 3}))           // 6
    fmt.Println(Sum([]float64{1.5, 2.5}))      // 4.0
}
```

### 4. Built-in constraint: constraints package

```go
import "golang.org/x/exp/constraints"

func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Min(10, 20))        // 10
    fmt.Println(Min(3.14, 2.71))     // 2.71
    fmt.Println(Min("apple", "banana")) // "apple"
}
```

`constraints.Ordered` includes: integers, floats, strings (types that support `<`, `>`, etc.)

---

## Generic Data Structures

### 1. Generic Stack

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

func main() {
    // Integer stack
    intStack := Stack[int]{}
    intStack.Push(1)
    intStack.Push(2)
    intStack.Push(3)
    
    val, ok := intStack.Pop()
    fmt.Println(val, ok)  // 3 true
    
    // String stack
    strStack := Stack[string]{}
    strStack.Push("hello")
    strStack.Push("world")
    
    val2, ok := strStack.Pop()
    fmt.Println(val2, ok)  // "world" true
}
```

### 2. Generic Linked List

```go
type Node[T any] struct {
    Value T
    Next  *Node[T]
}

type LinkedList[T any] struct {
    Head *Node[T]
}

func (l *LinkedList[T]) Add(value T) {
    newNode := &Node[T]{Value: value}
    
    if l.Head == nil {
        l.Head = newNode
        return
    }
    
    current := l.Head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
}

func (l *LinkedList[T]) Print() {
    current := l.Head
    for current != nil {
        fmt.Print(current.Value, " -> ")
        current = current.Next
    }
    fmt.Println("nil")
}

func main() {
    list := LinkedList[int]{}
    list.Add(1)
    list.Add(2)
    list.Add(3)
    list.Print()  // 1 -> 2 -> 3 -> nil
}
```

### 3. Generic Map Functions

```go
// Map: transform slice
func Map[T any, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Filter: keep elements matching predicate
func Filter[T any](slice []T, fn func(T) bool) []T {
    result := []T{}
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// Reduce: combine elements
func Reduce[T any, U any](slice []T, initial U, fn func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = fn(result, v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    
    // Map: square each number
    squared := Map(nums, func(n int) int {
        return n * n
    })
    fmt.Println(squared)  // [1 4 9 16 25]
    
    // Filter: only even numbers
    evens := Filter(nums, func(n int) bool {
        return n%2 == 0
    })
    fmt.Println(evens)  // [2 4]
    
    // Reduce: sum all numbers
    sum := Reduce(nums, 0, func(acc, n int) int {
        return acc + n
    })
    fmt.Println(sum)  // 15
}
```

---

## Multiple Type Parameters

### Pair Example

```go
// Pair holds two values of different types
type Pair[T any, U any] struct {
    First  T
    Second U
}

func NewPair[T any, U any](first T, second U) Pair[T, U] {
    return Pair[T, U]{
        First:  first,
        Second: second,
    }
}

func main() {
    p1 := NewPair(1, "one")
    fmt.Printf("%v: %v\n", p1.First, p1.Second)  // 1: one
    
    p2 := NewPair("Alice", 25)
    fmt.Printf("%v: %v\n", p2.First, p2.Second)  // Alice: 25
}
```

### Map Function with Different Types

```go
func MapTo[T any, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3}
    strs := MapTo(nums, func(n int) string {
        return fmt.Sprintf("Number: %d", n)
    })
    fmt.Println(strs)  // [Number: 1 Number: 2 Number: 3]
}
```

---

## Advanced Constraints

### 1. Method Constraints

```go
type Stringer interface {
    String() string
}

func PrintStrings[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

type Person struct {
    Name string
}

func (p Person) String() string {
    return "Person: " + p.Name
}

func main() {
    people := []Person{
        {Name: "Alice"},
        {Name: "Bob"},
    }
    PrintStrings(people)
}
```

### 2. Union Constraints

```go
type Integer interface {
    int | int8 | int16 | int32 | int64
}

type Float interface {
    float32 | float64
}

type Number interface {
    Integer | Float
}

func Add[T Number](a, b T) T {
    return a + b
}

func main() {
    fmt.Println(Add(1, 2))           // int: 3
    fmt.Println(Add(int64(10), 20))  // int64: 30
    fmt.Println(Add(3.14, 2.86))     // float64: 6.0
}
```

### 3. Approximate Constraints (~)

```go
type MyInt int

type SignedInteger interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

func Double[T SignedInteger](n T) T {
    return n * 2
}

func main() {
    var x int = 5
    fmt.Println(Double(x))  // 10
    
    var y MyInt = 7
    fmt.Println(Double(y))  // 14 (works with underlying type)
}
```

`~int` means "int or any type with underlying type int"

### 4. Complex Constraint Example

```go
type Addable interface {
    ~int | ~int64 | ~float64 | ~string
}

type Multipliable interface {
    ~int | ~int64 | ~float64
}

func Process[T Addable](items []T) T {
    var result T
    for _, item := range items {
        result += item  // Only works for Addable types
    }
    return result
}

func Multiply[T Multipliable](a, b T) T {
    return a * b  // Only works for numeric types
}
```

---

## Real-World Examples

### Example 1: Generic Cache

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type CacheItem[T any] struct {
    Value      T
    Expiration time.Time
}

type Cache[K comparable, V any] struct {
    items map[K]CacheItem[V]
    mu    sync.RWMutex
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{
        items: make(map[K]CacheItem[V]),
    }
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.items[key] = CacheItem[V]{
        Value:      value,
        Expiration: time.Now().Add(ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, exists := c.items[key]
    if !exists {
        var zero V
        return zero, false
    }
    
    if time.Now().After(item.Expiration) {
        var zero V
        return zero, false
    }
    
    return item.Value, true
}

func (c *Cache[K, V]) Delete(key K) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func main() {
    // String -> Int cache
    cache1 := NewCache[string, int]()
    cache1.Set("score", 100, 5*time.Second)
    
    if val, ok := cache1.Get("score"); ok {
        fmt.Println("Score:", val)  // Score: 100
    }
    
    // Int -> String cache
    cache2 := NewCache[int, string]()
    cache2.Set(1, "Alice", 5*time.Second)
    cache2.Set(2, "Bob", 5*time.Second)
    
    if val, ok := cache2.Get(1); ok {
        fmt.Println("User 1:", val)  // User 1: Alice
    }
    
    // Test expiration
    time.Sleep(6 * time.Second)
    if _, ok := cache1.Get("score"); !ok {
        fmt.Println("Cache expired!")
    }
}
```

### Example 2: Generic Repository Pattern

```go
type Repository[T any, ID comparable] interface {
    FindByID(id ID) (T, error)
    FindAll() ([]T, error)
    Save(entity T) error
    Delete(id ID) error
}

type UserRepository struct {
    users map[int]User
}

func (r *UserRepository) FindByID(id int) (User, error) {
    user, ok := r.users[id]
    if !ok {
        return User{}, fmt.Errorf("user not found")
    }
    return user, nil
}

func (r *UserRepository) FindAll() ([]User, error) {
    users := make([]User, 0, len(r.users))
    for _, user := range r.users {
        users = append(users, user)
    }
    return users, nil
}

func (r *UserRepository) Save(user User) error {
    r.users[user.ID] = user
    return nil
}

func (r *UserRepository) Delete(id int) error {
    delete(r.users, id)
    return nil
}
```

### Example 3: Generic Option Pattern

```go
type Option[T any] func(*T)

func WithName[T any](name string) Option[T] {
    return func(t *T) {
        // Set name field if T has it
    }
}

type Config struct {
    Name  string
    Port  int
    Debug bool
}

func NewConfig(opts ...Option[Config]) *Config {
    c := &Config{
        Port:  8080,
        Debug: false,
    }
    for _, opt := range opts {
        opt(c)
    }
    return c
}
```

---

## Generic Constraints Best Practices

### ✅ DO:

```go
// Use meaningful constraint names
type Numeric interface {
    int | float64
}

// Keep constraints focused
type Addable interface {
    int | int64 | float64 | string
}

// Use built-in constraints when available
import "golang.org/x/exp/constraints"

func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

### ❌ DON'T:

```go
// Don't over-constrain
type TooSpecific interface {
    int | int8  // Too narrow, use constraints.Integer
}

// Don't use generics when interface{} is clearer
func Print(v any) {  // Better than func Print[T any](v T)
    fmt.Println(v)
}

// Don't use generics for simple cases
func Add(a, b int) int {  // Simple, no need for generics
    return a + b
}
```

---

## Limitations

### 1. No generic methods (only functions and types)

```go
type Container[T any] struct {
    value T
}

// ✅ OK: method on generic type
func (c Container[T]) Get() T {
    return c.value
}

// ❌ NOT ALLOWED: generic method
// func (c Container) GenericMethod[U any](u U) { }
```

### 2. Cannot use type parameters in const

```go
// ❌ ERROR
func Example[T any]() {
    const x T = 10  // Cannot use type parameter in const
}
```

### 3. Type parameters must be used

```go
// ❌ ERROR: T is not used
func Bad[T any]() {
    fmt.Println("hello")
}

// ✅ OK: T is used
func Good[T any](value T) {
    fmt.Println(value)
}
```

---

## Summary

| Feature | Example | Use Case |
|---------|---------|----------|
| `any` | `func F[T any](v T)` | Accept any type |
| `comparable` | `func F[T comparable](v T)` | Need `==`, `!=` |
| Interface constraint | `func F[T Stringer](v T)` | Need specific methods |
| Union constraint | `int \| float64` | Limit to specific types |
| Multiple parameters | `[T any, U any]` | Different types in same function |
| Generic struct | `type Stack[T any]` | Reusable data structures |

### When to use generics:

- ✅ Type-safe collections (stack, queue, tree)
- ✅ Utility functions (map, filter, reduce)
- ✅ Algorithm implementations (sort, search)
- ✅ Avoiding code duplication

### When NOT to use:

- ❌ Simple interfaces work fine
- ❌ Type assertions are acceptable
- ❌ Over-engineering simple code
- ❌ When `any`/`interface{}` is clearer

---

Generics make Go more expressive while maintaining type safety! 🚀
