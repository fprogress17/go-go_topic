# Go Embedding & Promotion — Complete Guide

## Table of Contents
- [Overview](#overview)
- [Struct Embedding](#struct-embedding)
- [Interface Embedding](#interface-embedding)
- [Field & Method Promotion](#field--method-promotion)
- [Shadowing & Conflicts](#shadowing--conflicts)
- [Deep Examples](#deep-examples)
- [Embedding vs Composition](#embedding-vs-composition)
- [Best Practices](#best-practices)

---

## Overview

**Embedding** is Go’s way of **composition** — including one type inside another as an **anonymous field**. The embedded type’s fields and methods are **promoted** to the outer type, so you can access them directly.

| Concept | What it is |
|---------|------------|
| **Embedding** | Anonymous field in struct/interface: `type A struct { B }` |
| **Promotion** | Fields/methods of embedded type become accessible on outer type |
| **Struct embedding** | Embed a struct to inherit its fields and methods |
| **Interface embedding** | Embed interfaces to combine method sets |

---

## Struct Embedding

### Basic Syntax

```go
type Address struct {
    Street string
    City   string
    Zip    string
}

// Nested (not embedded)
type Person struct {
    Name    string
    Address Address  // named field
}

// Embedded (anonymous field)
type Employee struct {
    Name    string
    Address  // embedded - no field name, just type
}
```

### Accessing Embedded Fields

```go
func main() {
    // Nested: must use field name
    p := Person{
        Name: "Alice",
        Address: Address{
            Street: "123 Main St",
            City:   "Boston",
        },
    }
    fmt.Println(p.Address.City)  // Boston

    // Embedded: direct access (promoted)
    e := Employee{
        Name: "Bob",
        Address: Address{
            Street: "456 Oak Ave",
            City:   "NYC",
        },
    }
    fmt.Println(e.City)          // NYC (promoted!)
    fmt.Println(e.Address.City)  // Also works
}
```

### Method Promotion

Methods on the embedded type are **promoted** to the outer type:

```go
type Point struct {
    X, Y float64
}

func (p Point) Distance() float64 {
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

type Circle struct {
    Center Point  // embedded
    Radius float64
}

func main() {
    c := Circle{
        Center: Point{X: 0, Y: 0},
        Radius: 5,
    }
    
    // Method promoted!
    fmt.Println(c.Distance())      // 0 (distance from origin)
    fmt.Println(c.Center.Distance())  // Also works
}
```

### Pointer Embedding

You can embed a **pointer** to a struct:

```go
type Config struct {
    Host string
    Port int
}

type Server struct {
    *Config  // embedded pointer
    Name string
}

func main() {
    s := Server{
        Config: &Config{Host: "localhost", Port: 8080},
        Name:   "API",
    }
    
    fmt.Println(s.Host)  // localhost (promoted)
    fmt.Println(s.Port)  // 8080
}
```

**Warning:** If `Config` is `nil`, accessing promoted fields will panic.

---

## Interface Embedding

### Basic Syntax

Embed interfaces to **combine method sets**:

```go
type Reader interface {
    Read() string
}

type Writer interface {
    Write(string)
}

// Embed interfaces
type ReadWriter interface {
    Reader  // embedded
    Writer  // embedded
}
```

A type implements `ReadWriter` if it implements **both** `Reader` and `Writer`.

### Example

```go
type File struct {
    content string
}

func (f *File) Read() string {
    return f.content
}

func (f *File) Write(s string) {
    f.content = s
}

func main() {
    var rw ReadWriter = &File{}
    rw.Write("hello")
    fmt.Println(rw.Read())  // hello
}
```

### Multiple Interface Embedding

```go
type Closer interface {
    Close() error
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

### Interface Embedding in Structs

You can embed an **interface** in a struct:

```go
type Logger interface {
    Log(message string)
}

type ConsoleLogger struct{}

func (ConsoleLogger) Log(message string) {
    fmt.Println("[LOG]", message)
}

type Service struct {
    Logger  // embedded interface
    Name string
}

func (s *Service) DoWork() {
    s.Log(fmt.Sprintf("%s is working", s.Name))
}

func main() {
    svc := &Service{
        Logger: ConsoleLogger{},
        Name:   "UserService",
    }
    svc.DoWork()  // [LOG] UserService is working
}
```

This is useful for **dependency injection** — the struct holds an interface, and you can swap implementations.

---

## Field & Method Promotion

### When Promotion Happens

**Fields** and **methods** of embedded types are **promoted** (accessible directly on the outer type) if:

1. The field/method name is **not shadowed** by the outer type.
2. The field/method is **exported** (capitalized) — unexported fields/methods are not promoted.

### Field Promotion

```go
type Base struct {
    ID   int
    name string  // unexported
}

type Derived struct {
    Base
    Value string
}

func main() {
    d := Derived{
        Base:  Base{ID: 1, name: "test"},
        Value: "hello",
    }
    
    fmt.Println(d.ID)    // 1 (promoted)
    fmt.Println(d.Value) // hello
    // fmt.Println(d.name)  // ERROR: name not promoted (unexported)
    fmt.Println(d.Base.name)  // OK: access via embedded type
}
```

### Method Promotion

```go
type Base struct {
    Value int
}

func (b Base) GetValue() int {
    return b.Value
}

func (b *Base) SetValue(v int) {
    b.Value = v
}

type Derived struct {
    Base
}

func main() {
    d := Derived{Base: Base{Value: 10}}
    
    // Methods promoted!
    fmt.Println(d.GetValue())  // 10
    d.SetValue(20)
    fmt.Println(d.GetValue())  // 20
}
```

### Pointer vs Value Receiver

Promotion works for **both** value and pointer receivers:

```go
type Base struct {
    x int
}

func (b Base) Get() int { return b.x }
func (b *Base) Set(x int) { b.x = x }

type Derived struct {
    Base
}

func main() {
    d := &Derived{}
    d.Set(42)      // promoted pointer method
    fmt.Println(d.Get())  // 42 (promoted value method)
}
```

---

## Shadowing & Conflicts

### Field Shadowing

If the outer type has a field/method with the **same name**, it **shadows** the embedded one:

```go
type Base struct {
    Name string
}

type Derived struct {
    Base
    Name string  // shadows Base.Name
}

func main() {
    d := Derived{
        Base: Base{Name: "base"},
        Name: "derived",
    }
    
    fmt.Println(d.Name)        // "derived" (outer)
    fmt.Println(d.Base.Name)   // "base" (embedded)
}
```

### Method Shadowing

```go
type Base struct{}

func (Base) Do() {
    fmt.Println("Base.Do")
}

type Derived struct {
    Base
}

func (Derived) Do() {  // shadows Base.Do
    fmt.Println("Derived.Do")
}

func main() {
    d := Derived{}
    d.Do()           // Derived.Do
    d.Base.Do()      // Base.Do
}
```

### Ambiguity Error

If **multiple embedded types** have the same field/method and it’s not shadowed, you get an **ambiguity error**:

```go
type A struct {
    Value int
}

type B struct {
    Value int
}

type C struct {
    A
    B
}

func main() {
    c := C{
        A: A{Value: 1},
        B: B{Value: 2},
    }
    
    // fmt.Println(c.Value)  // ERROR: ambiguous selector c.Value
    fmt.Println(c.A.Value)  // 1
    fmt.Println(c.B.Value)  // 2
}
```

**Fix:** Shadow it in the outer type, or always use the explicit path (`c.A.Value`).

---

## Deep Examples

### Example 1: HTTP Handler with Logging

```go
type Logger interface {
    Log(message string)
}

type ConsoleLogger struct{}

func (ConsoleLogger) Log(message string) {
    fmt.Println("[LOG]", message)
}

type Handler struct {
    Logger  // embedded interface
    Path    string
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.Log(fmt.Sprintf("%s %s", r.Method, r.URL.Path))
    fmt.Fprintf(w, "Hello from %s", h.Path)
}

func main() {
    handler := &Handler{
        Logger: ConsoleLogger{},
        Path:   "/api",
    }
    http.Handle("/api", handler)
}
```

### Example 2: Timestamps Embedding

```go
type Timestamps struct {
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (t *Timestamps) Touch() {
    t.UpdatedAt = time.Now()
}

type User struct {
    Timestamps  // embedded
    ID   int
    Name string
}

type Post struct {
    Timestamps  // embedded
    ID    int
    Title string
}

func main() {
    u := User{
        Timestamps: Timestamps{CreatedAt: time.Now()},
        ID:          1,
        Name:        "Alice",
    }
    
    u.Touch()  // promoted method
    fmt.Println(u.UpdatedAt)
}
```

### Example 3: Multiple Embedding Levels

```go
type Animal struct {
    Name string
}

func (a Animal) Speak() {
    fmt.Printf("%s makes a sound\n", a.Name)
}

type Mammal struct {
    Animal  // embedded
    Legs    int
}

type Dog struct {
    Mammal  // embedded (which embeds Animal)
    Breed   string
}

func (d Dog) Speak() {  // shadows Animal.Speak
    fmt.Printf("%s barks!\n", d.Name)  // Name promoted from Animal
}

func main() {
    d := Dog{
        Mammal: Mammal{
            Animal: Animal{Name: "Buddy"},
            Legs:   4,
        },
        Breed: "Golden Retriever",
    }
    
    d.Speak()           // Buddy barks!
    d.Mammal.Speak()    // Buddy makes a sound
    fmt.Println(d.Name) // Buddy (promoted through Mammal)
}
```

### Example 4: Mutex Embedding

```go
type SafeCounter struct {
    sync.Mutex  // embedded
    count int
}

func (c *SafeCounter) Increment() {
    c.Lock()    // promoted method
    defer c.Unlock()  // promoted method
    c.count++
}

func (c *SafeCounter) Value() int {
    c.Lock()
    defer c.Unlock()
    return c.count
}

func main() {
    c := &SafeCounter{}
    c.Increment()
    fmt.Println(c.Value())  // 1
}
```

### Example 5: io.Reader/Writer Composition

```go
type ReadWriter struct {
    io.Reader  // embedded interface
    io.Writer  // embedded interface
}

func (rw ReadWriter) Copy() (int64, error) {
    return io.Copy(rw, rw)  // rw satisfies both Reader and Writer
}

func main() {
    var buf bytes.Buffer
    rw := ReadWriter{
        Reader: strings.NewReader("hello"),
        Writer: &buf,
    }
    
    rw.Copy()
    fmt.Println(buf.String())  // hello
}
```

### Example 6: Context Embedding

```go
type Request struct {
    context.Context  // embedded
    Method string
    Path   string
}

func (r *Request) WithTimeout(d time.Duration) *Request {
    ctx, cancel := context.WithTimeout(r.Context, d)
    return &Request{
        Context: ctx,
        Method:  r.Method,
        Path:    r.Path,
    }
}

func main() {
    req := &Request{
        Context: context.Background(),
        Method: "GET",
        Path:   "/api",
    }
    
    req2 := req.WithTimeout(5 * time.Second)
    fmt.Println(req2.Method)  // GET
}
```

### Example 7: JSON with Embedded Timestamps

```go
type Timestamps struct {
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type Article struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
    Timestamps  // embedded - fields promoted to top level in JSON
}

func main() {
    a := Article{
        ID:    1,
        Title: "Go Embedding",
        Timestamps: Timestamps{
            CreatedAt: time.Now(),
            UpdatedAt: time.Now(),
        },
    }
    
    b, _ := json.Marshal(a)
    fmt.Println(string(b))
    // {"id":1,"title":"Go Embedding","created_at":"...","updated_at":"..."}
}
```

---

## Embedding vs Composition

### Named Field (Composition)

```go
type Person struct {
    Address Address  // named field
}

p := Person{Address: Address{City: "NYC"}}
fmt.Println(p.Address.City)  // must use field name
```

**Use when:** You want **explicit** access, no promotion, clearer intent.

### Embedded Field (Embedding)

```go
type Person struct {
    Address  // embedded
}

p := Person{Address: Address{City: "NYC"}}
fmt.Println(p.City)  // promoted - direct access
```

**Use when:** You want **promotion** — the embedded type’s fields/methods become part of the outer type’s API.

---

## Best Practices

### ✅ DO:

- **Embed interfaces** to combine method sets (e.g. `ReadWriter`, `ReadWriteCloser`).
- **Embed structs** when you want to inherit fields/methods and make them part of the outer type’s API.
- **Use embedding for “is-a” relationships** (e.g. `Dog` embeds `Animal`).
- **Embed common fields** (e.g. `Timestamps`, `ID`) to avoid repetition.
- **Embed mutexes** for convenience: `sync.Mutex` → `c.Lock()` instead of `c.mu.Lock()`.

### ❌ DON'T:

- **Don’t embed just to save typing** — if you don’t want promotion, use a named field.
- **Don’t create ambiguity** — if multiple embedded types have the same name, shadow it or always use explicit paths.
- **Don’t embed unexported types** from other packages — you can’t embed them anyway.
- **Don’t embed pointers** without ensuring they’re initialized — accessing promoted fields on `nil` panics.
- **Don’t over-embed** — deep embedding chains can make code hard to follow.

---

## Summary

| Concept | Syntax | Effect |
|--------|--------|--------|
| **Struct embedding** | `type A struct { B }` | Fields and methods of `B` promoted to `A` |
| **Interface embedding** | `type I interface { J; K }` | `I` requires both `J` and `K` |
| **Field promotion** | `a.Field` (from embedded `B`) | Direct access without `a.B.Field` |
| **Method promotion** | `a.Method()` (from embedded `B`) | Call without `a.B.Method()` |
| **Shadowing** | Outer type has same name | Outer name takes precedence |
| **Ambiguity** | Multiple embedded types have same name | Must use explicit path (`a.B.Field`) |

**Embedding** enables composition and code reuse while keeping the API clean. Use it when you want the embedded type’s fields/methods to be part of the outer type’s public interface.
