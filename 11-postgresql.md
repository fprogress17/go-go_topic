# Go PostgreSQL - Complete Guide

## Table of Contents
- [Setup & Installation](#setup--installation)
- [Using database/sql with pq](#using-databasesql-with-pq)
- [Using pgx (Recommended)](#using-pgx-recommended)
- [Transactions](#transactions)
- [Real-World REST API Example](#real-world-rest-api-example)
- [Best Practices](#best-practices)

---

## Setup & Installation

### 1. Install PostgreSQL Driver

```bash
# Standard library database/sql interface
go get github.com/lib/pq

# OR use pgx (more features, better performance)
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
```

**Recommended:** `pgx` for production (better performance, more PostgreSQL features)

### 2. Database Connection String

```go
// Format
"postgres://username:password@localhost:5432/database_name?sslmode=disable"

// Example
"postgres://myuser:mypassword@localhost:5432/mydb?sslmode=disable"

// With options
"postgres://user:pass@host:5432/db?sslmode=require&pool_max_conns=10"
```

---

## Using database/sql with pq

### 1. Basic Connection

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/lib/pq"  // PostgreSQL driver
)

func main() {
    // Connection string
    connStr := "postgres://postgres:password@localhost:5432/testdb?sslmode=disable"
    
    // Open connection
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // Test connection
    err = db.Ping()
    if err != nil {
        log.Fatal("Cannot connect to database:", err)
    }
    
    fmt.Println("Successfully connected to database!")
}
```

### 2. Create Table

```go
func createTable(db *sql.DB) error {
    query := `
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        age INT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )`
    
    _, err := db.Exec(query)
    return err
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    if err := createTable(db); err != nil {
        log.Fatal(err)
    }
    fmt.Println("Table created successfully")
}
```

### 3. Insert Data

```go
type User struct {
    ID        int
    Name      string
    Email     string
    Age       int
    CreatedAt time.Time
}

func insertUser(db *sql.DB, user User) (int, error) {
    query := `
        INSERT INTO users (name, email, age) 
        VALUES ($1, $2, $3) 
        RETURNING id
    `
    
    var id int
    err := db.QueryRow(query, user.Name, user.Email, user.Age).Scan(&id)
    return id, err
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   30,
    }
    
    id, err := insertUser(db, user)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("User created with ID: %d\n", id)
}
```

### 4. Query Single Row

```go
func getUserByID(db *sql.DB, id int) (*User, error) {
    query := `
        SELECT id, name, email, age, created_at 
        FROM users 
        WHERE id = $1
    `
    
    var user User
    err := db.QueryRow(query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Age,
        &user.CreatedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    user, err := getUserByID(db, 1)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("User: %+v\n", user)
}
```

### 5. Query Multiple Rows

```go
func getAllUsers(db *sql.DB) ([]User, error) {
    query := `
        SELECT id, name, email, age, created_at 
        FROM users 
        ORDER BY id
    `
    
    rows, err := db.Query(query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var user User
        err := rows.Scan(
            &user.ID,
            &user.Name,
            &user.Email,
            &user.Age,
            &user.CreatedAt,
        )
        if err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    
    // Check for errors during iteration
    if err = rows.Err(); err != nil {
        return nil, err
    }
    
    return users, nil
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    users, err := getAllUsers(db)
    if err != nil {
        log.Fatal(err)
    }
    
    for _, user := range users {
        fmt.Printf("ID: %d, Name: %s, Email: %s\n", 
            user.ID, user.Name, user.Email)
    }
}
```

### 6. Update Data

```go
func updateUser(db *sql.DB, id int, name, email string) error {
    query := `
        UPDATE users 
        SET name = $1, email = $2 
        WHERE id = $3
    `
    
    result, err := db.Exec(query, name, email, id)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    err := updateUser(db, 1, "Alice Smith", "alice.smith@example.com")
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("User updated successfully")
}
```

### 7. Delete Data

```go
func deleteUser(db *sql.DB, id int) error {
    query := `DELETE FROM users WHERE id = $1`
    
    result, err := db.Exec(query, id)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    err := deleteUser(db, 1)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("User deleted successfully")
}
```

---

## Transactions

```go
func transferMoney(db *sql.DB, fromID, toID int, amount float64) error {
    // Begin transaction
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    
    // Defer rollback (no-op if commit succeeds)
    defer tx.Rollback()
    
    // Deduct from sender
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, fromID,
    )
    if err != nil {
        return err
    }
    
    // Add to receiver
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    )
    if err != nil {
        return err
    }
    
    // Commit transaction
    return tx.Commit()
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    defer db.Close()
    
    err := transferMoney(db, 1, 2, 100.50)
    if err != nil {
        log.Fatal("Transfer failed:", err)
    }
    
    fmt.Println("Transfer successful")
}
```

---

## Using pgx (Recommended)

### 1. Basic Connection with pgx

```go
package main

import (
    "context"
    "fmt"
    "log"
    
    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()
    
    connStr := "postgres://postgres:password@localhost:5432/testdb"
    
    conn, err := pgx.Connect(ctx, connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)
    
    // Test connection
    var greeting string
    err = conn.QueryRow(ctx, "SELECT 'Hello, PostgreSQL!'").Scan(&greeting)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println(greeting)
}
```

### 2. Connection Pool with pgxpool

```go
package main

import (
    "context"
    "fmt"
    "log"
    
    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    ctx := context.Background()
    
    // Connection pool
    pool, err := pgxpool.New(ctx, "postgres://postgres:password@localhost:5432/testdb")
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()
    
    // Test connection
    err = pool.Ping(ctx)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Connected to database with pool")
}
```

### 3. CRUD Operations with pgx

```go
type User struct {
    ID        int
    Name      string
    Email     string
    Age       int
    CreatedAt time.Time
}

// Insert
func createUser(ctx context.Context, pool *pgxpool.Pool, user User) (int, error) {
    query := `
        INSERT INTO users (name, email, age) 
        VALUES ($1, $2, $3) 
        RETURNING id
    `
    
    var id int
    err := pool.QueryRow(ctx, query, user.Name, user.Email, user.Age).Scan(&id)
    return id, err
}

// Select one
func getUser(ctx context.Context, pool *pgxpool.Pool, id int) (*User, error) {
    query := `
        SELECT id, name, email, age, created_at 
        FROM users 
        WHERE id = $1
    `
    
    var user User
    err := pool.QueryRow(ctx, query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Age,
        &user.CreatedAt,
    )
    
    if err == pgx.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    
    return &user, err
}

// Select many
func getAllUsers(ctx context.Context, pool *pgxpool.Pool) ([]User, error) {
    query := `SELECT id, name, email, age, created_at FROM users ORDER BY id`
    
    rows, err := pool.Query(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var user User
        err := rows.Scan(
            &user.ID,
            &user.Name,
            &user.Email,
            &user.Age,
            &user.CreatedAt,
        )
        if err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    
    return users, rows.Err()
}

// Update
func updateUser(ctx context.Context, pool *pgxpool.Pool, id int, name, email string) error {
    query := `UPDATE users SET name = $1, email = $2 WHERE id = $3`
    
    tag, err := pool.Exec(ctx, query, name, email, id)
    if err != nil {
        return err
    }
    
    if tag.RowsAffected() == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}

// Delete
func deleteUser(ctx context.Context, pool *pgxpool.Pool, id int) error {
    query := `DELETE FROM users WHERE id = $1`
    
    tag, err := pool.Exec(ctx, query, id)
    if err != nil {
        return err
    }
    
    if tag.RowsAffected() == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}

func main() {
    ctx := context.Background()
    
    pool, err := pgxpool.New(ctx, "postgres://postgres:password@localhost:5432/testdb")
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()
    
    // Create
    user := User{Name: "Alice", Email: "alice@example.com", Age: 30}
    id, err := createUser(ctx, pool, user)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created user with ID: %d\n", id)
    
    // Read
    u, err := getUser(ctx, pool, id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User: %+v\n", u)
    
    // Update
    err = updateUser(ctx, pool, id, "Alice Smith", "alice.smith@example.com")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("User updated")
    
    // List all
    users, err := getAllUsers(ctx, pool)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Total users: %d\n", len(users))
    
    // Delete
    err = deleteUser(ctx, pool, id)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("User deleted")
}
```

---

## Real-World REST API Example

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
    "time"
    
    "github.com/jackc/pgx/v5/pgxpool"
)

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Age       int       `json:"age"`
    CreatedAt time.Time `json:"created_at"`
}

type Server struct {
    pool *pgxpool.Pool
}

func NewServer(pool *pgxpool.Pool) *Server {
    return &Server{pool: pool}
}

func (s *Server) setupRoutes() {
    http.HandleFunc("/users", s.usersHandler)
    http.HandleFunc("/users/", s.userHandler)
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
    ctx := r.Context()
    
    query := `SELECT id, name, email, age, created_at FROM users ORDER BY id`
    
    rows, err := s.pool.Query(ctx, query)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var user User
        err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.Age, &user.CreatedAt)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        users = append(users, user)
    }
    
    respondJSON(w, http.StatusOK, users)
}

func (s *Server) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    query := `
        INSERT INTO users (name, email, age) 
        VALUES ($1, $2, $3) 
        RETURNING id, created_at
    `
    
    err := s.pool.QueryRow(ctx, query, user.Name, user.Email, user.Age).Scan(
        &user.ID, &user.CreatedAt,
    )
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    respondJSON(w, http.StatusCreated, user)
}

func (s *Server) handleGetUser(w http.ResponseWriter, r *http.Request, id int) {
    ctx := r.Context()
    
    query := `
        SELECT id, name, email, age, created_at 
        FROM users 
        WHERE id = $1
    `
    
    var user User
    err := s.pool.QueryRow(ctx, query, id).Scan(
        &user.ID, &user.Name, &user.Email, &user.Age, &user.CreatedAt,
    )
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    respondJSON(w, http.StatusOK, user)
}

func (s *Server) handleUpdateUser(w http.ResponseWriter, r *http.Request, id int) {
    ctx := r.Context()
    
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    query := `
        UPDATE users 
        SET name = $1, email = $2, age = $3 
        WHERE id = $4
    `
    
    tag, err := s.pool.Exec(ctx, query, user.Name, user.Email, user.Age, id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    if tag.RowsAffected() == 0 {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    user.ID = id
    respondJSON(w, http.StatusOK, user)
}

func (s *Server) handleDeleteUser(w http.ResponseWriter, r *http.Request, id int) {
    ctx := r.Context()
    
    query := `DELETE FROM users WHERE id = $1`
    
    tag, err := s.pool.Exec(ctx, query, id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    if tag.RowsAffected() == 0 {
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
    ctx := context.Background()
    
    connStr := "postgres://postgres:password@localhost:5432/testdb"
    
    pool, err := pgxpool.New(ctx, connStr)
    if err != nil {
        log.Fatal("Unable to connect to database:", err)
    }
    defer pool.Close()
    
    // Create table
    _, err = pool.Exec(ctx, `
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL,
            age INT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    `)
    if err != nil {
        log.Fatal("Unable to create table:", err)
    }
    
    server := NewServer(pool)
    server.setupRoutes()
    
    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Test:**
```bash
# Create user
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","age":30}'

# Get all users
curl http://localhost:8080/users

# Get user by ID
curl http://localhost:8080/users/1

# Update user
curl -X PUT http://localhost:8080/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Smith","email":"alice.smith@example.com","age":31}'

# Delete user
curl -X DELETE http://localhost:8080/users/1
```

---

## Summary

| Operation | database/sql | pgx |
|-----------|--------------|-----|
| Connect | `sql.Open()` | `pgx.Connect()` / `pgxpool.New()` |
| Query one | `db.QueryRow().Scan()` | `pool.QueryRow().Scan()` |
| Query many | `db.Query()` + `rows.Next()` | `pool.Query()` + `rows.Next()` |
| Execute | `db.Exec()` | `pool.Exec()` |
| Transaction | `db.Begin()` | `pool.Begin()` |
| Placeholder | `$1, $2, $3` | `$1, $2, $3` |

---

## Best Practices

### ✅ DO:

- Use connection pools (`pgxpool`)
- Use context for timeouts/cancellation
- Use prepared statements for repeated queries
- Use transactions for multi-step operations
- Handle `sql.ErrNoRows` / `pgx.ErrNoRows`
- Always `defer rows.Close()`
- Use `$1, $2` placeholders (prevent SQL injection)

### ❌ DON'T:

- Don't use string concatenation for queries
- Don't ignore errors
- Don't forget to close rows/connections
- Don't use `database/sql` without connection pooling

Go + PostgreSQL = Production-ready backend! 🚀🐘
