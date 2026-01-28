# Go HTTP - Complete Guide

## Table of Contents
- [HTTP Server Basics](#http-server-basics)
- [Request Handling](#request-handling)
- [Response Handling](#response-handling)
- [Custom Server Configuration](#custom-server-configuration)
- [Middleware](#middleware)
- [HTTP Client](#http-client)
- [Real-World REST API Example](#real-world-rest-api-example)
- [Best Practices](#best-practices)

---

## HTTP Server Basics

### 1. Simple Server

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    // Handler function
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })
    
    // Start server
    fmt.Println("Server running on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Test:**
```bash
curl http://localhost:8080
# Output: Hello, World!
```

### 2. Multiple Routes

```go
func main() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)
    http.HandleFunc("/contact", contactHandler)
    
    fmt.Println("Server running on :8080")
    http.ListenAndServe(":8080", nil)
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Home Page")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "About Page")
}

func contactHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Contact Page")
}
```

### 3. HTTP Methods

```go
func userHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        fmt.Fprintf(w, "GET request")
    case http.MethodPost:
        fmt.Fprintf(w, "POST request")
    case http.MethodPut:
        fmt.Fprintf(w, "PUT request")
    case http.MethodDelete:
        fmt.Fprintf(w, "DELETE request")
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Method not allowed")
    }
}

func main() {
    http.HandleFunc("/user", userHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Test:**
```bash
curl -X GET http://localhost:8080/user
curl -X POST http://localhost:8080/user
curl -X PUT http://localhost:8080/user
curl -X DELETE http://localhost:8080/user
```

---

## Request Handling

### 1. URL Parameters (Query String)

```go
func searchHandler(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    query := r.URL.Query()
    
    q := query.Get("q")           // Single value
    page := query.Get("page")     // Single value
    tags := query["tag"]          // Multiple values
    
    fmt.Fprintf(w, "Search: %s\n", q)
    fmt.Fprintf(w, "Page: %s\n", page)
    fmt.Fprintf(w, "Tags: %v\n", tags)
}

func main() {
    http.HandleFunc("/search", searchHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Test:**
```bash
curl "http://localhost:8080/search?q=golang&page=1&tag=web&tag=server"
# Output:
# Search: golang
# Page: 1
# Tags: [web server]
```

### 2. Path Parameters (Manual)

```go
import (
    "net/http"
    "strings"
)

func userHandler(w http.ResponseWriter, r *http.Request) {
    // Extract ID from path: /user/123
    path := strings.TrimPrefix(r.URL.Path, "/user/")
    
    if path == "" {
        http.Error(w, "User ID required", http.StatusBadRequest)
        return
    }
    
    fmt.Fprintf(w, "User ID: %s", path)
}

func main() {
    http.HandleFunc("/user/", userHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Test:**
```bash
curl http://localhost:8080/user/123
# Output: User ID: 123
```

### 3. Reading Request Body (JSON)

```go
import (
    "encoding/json"
    "fmt"
    "net/http"
)

type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    var user User
    
    // Decode JSON body
    decoder := json.NewDecoder(r.Body)
    err := decoder.Decode(&user)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    // Process user...
    fmt.Fprintf(w, "Created user: %s (%s)", user.Name, user.Email)
}

func main() {
    http.HandleFunc("/user", createUserHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Test:**
```bash
curl -X POST http://localhost:8080/user \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","age":30}'
```

### 4. Reading Headers

```go
func headerHandler(w http.ResponseWriter, r *http.Request) {
    // Get specific header
    userAgent := r.Header.Get("User-Agent")
    auth := r.Header.Get("Authorization")
    
    // Get all values for a header
    accepts := r.Header["Accept"]
    
    fmt.Fprintf(w, "User-Agent: %s\n", userAgent)
    fmt.Fprintf(w, "Authorization: %s\n", auth)
    fmt.Fprintf(w, "Accept: %v\n", accepts)
    
    // Iterate all headers
    fmt.Fprintf(w, "\nAll Headers:\n")
    for key, values := range r.Header {
        fmt.Fprintf(w, "%s: %v\n", key, values)
    }
}
```

---

## Response Handling

### 1. Setting Status Codes

```go
func statusHandler(w http.ResponseWriter, r *http.Request) {
    // Set status code
    w.WriteHeader(http.StatusCreated)  // 201
    fmt.Fprintf(w, "Resource created")
}

func notFoundHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNotFound)  // 404
    fmt.Fprintf(w, "Not found")
}

func errorHandler(w http.ResponseWriter, r *http.Request) {
    // Helper for errors
    http.Error(w, "Internal server error", http.StatusInternalServerError)
}
```

**Common Status Codes:**
```go
http.StatusOK                  // 200
http.StatusCreated             // 201
http.StatusNoContent           // 204
http.StatusBadRequest          // 400
http.StatusUnauthorized        // 401
http.StatusForbidden           // 403
http.StatusNotFound            // 404
http.StatusMethodNotAllowed    // 405
http.StatusInternalServerError // 500
```

### 2. JSON Response

```go
type Response struct {
    Success bool        `json:"success"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func jsonHandler(w http.ResponseWriter, r *http.Request) {
    resp := Response{
        Success: true,
        Message: "Data retrieved successfully",
        Data: map[string]interface{}{
            "users": []string{"Alice", "Bob"},
            "count": 2,
        },
    }
    
    // Set content type
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    
    // Encode JSON
    json.NewEncoder(w).Encode(resp)
}
```

**Test:**
```bash
curl http://localhost:8080/api/data
# Output: {"success":true,"message":"Data retrieved successfully","data":{"count":2,"users":["Alice","Bob"]}}
```

### 3. Setting Response Headers

```go
func headersHandler(w http.ResponseWriter, r *http.Request) {
    // Set headers (must be before WriteHeader)
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom-Header", "MyValue")
    w.Header().Set("Cache-Control", "no-cache")
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

---

## Custom Server Configuration

```go
import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", homeHandler)
    mux.HandleFunc("/api/users", usersHandler)
    
    // Custom server configuration
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
        MaxHeaderBytes: 1 << 20, // 1 MB
    }
    
    // Start server in goroutine
    go func() {
        log.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt)
    <-quit
    
    log.Println("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Server shutdown error: %v", err)
    }
    
    log.Println("Server stopped")
}
```

---

## Middleware

### 1. Basic Middleware

```go
// Logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        
        next.ServeHTTP(w, r)
        
        log.Printf("Completed in %v", time.Since(start))
    })
}

// Authentication middleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        if token != "Bearer secret-token" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", homeHandler)
    mux.HandleFunc("/api/protected", protectedHandler)
    
    // Apply middleware
    handler := loggingMiddleware(authMiddleware(mux))
    
    http.ListenAndServe(":8080", handler)
}
```

### 2. Chain Multiple Middleware

```go
type Middleware func(http.Handler) http.Handler

func chainMiddleware(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", homeHandler)
    
    handler := chainMiddleware(
        mux,
        loggingMiddleware,
        authMiddleware,
        corsMiddleware,
    )
    
    http.ListenAndServe(":8080", handler)
}

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

## HTTP Client

### 1. Simple GET Request

```go
import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    resp, err := http.Get("https://api.github.com/users/golang")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Status:", resp.Status)
    fmt.Println("Body:", string(body))
}
```

### 2. POST Request with JSON

```go
func main() {
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
    }
    
    jsonData, _ := json.Marshal(user)
    
    resp, err := http.Post(
        "http://localhost:8080/user",
        "application/json",
        bytes.NewBuffer(jsonData),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### 3. Custom Request with Headers

```go
func main() {
    // Create request
    req, err := http.NewRequest("GET", "https://api.example.com/data", nil)
    if err != nil {
        log.Fatal(err)
    }
    
    // Add headers
    req.Header.Set("Authorization", "Bearer token123")
    req.Header.Set("User-Agent", "MyApp/1.0")
    
    // Send request
    client := &http.Client{
        Timeout: 10 * time.Second,
    }
    
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### 4. Custom HTTP Client

```go
func main() {
    client := &http.Client{
        Timeout: 30 * time.Second,
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            IdleConnTimeout:     90 * time.Second,
        },
    }
    
    req, _ := http.NewRequest("GET", "https://api.example.com", nil)
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
}
```

---

## Real-World REST API Example

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
    "sync"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

type UserStore struct {
    mu    sync.RWMutex
    users map[int]User
    nextID int
}

func NewUserStore() *UserStore {
    return &UserStore{
        users: make(map[int]User),
        nextID: 1,
    }
}

func (s *UserStore) Create(user User) User {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    user.ID = s.nextID
    s.users[user.ID] = user
    s.nextID++
    
    return user
}

func (s *UserStore) GetAll() []User {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    users := make([]User, 0, len(s.users))
    for _, user := range s.users {
        users = append(users, user)
    }
    return users
}

func (s *UserStore) Get(id int) (User, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    user, exists := s.users[id]
    return user, exists
}

func (s *UserStore) Update(id int, user User) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if _, exists := s.users[id]; !exists {
        return false
    }
    
    user.ID = id
    s.users[id] = user
    return true
}

func (s *UserStore) Delete(id int) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if _, exists := s.users[id]; !exists {
        return false
    }
    
    delete(s.users, id)
    return true
}

type Server struct {
    store *UserStore
}

func NewServer() *Server {
    return &Server{
        store: NewUserStore(),
    }
}

func (s *Server) usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        s.handleGetUsers(w, r)
    case http.MethodPost:
        s.handleCreateUser(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (s *Server) userHandler(w http.ResponseWriter, r *http.Request) {
    // Extract ID from path
    idStr := strings.TrimPrefix(r.URL.Path, "/users/")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    switch r.Method {
    case http.MethodGet:
        s.handleGetUser(w, r, id)
    case http.MethodPut:
        s.handleUpdateUser(w, r, id)
    case http.MethodDelete:
        s.handleDeleteUser(w, r, id)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (s *Server) handleGetUsers(w http.ResponseWriter, r *http.Request) {
    users := s.store.GetAll()
    respondJSON(w, http.StatusOK, users)
}

func (s *Server) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    created := s.store.Create(user)
    respondJSON(w, http.StatusCreated, created)
}

func (s *Server) handleGetUser(w http.ResponseWriter, r *http.Request, id int) {
    user, exists := s.store.Get(id)
    if !exists {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    respondJSON(w, http.StatusOK, user)
}

func (s *Server) handleUpdateUser(w http.ResponseWriter, r *http.Request, id int) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    if !s.store.Update(id, user) {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    user.ID = id
    respondJSON(w, http.StatusOK, user)
}

func (s *Server) handleDeleteUser(w http.ResponseWriter, r *http.Request, id int) {
    if !s.store.Delete(id) {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    w.WriteHeader(http.StatusNoContent)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func main() {
    server := NewServer()
    
    http.HandleFunc("/users", server.usersHandler)
    http.HandleFunc("/users/", server.userHandler)
    
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Test the API:**
```bash
# Create user
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# Get all users
curl http://localhost:8080/users

# Get specific user
curl http://localhost:8080/users/1

# Update user
curl -X PUT http://localhost:8080/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Smith","email":"alice.smith@example.com"}'

# Delete user
curl -X DELETE http://localhost:8080/users/1
```

---

## Summary

| Feature | Code |
|---------|------|
| Simple server | `http.ListenAndServe(":8080", nil)` |
| Route handler | `http.HandleFunc("/path", handler)` |
| Read JSON | `json.NewDecoder(r.Body).Decode(&data)` |
| Write JSON | `json.NewEncoder(w).Encode(data)` |
| Query params | `r.URL.Query().Get("key")` |
| Headers | `r.Header.Get("Key")` / `w.Header().Set("Key", "val")` |
| Status code | `w.WriteHeader(http.StatusOK)` |
| Middleware | `func(http.Handler) http.Handler` |
| HTTP client | `http.Get(url)` / `client.Do(req)` |

Go's `net/http` is powerful, production-ready, and requires no external dependencies! 🚀
