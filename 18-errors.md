# Go `errors` Package — Complete Guide

## Table of Contents
- [Overview](#overview)
- [Creating Errors](#creating-errors)
- [Wrapping Errors](#wrapping-errors)
- [errors.Is](#errorsis)
- [errors.As](#errorsas)
- [errors.Unwrap](#errorsunwrap)
- [Custom Error Types](#custom-error-types)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)

---

## Overview

The `errors` package (and `fmt`) provide **error creation**, **wrapping**, and **inspection**. Use it for:

- **Wrapping** errors with context (`fmt.Errorf("...: %w", err)`).
- **Checking** sentinel or type with `errors.Is` and `errors.As`.
- **Custom** error types for structured, actionable errors.

---

## Creating Errors

### `errors.New`

```go
var ErrNotFound = errors.New("resource not found")

func fetch(id int) error {
    if id < 0 {
        return errors.New("invalid id")
    }
    // ...
    return nil
}
```

### `fmt.Errorf`

```go
return fmt.Errorf("user %d not found", id)
```

Use **`%w`** only when wrapping; see below.

### Sentinel errors

```go
var (
    ErrNotFound   = errors.New("not found")
    ErrPermission = errors.New("permission denied")
)

if errors.Is(err, ErrNotFound) {
    // handle
}
```

---

## Wrapping Errors

Use **`%w`** inside `fmt.Errorf` to wrap an underlying error. The wrapped error is available to `errors.Is` and `errors.As`.

```go
if err != nil {
    return fmt.Errorf("fetch user: %w", err)
}
```

### Chain of wraps

```go
// in db layer
return fmt.Errorf("query user id=%d: %w", id, sql.ErrNoRows)

// in service layer
return fmt.Errorf("get user: %w", err)

// in handler
return fmt.Errorf("api getUser: %w", err)
```

---

## errors.Is

**`errors.Is(err, target)`** reports whether `err` or any error in its chain **equals** `target` (or implements `Is(error) bool` and that returns true).

### Basic usage

```go
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

if errors.Is(err, os.ErrNotExist) {
    return nil, ErrNotFound
}

if errors.Is(err, context.Canceled) {
    return nil, err
}
```

### Why not `==`?

Wrapped errors are not `==` to the inner error. `errors.Is` unwraps and compares.

```go
err := fmt.Errorf("wrap: %w", os.ErrNotExist)
err == os.ErrNotExist       // false
errors.Is(err, os.ErrNotExist)  // true
```

---

## errors.As

**`errors.As(err, target)`** finds the **first** error in `err`’s chain that can be **assigned to** `target` (a pointer to interface or type). If found, it assigns and returns `true`.

### Basic usage

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path, pathErr.Op)
}

var e *MyError
if errors.As(err, &e) {
    fmt.Println(e.Code, e.Message)
}
```

### target must be pointer

```go
var pe *os.PathError
errors.As(err, &pe)   // ok
errors.As(err, pe)    // wrong: need pointer
```

---

## errors.Unwrap

**`errors.Unwrap(err)`** returns the underlying error if `err` implements `Unwrap() error`, else `nil`.

```go
func Unwrap(err error) error
```

Most code should use **`errors.Is`** and **`errors.As`** instead of calling `Unwrap` directly.

---

## Custom Error Types

### Struct with `Error()` and `Unwrap()`

```go
type ValidationError struct {
    Field   string
    Message string
    Err     error
}

func (e *ValidationError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %s (%v)", e.Field, e.Message, e.Err)
    }
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

func (e *ValidationError) Unwrap() error {
    return e.Err
}
```

### Optional `Is` for sentinels

```go
type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

var ErrAuth = &MyError{Code: 401, Message: "unauthorized"}

func (e *MyError) Is(target error) bool {
    t, ok := target.(*MyError)
    if !ok {
        return false
    }
    return e.Code == t.Code
}
```

Then `errors.Is(err, ErrAuth)` works for any `*MyError` with the same `Code`.

---

## Deep Examples

### Example 1: Service layer wrapping

```go
func (s *Service) GetUser(ctx context.Context, id int) (*User, error) {
    u, err := s.repo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("get user id=%d: %w", id, err)
    }
    return u, nil
}
```

### Example 2: Handler checking error type

```go
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    u, err := h.svc.GetUser(r.Context(), getUserID(r))
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            http.Error(w, "user not found", http.StatusNotFound)
            return
        }
        var ve *ValidationError
        if errors.As(err, &ve) {
            http.Error(w, ve.Message, http.StatusBadRequest)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(u)
}
```

### Example 3: Retry on temporary errors

```go
type temporary interface {
    Temporary() bool
}

func isTemporary(err error) bool {
    var t temporary
    return errors.As(err, &t) && t.Temporary()
}

func doWithRetry(ctx context.Context, fn func() error, n int) error {
    var err error
    for i := 0; i < n; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        if ctx.Err() != nil {
            return ctx.Err()
        }
        if !isTemporary(err) {
            return err
        }
        time.Sleep(time.Second * time.Duration(i+1))
    }
    return err
}
```

### Example 4: Logging full chain

```go
func logErr(err error) {
    for err != nil {
        log.Printf("  %v", err)
        err = errors.Unwrap(err)
    }
}
```

---

## Best Practices

- **Wrap** with **`%w`** when adding context: `fmt.Errorf("doing X: %w", err)`.
- **Check** sentinels with **`errors.Is`**; check types with **`errors.As`**. Prefer these over `==` or type assertions on `err` alone.
- **Define sentinels** as `var ErrX = errors.New("...")` (or custom types) and use `errors.Is(err, ErrX)`.
- **Custom types**: implement `Error()` and optionally `Unwrap()`; implement `Is()` only when you need custom `errors.Is` behavior.
- **Don’t wrap** when you’re about to lose the underlying error (e.g. logging and returning something else). Either wrap and return, or handle fully.
- **Use `errors.Join`** (Go 1.20+) when returning multiple errors: `return errors.Join(err1, err2)`.

---

## Summary

| Function | Purpose |
|----------|---------|
| `errors.New` | Create a simple error |
| `fmt.Errorf("...%w", err)` | Wrap error with context |
| `errors.Is(err, target)` | Check sentinel / equality in chain |
| `errors.As(err, &target)` | Extract first matching type in chain |
| `errors.Unwrap(err)` | Get next error in chain |

Wrapping with `%w` and using `Is`/`As` gives you clear, inspectable error chains and better handling in real-world code.
