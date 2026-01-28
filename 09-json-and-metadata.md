# Go JSON & Metadata (Struct Tags) - Complete Guide

## Table of Contents
- [Basic JSON Operations](#basic-json-operations)
- [JSON Struct Tags](#json-struct-tags)
- [JSON Tag Options Explained](#json-tag-options-explained)
- [Nested Structures](#nested-structures)
- [Pointers & nil Values](#pointers--nil-values)
- [Multiple Struct Tags](#multiple-struct-tags)
- [Custom JSON Marshaling](#custom-json-marshaling)
- [Real-World API Example](#real-world-api-example)
- [Best Practices](#best-practices)

---

## Basic JSON Operations

### 1. Marshal (Go → JSON)

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    Name  string
    Email string
    Age   int
}

func main() {
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   30,
    }
    
    // Convert struct to JSON
    jsonData, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Println(string(jsonData))
    // Output: {"Name":"Alice","Email":"alice@example.com","Age":30}
}
```

### 2. Unmarshal (JSON → Go)

```go
func main() {
    jsonStr := `{"Name":"Bob","Email":"bob@example.com","Age":25}`
    
    var user User
    err := json.Unmarshal([]byte(jsonStr), &user)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    fmt.Printf("%+v\n", user)
    // Output: {Name:Bob Email:bob@example.com Age:25}
}
```

### 3. Pretty Print (Indented)

```go
func main() {
    user := User{Name: "Alice", Email: "alice@example.com", Age: 30}
    
    jsonData, _ := json.MarshalIndent(user, "", "  ")
    fmt.Println(string(jsonData))
}
```

**Output:**
```json
{
  "Name": "Alice",
  "Email": "alice@example.com",
  "Age": 30
}
```

---

## JSON Struct Tags

### Basic Syntax

```go
type FieldName TypeName `json:"json_name"`
```

### Common Tag Options

```go
type User struct {
    // Custom JSON field name
    Name string `json:"name"`
    
    // Omit if empty (zero value)
    Email string `json:"email,omitempty"`
    
    // Never include in JSON
    Password string `json:"-"`
    
    // Keep field name, but omit if empty
    Phone string `json:",omitempty"`
    
    // String encoding for number
    ID int `json:"id,string"`
}

func main() {
    user := User{
        Name:     "Alice",
        Email:    "",  // Empty - will be omitted
        Password: "secret123",
        Phone:    "",  // Empty - will be omitted
        ID:       42,
    }
    
    jsonData, _ := json.Marshal(user)
    fmt.Println(string(jsonData))
    // Output: {"name":"Alice","id":"42"}
    // Note: Email, Password, Phone not included
}
```

---

## JSON Tag Options Explained

### 1. Custom Field Names

```go
type Product struct {
    ID    int    `json:"product_id"`
    Name  string `json:"product_name"`
    Price float64 `json:"price_usd"`
}

func main() {
    p := Product{ID: 1, Name: "Laptop", Price: 999.99}
    jsonData, _ := json.Marshal(p)
    fmt.Println(string(jsonData))
    // Output: {"product_id":1,"product_name":"Laptop","price_usd":999.99}
}
```

### 2. omitempty - Skip Zero Values

```go
type Response struct {
    Success bool   `json:"success"`
    Message string `json:"message,omitempty"`
    Data    []int  `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
}

func main() {
    // All fields set
    r1 := Response{
        Success: true,
        Message: "OK",
        Data:    []int{1, 2, 3},
    }
    json1, _ := json.Marshal(r1)
    fmt.Println(string(json1))
    // {"success":true,"message":"OK","data":[1,2,3]}
    
    // Empty fields omitted
    r2 := Response{
        Success: false,
        // Message, Data, Error are empty
    }
    json2, _ := json.Marshal(r2)
    fmt.Println(string(json2))
    // {"success":false}
}
```

**Zero values for omitempty:**
- `false` for bool
- `0` for numeric types
- `""` for strings
- `nil` for pointers, slices, maps, interfaces
- Empty structs

### 3. `-` Skip Field Entirely

```go
type User struct {
    Username string `json:"username"`
    Password string `json:"-"`  // Never serialized
    APIKey   string `json:"-"`  // Never serialized
}

func main() {
    u := User{
        Username: "alice",
        Password: "secret123",
        APIKey:   "key-abc-123",
    }
    
    jsonData, _ := json.Marshal(u)
    fmt.Println(string(jsonData))
    // Output: {"username":"alice"}
    // Password and APIKey not included
}
```

### 4. `string` - Encode as String

```go
type Order struct {
    ID     int64   `json:"id,string"`
    Total  float64 `json:"total,string"`
    Active bool    `json:"active,string"`
}

func main() {
    o := Order{ID: 123, Total: 99.99, Active: true}
    jsonData, _ := json.Marshal(o)
    fmt.Println(string(jsonData))
    // Output: {"id":"123","total":"99.99","active":"true"}
}
```

---

## Nested Structures

```go
type Address struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    Country string `json:"country"`
}

type Person struct {
    Name    string  `json:"name"`
    Age     int     `json:"age"`
    Address Address `json:"address"`
}

func main() {
    p := Person{
        Name: "Alice",
        Age:  30,
        Address: Address{
            Street:  "123 Main St",
            City:    "Boston",
            Country: "USA",
        },
    }
    
    jsonData, _ := json.MarshalIndent(p, "", "  ")
    fmt.Println(string(jsonData))
}
```

**Output:**
```json
{
  "name": "Alice",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "Boston",
    "country": "USA"
  }
}
```

---

## Embedded Structs

```go
type Timestamps struct {
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type Article struct {
    ID      int    `json:"id"`
    Title   string `json:"title"`
    Content string `json:"content"`
    Timestamps  // Embedded - fields promoted to top level
}

func main() {
    a := Article{
        ID:      1,
        Title:   "Go Tutorial",
        Content: "Learning Go...",
        Timestamps: Timestamps{
            CreatedAt: time.Now(),
            UpdatedAt: time.Now(),
        },
    }
    
    jsonData, _ := json.MarshalIndent(a, "", "  ")
    fmt.Println(string(jsonData))
}
```

**Output:**
```json
{
  "id": 1,
  "title": "Go Tutorial",
  "content": "Learning Go...",
  "created_at": "2026-01-27T10:30:00Z",
  "updated_at": "2026-01-27T10:30:00Z"
}
```

---

## Pointers & nil Values

```go
type User struct {
    Name  string  `json:"name"`
    Email *string `json:"email,omitempty"`  // Pointer allows nil
    Age   *int    `json:"age,omitempty"`
}

func main() {
    email := "alice@example.com"
    age := 30
    
    u1 := User{
        Name:  "Alice",
        Email: &email,
        Age:   &age,
    }
    json1, _ := json.Marshal(u1)
    fmt.Println(string(json1))
    // {"name":"Alice","email":"alice@example.com","age":30}
    
    u2 := User{
        Name: "Bob",
        // Email and Age are nil
    }
    json2, _ := json.Marshal(u2)
    fmt.Println(string(json2))
    // {"name":"Bob"}
}
```

**Pointer vs Value:**
- **Value:** Can't distinguish between "not set" and "zero value"
- **Pointer:** `nil` means "not set", value means "explicitly set"

---

## Multiple Struct Tags

```go
import (
    "encoding/json"
    "encoding/xml"
)

type User struct {
    ID       int    `json:"id" xml:"id" db:"user_id"`
    Username string `json:"username" xml:"username" db:"username"`
    Email    string `json:"email,omitempty" xml:"email" db:"email"`
}

func main() {
    u := User{ID: 1, Username: "alice", Email: "alice@example.com"}
    
    // JSON
    jsonData, _ := json.Marshal(u)
    fmt.Println("JSON:", string(jsonData))
    
    // XML
    xmlData, _ := xml.Marshal(u)
    fmt.Println("XML:", string(xmlData))
}
```

**Output:**
```
JSON: {"id":1,"username":"alice","email":"alice@example.com"}
XML: <User><id>1</id><username>alice</username><email>alice@example.com</email></User>
```

---

## Custom JSON Marshaling

### Method 1: Implement MarshalJSON

```go
type User struct {
    Name      string
    Email     string
    Password  string
    CreatedAt time.Time
}

// Custom JSON marshaling
func (u User) MarshalJSON() ([]byte, error) {
    type Alias User  // Avoid recursion
    return json.Marshal(&struct {
        *Alias
        Password  string `json:"-"`  // Exclude password
        CreatedAt string `json:"created_at"`
    }{
        Alias:     (*Alias)(&u),
        CreatedAt: u.CreatedAt.Format("2006-01-02"),
    })
}

func main() {
    u := User{
        Name:      "Alice",
        Email:     "alice@example.com",
        Password:  "secret123",
        CreatedAt: time.Now(),
    }
    
    jsonData, _ := json.Marshal(u)
    fmt.Println(string(jsonData))
    // Output: {"Name":"Alice","Email":"alice@example.com","created_at":"2026-01-27"}
}
```

### Method 2: Implement UnmarshalJSON

```go
type Date struct {
    time.Time
}

// Custom unmarshaling for date format
func (d *Date) UnmarshalJSON(data []byte) error {
    str := string(data)
    str = str[1 : len(str)-1]  // Remove quotes
    
    t, err := time.Parse("2006-01-02", str)
    if err != nil {
        return err
    }
    
    d.Time = t
    return nil
}

type Event struct {
    Name string `json:"name"`
    Date Date   `json:"date"`
}

func main() {
    jsonStr := `{"name":"Conference","date":"2026-03-15"}`
    
    var event Event
    json.Unmarshal([]byte(jsonStr), &event)
    
    fmt.Printf("%+v\n", event)
    // Output: {Name:Conference Date:{Time:2026-03-15 00:00:00 +0000 UTC}}
}
```

---

## Real-World API Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

// Request payload
type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

// Response payload
type UserResponse struct {
    ID        int       `json:"id"`
    Username  string    `json:"username"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    // Password never exposed
}

// Error response
type ErrorResponse struct {
    Error   string            `json:"error"`
    Details map[string]string `json:"details,omitempty"`
}

// Success response with data
type APIResponse struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *string     `json:"error,omitempty"`
}

func main() {
    // Simulate API request
    reqJSON := `{
        "username": "alice",
        "email": "alice@example.com",
        "password": "secret123"
    }`
    
    var req CreateUserRequest
    json.Unmarshal([]byte(reqJSON), &req)
    
    // Simulate creating user
    user := UserResponse{
        ID:        1,
        Username:  req.Username,
        Email:     req.Email,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    
    // Success response
    resp := APIResponse{
        Success: true,
        Data:    user,
    }
    
    respJSON, _ := json.MarshalIndent(resp, "", "  ")
    fmt.Println(string(respJSON))
}
```

**Output:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "username": "alice",
    "email": "alice@example.com",
    "created_at": "2026-01-27T10:30:00Z",
    "updated_at": "2026-01-27T10:30:00Z"
  }
}
```

---

## Common Patterns & Best Practices

### Pattern 1: Separate Internal & External Models

```go
// Internal model (database)
type userModel struct {
    ID           int
    Username     string
    Email        string
    PasswordHash string
    CreatedAt    time.Time
}

// External model (API)
type UserDTO struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

// Convert internal → external
func (u *userModel) ToDTO() UserDTO {
    return UserDTO{
        ID:       u.ID,
        Username: u.Username,
        Email:    u.Email,
        // Password not exposed
    }
}
```

### Pattern 2: Handle Unknown Fields

```go
type Config struct {
    Host string `json:"host"`
    Port int    `json:"port"`
    // Store unknown fields
    Extra map[string]interface{} `json:"-"`
}

func (c *Config) UnmarshalJSON(data []byte) error {
    type Alias Config
    aux := &struct {
        *Alias
    }{
        Alias: (*Alias)(c),
    }
    
    if err := json.Unmarshal(data, aux); err != nil {
        return err
    }
    
    // Parse into map to capture unknowns
    var raw map[string]interface{}
    json.Unmarshal(data, &raw)
    
    delete(raw, "host")
    delete(raw, "port")
    c.Extra = raw
    
    return nil
}
```

### Pattern 3: JSON Streaming (Large Data)

```go
import (
    "encoding/json"
    "os"
)

func writeUsers(users []User) error {
    file, _ := os.Create("users.json")
    defer file.Close()
    
    encoder := json.NewEncoder(file)
    encoder.SetIndent("", "  ")
    
    return encoder.Encode(users)
}

func readUsers() ([]User, error) {
    file, _ := os.Open("users.json")
    defer file.Close()
    
    var users []User
    decoder := json.NewDecoder(file)
    
    return users, decoder.Decode(&users)
}
```

---

## Summary Table

| Tag | Purpose | Example |
|-----|---------|---------|
| `json:"name"` | Custom field name | `ID int json:"user_id"` |
| `,omitempty` | Skip if zero value | `Email string json:"email,omitempty"` |
| `-` | Never include | `Password string json:"-"` |
| `,string` | Encode as string | `ID int json:"id,string"` |
| Multiple tags | Different encodings | `Name string json:"name" xml:"Name"` |

---

## Key Points

- **Exported fields only** (capitalized) are serialized
- Use `omitempty` for optional fields
- Use `-` for sensitive data
- Use pointers to distinguish `nil` vs zero
- Custom marshaling for complex transformations
- Separate internal/external models for security

JSON tags are essential for API development in Go! 🚀
