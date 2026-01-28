# Go Interfaces & Structs - Complete Guide

## Table of Contents
- [STRUCT (구조체)](#struct-구조체)
- [INTERFACE (인터페이스)](#interface-인터페이스)
- [Combining Structs & Interfaces](#combining-structs--interfaces)
- [Best Practices](#best-practices)

---

## STRUCT (구조체)

### Basic Struct Definition

```go
type Person struct {
    Name string
    Age  int
    Email string
}

func main() {
    // Method 1: Field by field
    var p1 Person
    p1.Name = "Alice"
    p1.Age = 30
    p1.Email = "alice@example.com"
    
    // Method 2: Literal syntax
    p2 := Person{
        Name: "Bob",
        Age: 25,
        Email: "bob@example.com",
    }
    
    // Method 3: Positional (not recommended)
    p3 := Person{"Charlie", 35, "charlie@example.com"}
    
    fmt.Println(p1, p2, p3)
}
```

### Struct Methods

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Method with value receiver (doesn't modify original)
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Method with pointer receiver (can modify original)
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    
    fmt.Println("Area:", rect.Area())  // 50
    
    rect.Scale(2)
    fmt.Println("After scale:", rect)  // {20 10}
    fmt.Println("New area:", rect.Area())  // 200
}
```

**Key difference:**
- **Value receiver** (`r Rectangle`): Copy, read-only
- **Pointer receiver** (`r *Rectangle`): Original, can modify

### Embedded Structs (Composition)

```go
type Address struct {
    Street string
    City   string
    Zip    string
}

type Person struct {
    Name    string
    Age     int
    Address Address  // Nested
}

// Better: Embedding (anonymous field)
type Employee struct {
    Name   string
    ID     int
    Address  // Embedded - promoted fields
}

func main() {
    // Nested access
    p := Person{
        Name: "Alice",
        Age: 30,
        Address: Address{
            Street: "123 Main St",
            City: "Boston",
            Zip: "02101",
        },
    }
    fmt.Println(p.Address.City)  // Boston
    
    // Embedded - direct access
    e := Employee{
        Name: "Bob",
        ID: 1001,
        Address: Address{
            Street: "456 Oak Ave",
            City: "NYC",
            Zip: "10001",
        },
    }
    fmt.Println(e.City)  // NYC (promoted field!)
    fmt.Println(e.Address.City)  // Also works
}
```

### Struct Tags (메타데이터)

```go
import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email,omitempty"`
    Password string `json:"-"`  // Never serialized
}

func main() {
    u := User{
        ID:       1,
        Name:     "Alice",
        Email:    "",
        Password: "secret123",
    }
    
    data, _ := json.Marshal(u)
    fmt.Println(string(data))
    // Output: {"id":1,"name":"Alice"}
    // Note: Email omitted (empty), Password excluded
}
```

**Common tags:**
- `json:"fieldname"` - JSON serialization
- `xml:"fieldname"` - XML serialization
- `db:"column_name"` - Database mapping
- `validate:"required"` - Validation rules

### Anonymous Structs

```go
func main() {
    // Inline struct definition
    config := struct {
        Host string
        Port int
    }{
        Host: "localhost",
        Port: 8080,
    }
    
    fmt.Printf("%+v\n", config)  // {Host:localhost Port:8080}
}

// Useful for one-off data structures
func getServerInfo() struct {
    Status string
    Uptime int
} {
    return struct {
        Status string
        Uptime int
    }{
        Status: "running",
        Uptime: 3600,
    }
}
```

---

## INTERFACE (인터페이스)

### What is an Interface?

An interface defines **behavior** (method signatures), not data.

```go
// Define interface
type Speaker interface {
    Speak() string
}

// Implement interface (implicit)
type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return "Woof!"
}

type Cat struct {
    Name string
}

func (c Cat) Speak() string {
    return "Meow!"
}

// Function that accepts any Speaker
func MakeSound(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    dog := Dog{Name: "Buddy"}
    cat := Cat{Name: "Whiskers"}
    
    MakeSound(dog)  // Woof!
    MakeSound(cat)  // Meow!
}
```

**No explicit "implements" keyword!** If a type has the required methods, it implements the interface.

### Empty Interface `any`

```go
// any is alias for interface{}
func Print(v any) {
    fmt.Println(v)
}

func main() {
    Print(42)
    Print("hello")
    Print(true)
    Print([]int{1, 2, 3})
    // Accepts anything!
}
```

### Type Assertion

```go
func main() {
    var i any = "hello"
    
    // Type assertion
    s := i.(string)
    fmt.Println(s)  // hello
    
    // Safe type assertion
    s2, ok := i.(string)
    if ok {
        fmt.Println("String:", s2)
    }
    
    // This would panic
    // n := i.(int)  // panic: interface conversion
    
    // Safe version
    n, ok := i.(int)
    if !ok {
        fmt.Println("Not an int")
    }
}
```

### Type Switch

```go
func describe(i any) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    case []int:
        fmt.Printf("Slice of ints: %v\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

func main() {
    describe(42)          // Integer: 42
    describe("hello")     // String: hello
    describe(true)        // Boolean: true
    describe([]int{1,2})  // Slice of ints: [1 2]
    describe(3.14)        // Unknown type: float64
}
```

### Multiple Interfaces

```go
type Reader interface {
    Read() string
}

type Writer interface {
    Write(string)
}

// Composite interface
type ReadWriter interface {
    Reader
    Writer
}

type File struct {
    Content string
}

func (f *File) Read() string {
    return f.Content
}

func (f *File) Write(data string) {
    f.Content = data
}

func main() {
    var rw ReadWriter = &File{}
    
    rw.Write("Hello, World!")
    fmt.Println(rw.Read())  // Hello, World!
}
```

### Common Standard Interfaces

#### 1. `io.Reader` & `io.Writer`

```go
import (
    "fmt"
    "io"
    "strings"
)

func main() {
    // strings.Reader implements io.Reader
    reader := strings.NewReader("Hello, Go!")
    
    buf := make([]byte, 5)
    for {
        n, err := reader.Read(buf)
        if err == io.EOF {
            break
        }
        fmt.Print(string(buf[:n]))
    }
    // Output: Hello, Go!
}
```

#### 2. `fmt.Stringer`

```go
type Person struct {
    Name string
    Age  int
}

// Implement Stringer interface
func (p Person) String() string {
    return fmt.Sprintf("%s (%d years old)", p.Name, p.Age)
}

func main() {
    p := Person{Name: "Alice", Age: 30}
    fmt.Println(p)  // Alice (30 years old)
    // Instead of: {Alice 30}
}
```

#### 3. `error` Interface

```go
type error interface {
    Error() string
}

// Custom error
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return ValidationError{
            Field:   "age",
            Message: "cannot be negative",
        }
    }
    return nil
}

func main() {
    if err := validateAge(-5); err != nil {
        fmt.Println("Error:", err)
        // Error: age: cannot be negative
    }
}
```

---

## Combining Structs & Interfaces

### Example: Shape System

```go
package main

import (
    "fmt"
    "math"
)

// Interface
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Struct 1: Circle
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

// Struct 2: Rectangle
type Rectangle struct {
    Width  float64
    Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Function that works with any Shape
func PrintShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f\n", s.Area())
    fmt.Printf("Perimeter: %.2f\n", s.Perimeter())
}

func main() {
    circle := Circle{Radius: 5}
    rect := Rectangle{Width: 10, Height: 5}
    
    fmt.Println("Circle:")
    PrintShapeInfo(circle)
    
    fmt.Println("\nRectangle:")
    PrintShapeInfo(rect)
    
    // Slice of different shapes
    shapes := []Shape{circle, rect}
    
    totalArea := 0.0
    for _, shape := range shapes {
        totalArea += shape.Area()
    }
    fmt.Printf("\nTotal area: %.2f\n", totalArea)
}
```

**Output:**
```
Circle:
Area: 78.54
Perimeter: 31.42

Rectangle:
Area: 50.00
Perimeter: 30.00

Total area: 128.54
```

### Real-World Example: Payment System

```go
package main

import (
    "fmt"
    "time"
)

// Interface
type PaymentMethod interface {
    Pay(amount float64) error
    Refund(amount float64) error
}

// Struct 1: Credit Card
type CreditCard struct {
    Number     string
    HolderName string
    CVV        string
}

func (cc CreditCard) Pay(amount float64) error {
    fmt.Printf("Charging $%.2f to card ending in %s\n", 
        amount, cc.Number[len(cc.Number)-4:])
    time.Sleep(100 * time.Millisecond)
    return nil
}

func (cc CreditCard) Refund(amount float64) error {
    fmt.Printf("Refunding $%.2f to card ending in %s\n", 
        amount, cc.Number[len(cc.Number)-4:])
    return nil
}

// Struct 2: PayPal
type PayPal struct {
    Email string
}

func (pp PayPal) Pay(amount float64) error {
    fmt.Printf("PayPal payment of $%.2f to %s\n", amount, pp.Email)
    time.Sleep(100 * time.Millisecond)
    return nil
}

func (pp PayPal) Refund(amount float64) error {
    fmt.Printf("PayPal refund of $%.2f to %s\n", amount, pp.Email)
    return nil
}

// Payment processor
type Order struct {
    ID            string
    Amount        float64
    PaymentMethod PaymentMethod
}

func (o *Order) Process() error {
    fmt.Printf("\n=== Processing Order %s ===\n", o.ID)
    return o.PaymentMethod.Pay(o.Amount)
}

func (o *Order) Cancel() error {
    fmt.Printf("\n=== Cancelling Order %s ===\n", o.ID)
    return o.PaymentMethod.Refund(o.Amount)
}

func main() {
    creditCard := CreditCard{
        Number:     "1234567812345678",
        HolderName: "Alice Smith",
        CVV:        "123",
    }
    
    paypal := PayPal{
        Email: "bob@example.com",
    }
    
    order1 := Order{ID: "ORD-001", Amount: 99.99, PaymentMethod: creditCard}
    order2 := Order{ID: "ORD-002", Amount: 149.99, PaymentMethod: paypal}
    
    orders := []*Order{&order1, &order2}
    
    for _, order := range orders {
        if err := order.Process(); err != nil {
            fmt.Printf("Error: %v\n", err)
        }
    }
    
    order2.Cancel()
}
```

---

## Struct vs Interface

| Aspect | Struct | Interface |
|--------|--------|-----------|
| Purpose | Data structure | Behavior contract |
| Contains | Fields (data) | Method signatures |
| Implementation | Explicit definition | Implicit (duck typing) |
| Usage | Create concrete types | Define abstractions |
| Instantiation | Can create instances | Cannot instantiate |

### Advanced Pattern: Embedding Interface in Struct

```go
type Logger interface {
    Log(message string)
}

type ConsoleLogger struct{}

func (ConsoleLogger) Log(message string) {
    fmt.Println("[LOG]", message)
}

// Embed interface in struct
type Service struct {
    Logger  // Embedded interface
    Name string
}

func (s *Service) DoWork() {
    s.Log(fmt.Sprintf("%s is doing work", s.Name))
}

func main() {
    svc := &Service{
        Logger: ConsoleLogger{},
        Name:   "UserService",
    }
    
    svc.DoWork()  // [LOG] UserService is doing work
}
```

---

## Best Practices

### ✅ DO:

```go
// Accept interfaces, return structs (flexible input, concrete output)
func Process(r io.Reader) error  // Good: flexible input

func NewUser(name string) *User   // Good: concrete output

// Use pointer receivers when modifying
func (u *User) UpdateName(name string) {
    u.Name = name
}

// Use value receivers for read-only operations
func (u User) FullName() string {
    return u.FirstName + " " + u.LastName
}
```

### ❌ DON'T:

```go
// Don't store context in struct
type Server struct {
    ctx context.Context  // BAD!
}

// Don't pass nil interface implementations
var w io.Writer = nil  // Can cause nil pointer panic

// Don't use interfaces with only one implementation
// Wait until you have at least two implementations
```

---

## Key Takeaways

**Struct:**
- Define data structures
- Use methods to add behavior
- Value vs pointer receivers matter
- Embedding enables composition
- Tags add metadata

**Interface:**
- Define contracts (behavior)
- Implicit implementation (no keywords)
- Enable polymorphism
- Use type assertion/switch to extract concrete types
- Empty interface accepts anything

**Golden Rule:**

> **Accept interfaces, return structs** (flexible input, concrete output)

```go
// Good: flexible input
func Process(r io.Reader) error

// Good: concrete output
func NewUser(name string) *User
```

This is the foundation of Go's type system! 🚀
