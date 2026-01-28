# Go Capitalization - Visibility & Naming Rules

## Table of Contents
- [The Rule](#the-rule)
- [Variables](#variables)
- [Functions](#functions)
- [Structs & Fields](#structs--fields)
- [Methods](#methods)
- [Interfaces](#interfaces)
- [Constants](#constants)
- [Type Aliases](#type-aliases)
- [Real-World Example](#real-world-example)
- [Naming Conventions](#naming-conventions)
- [Best Practices](#best-practices)

---

## The Rule

In Go, **capitalization determines visibility** (public vs private).

| Capitalization | Visibility | Accessible from |
|----------------|-----------|-----------------|
| **Uppercase (Capital)** | Public (Exported) | Other packages |
| **lowercase (Non-capital)** | Private (Unexported) | Same package only |

---

## Variables

### Exported (Public)

```go
package mypackage

var PublicVar = "visible everywhere"
var Counter = 100
var DatabaseURL = "localhost:5432"
```

**Can be accessed from other packages:**

```go
package main

import "myproject/mypackage"

func main() {
    fmt.Println(mypackage.PublicVar)    // ✅ OK
    fmt.Println(mypackage.Counter)      // ✅ OK
    fmt.Println(mypackage.DatabaseURL)  // ✅ OK
}
```

### Unexported (Private)

```go
package mypackage

var privateVar = "only visible in mypackage"
var counter = 0
var databaseURL = "localhost:5432"
```

**Cannot be accessed from other packages:**

```go
package main

import "myproject/mypackage"

func main() {
    fmt.Println(mypackage.privateVar)   // ❌ ERROR: undefined
    fmt.Println(mypackage.counter)      // ❌ ERROR: undefined
    fmt.Println(mypackage.databaseURL)  // ❌ ERROR: undefined
}
```

---

## Functions

### Exported (Public)

```go
package math

// Public function - can be called from other packages
func Add(a, b int) int {
    return a + b
}

func Multiply(a, b int) int {
    return a * b
}
```

**Usage:**

```go
package main

import "myproject/math"

func main() {
    result := math.Add(5, 3)      // ✅ OK
    product := math.Multiply(4, 2) // ✅ OK
}
```

### Unexported (Private)

```go
package math

// Private function - only usable within math package
func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

// Public function that uses private functions
func Calculate(a, b int) int {
    sum := add(a, b)         // ✅ OK (same package)
    return multiply(sum, 2)  // ✅ OK (same package)
}
```

**Usage:**

```go
package main

import "myproject/math"

func main() {
    result := math.Calculate(5, 3)  // ✅ OK (public)
    // sum := math.add(5, 3)        // ❌ ERROR: undefined
    // prod := math.multiply(4, 2)  // ❌ ERROR: undefined
}
```

---

## Structs & Fields

### Struct Visibility

```go
package user

// Exported struct
type User struct {
    Name  string  // Exported field
    Email string  // Exported field
    age   int     // Unexported field (private)
}

// Unexported struct
type session struct {
    token   string
    expires time.Time
}
```

**Usage:**

```go
package main

import "myproject/user"

func main() {
    // Can create User (exported)
    u := user.User{
        Name:  "Alice",
        Email: "alice@example.com",
        // age: 30,  // ❌ ERROR: unknown field (unexported)
    }
    
    fmt.Println(u.Name)   // ✅ OK
    fmt.Println(u.Email)  // ✅ OK
    // fmt.Println(u.age) // ❌ ERROR: u.age undefined
    
    // Cannot create session (unexported)
    // s := user.session{}  // ❌ ERROR: undefined
}
```

### Mixed Visibility Pattern

```go
package account

type Account struct {
    ID       string  // Public
    Balance  float64 // Public
    password string  // Private (hidden from other packages)
}

// Public method
func (a *Account) Deposit(amount float64) {
    a.Balance += amount
}

// Private method
func (a *Account) validatePassword(pass string) bool {
    return a.password == pass
}

// Public method using private method
func (a *Account) Withdraw(amount float64, pass string) error {
    if !a.validatePassword(pass) {  // ✅ OK (same package)
        return errors.New("invalid password")
    }
    if a.Balance < amount {
        return errors.New("insufficient funds")
    }
    a.Balance -= amount
    return nil
}
```

---

## Methods

```go
package car

type Car struct {
    Brand string
    speed int  // Private
}

// Exported method
func (c *Car) Start() {
    c.speed = 0
    c.ignite()  // Can call private method
}

// Unexported method
func (c *Car) ignite() {
    fmt.Println("Engine starting...")
}

// Exported getter for private field
func (c *Car) GetSpeed() int {
    return c.speed
}

// Exported setter for private field
func (c *Car) SetSpeed(s int) {
    c.speed = s
}
```

---

## Interfaces

```go
package storage

// Exported interface
type Storage interface {
    Save(key, value string) error
    Load(key string) (string, error)
}

// Unexported interface
type cache interface {
    get(key string) (string, bool)
    set(key, value string)
}
```

---

## Constants

```go
package config

// Exported constants
const MaxConnections = 100
const DefaultTimeout = 30

// Unexported constants
const apiKey = "secret-key-123"
const retryAttempts = 3
```

---

## Type Aliases

```go
package types

// Exported type alias
type UserID string
type OrderID int

// Unexported type alias
type sessionID string
```

---

## Real-World Example

### Package Structure

```
myapp/
├── main.go
└── user/
    └── user.go
```

### user/user.go (Package)

```go
package user

import (
    "errors"
    "fmt"
    "time"
)

// Exported struct with mixed visibility fields
type User struct {
    ID        string    // Public
    Username  string    // Public
    Email     string    // Public
    CreatedAt time.Time // Public
    password  string    // Private
    loginCount int      // Private
}

// Exported constructor (factory function)
func New(username, email, password string) (*User, error) {
    if !isValidEmail(email) {  // Uses private function
        return nil, errors.New("invalid email")
    }
    
    return &User{
        ID:        generateID(),  // Uses private function
        Username:  username,
        Email:     email,
        CreatedAt: time.Now(),
        password:  hashPassword(password),  // Uses private function
        loginCount: 0,
    }, nil
}

// Exported method
func (u *User) Login(password string) error {
    if u.password != hashPassword(password) {
        return errors.New("invalid password")
    }
    u.incrementLoginCount()  // Uses private method
    return nil
}

// Exported getter
func (u *User) GetLoginCount() int {
    return u.loginCount
}

// Private method
func (u *User) incrementLoginCount() {
    u.loginCount++
}

// Private helper functions
func isValidEmail(email string) bool {
    return len(email) > 5 && contains(email, "@")
}

func generateID() string {
    return fmt.Sprintf("user_%d", time.Now().UnixNano())
}

func hashPassword(password string) string {
    // Simplified - use real hashing in production
    return fmt.Sprintf("hashed_%s", password)
}

func contains(s, substr string) bool {
    return len(s) > 0 && len(substr) > 0
}
```

### main.go (Usage)

```go
package main

import (
    "fmt"
    "myapp/user"
)

func main() {
    // ✅ Can use exported New function
    u, err := user.New("alice", "alice@example.com", "secret123")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    // ✅ Can access exported fields
    fmt.Println("User ID:", u.ID)
    fmt.Println("Username:", u.Username)
    fmt.Println("Email:", u.Email)
    
    // ❌ Cannot access unexported fields
    // fmt.Println(u.password)    // ERROR: u.password undefined
    // fmt.Println(u.loginCount)  // ERROR: u.loginCount undefined
    
    // ✅ Can call exported methods
    err = u.Login("secret123")
    if err != nil {
        fmt.Println("Login failed:", err)
        return
    }
    
    // ✅ Can use exported getter
    fmt.Println("Login count:", u.GetLoginCount())
    
    // ❌ Cannot call unexported methods
    // u.incrementLoginCount()  // ERROR: undefined
    
    // ❌ Cannot use unexported functions
    // valid := user.isValidEmail("test@test.com")  // ERROR: undefined
}
```

---

## Naming Conventions

### Variables

```go
// Exported
var GlobalConfig Config
var HTTPClient *http.Client
var APIEndpoint string

// Unexported
var defaultTimeout time.Duration
var connectionPool *Pool
var cacheSize int
```

### Functions

```go
// Exported - describe what they do
func CreateUser(name string) *User
func ValidateInput(data string) bool
func SendEmail(to, subject, body string) error

// Unexported - internal helpers
func parseConfig() Config
func initDatabase() error
func logError(err error)
```

### Acronyms

```go
// Correct: Keep acronyms consistent
var HTTPServer *http.Server  // HTTP, not Http
var APIKey string             // API, not Api
var URLParser Parser          // URL, not Url
var IDGenerator Generator     // ID, not Id

// But for unexported:
var httpClient *http.Client
var apiEndpoint string
var urlPattern string
```

---

## Common Patterns

### Pattern 1: Public API with Private Implementation

```go
package cache

// Public interface
type Cache interface {
    Get(key string) (interface{}, bool)
    Set(key string, value interface{})
}

// Private implementation
type memoryCache struct {
    data map[string]interface{}
}

// Public constructor
func NewCache() Cache {
    return &memoryCache{
        data: make(map[string]interface{}),
    }
}

func (m *memoryCache) Get(key string) (interface{}, bool) {
    val, ok := m.data[key]
    return val, ok
}

func (m *memoryCache) Set(key string, value interface{}) {
    m.data[key] = value
}
```

### Pattern 2: Getter/Setter Pattern

```go
package person

type Person struct {
    name string  // Private
    age  int     // Private
}

// Getters (exported)
func (p *Person) Name() string {
    return p.name
}

func (p *Person) Age() int {
    return p.age
}

// Setters (exported)
func (p *Person) SetName(name string) {
    p.name = name
}

func (p *Person) SetAge(age int) {
    if age >= 0 {
        p.age = age
    }
}
```

---

## Summary

| Item | Exported (Public) | Unexported (Private) |
|------|------------------|---------------------|
| Variable | `var Counter int` | `var counter int` |
| Function | `func Calculate()` | `func calculate()` |
| Struct | `type User struct{}` | `type user struct{}` |
| Field | `Name string` | `name string` |
| Method | `func (u *User) Save()` | `func (u *user) save()` |
| Interface | `type Writer interface{}` | `type writer interface{}` |
| Constant | `const MaxSize = 100` | `const maxSize = 100` |

---

## Key Points

- **Capital first letter** = Visible outside package
- **Lowercase first letter** = Only visible within package
- **No public/private keywords** in Go
- Applies to: variables, functions, types, struct fields, methods
- **Package is the unit of encapsulation**

This is Go's way of information hiding and API design! 🔐
