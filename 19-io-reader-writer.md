# Go `io.Reader` / `io.Writer` — Complete Guide

## Table of Contents
- [Overview](#overview)
- [io.Reader](#ioreader)
- [io.Writer](#iowriter)
- [Common io Helpers](#common-io-helpers)
- [Composing Readers and Writers](#composing-readers-and-writers)
- [Deep Examples](#deep-examples)
- [Best Practices](#best-practices)

---

## Overview

**`io.Reader`** and **`io.Writer`** are the core interfaces for **streaming I/O** in Go. Much of the standard library (files, HTTP, JSON, compression, etc.) works in terms of readers and writers.

| Interface | Method | Meaning |
|-----------|--------|--------|
| `io.Reader` | `Read(p []byte) (n int, err error)` | Fill `p` with up to `len(p)` bytes; return bytes read and error |
| `io.Writer` | `Write(p []byte) (n int, err error)` | Write `p`; return bytes written and error |

**`io.EOF`** is the normal “no more data” error for readers. **`n > 0`** can be returned **with** `io.EOF` on the last read.

---

## io.Reader

### Interface

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

- **`n`**: bytes read into `p` (`0 ≤ n ≤ len(p)`).
- **`err`**: `nil`, `io.EOF` (no more data), or another error.
- **Contract**: if `n > 0`, `err` may still be `io.EOF` (final read).

### Reading until EOF

```go
b, err := io.ReadAll(r)
if err != nil {
    return err
}
// use b
```

### Reading into a buffer (loop)

```go
buf := make([]byte, 32*1024)
for {
    n, err := r.Read(buf)
    if n > 0 {
        process(buf[:n])
    }
    if err != nil {
        if err == io.EOF {
            break
        }
        return err
    }
}
```

### Common readers

| Type | Package | Use |
|------|---------|-----|
| `*os.File` | `os` | File |
| `*strings.Reader` | `strings` | In-memory string |
| `*bytes.Reader` | `bytes` | In-memory `[]byte` |
| `*http.Response` body | `net/http` | HTTP response body |
| `*bufio.Reader` | `bufio` | Buffered read |

---

## io.Writer

### Interface

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

- **`n`**: bytes written from `p` (`0 ≤ n ≤ len(p)`).
- **`err`**: `nil` or an error. If `err != nil`, `n` may still be `> 0`.

### Common writers

| Type | Package | Use |
|------|---------|-----|
| `*os.File` | `os` | File |
| `*bytes.Buffer` | `bytes` | In-memory buffer |
| `*http.Request` body | `net/http` | HTTP request body |
| `*bufio.Writer` | `bufio` | Buffered write |
| `*json.Encoder` | `encoding/json` | JSON to writer |

---

## Common io Helpers

### io.ReadAll

```go
b, err := io.ReadAll(r)
// b is []byte of entire stream
```

### io.Copy

```go
n, err := io.Copy(dst io.Writer, src io.Reader)
// copies until EOF or error; n = bytes copied
```

### io.CopyBuffer

```go
buf := make([]byte, 32*1024)
n, err := io.CopyBuffer(dst, src, buf)
```

### io.WriteString

```go
n, err := io.WriteString(w, "hello")
```

### ioutil / os (historical)

- **`os.ReadFile`** / **`os.WriteFile`**: read/write whole file. Prefer these over `ioutil.ReadFile` / `ioutil.WriteFile`.

---

## Composing Readers and Writers

### io.MultiReader

```go
r := io.MultiReader(r1, r2, r3)
// read from r1 until EOF, then r2, then r3
```

### io.MultiWriter

```go
w := io.MultiWriter(w1, w2)
w.Write(p)  // writes to both w1 and w2
```

### io.LimitReader

```go
r := io.LimitReader(src, 1024)
// reads at most 1024 bytes from src, then EOF
```

### io.TeeReader

```go
r := io.TeeReader(src, w)
// each read from r also writes the same bytes to w
```

### io.Pipe

```go
pr, pw := io.Pipe()
// pr.Read blocks until pw.Write; useful to connect Reader-based and Writer-based APIs
go func() {
    json.NewEncoder(pw).Encode(data)
    pw.Close()
}()
io.Copy(dst, pr)
```

---

## Deep Examples

### Example 1: HTTP handler — read body, unmarshal JSON

```go
func handleCreate(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    var req CreateRequest
    if err := json.Unmarshal(body, &req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    // ...
}
```

Or with `json.Decoder`:

```go
var req CreateRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
}
```

### Example 2: Read file, write to HTTP response

```go
f, err := os.Open("data.json")
if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
}
defer f.Close()

w.Header().Set("Content-Type", "application/json")
io.Copy(w, f)
```

### Example 3: TeeReader — checksum while reading

```go
h := sha256.New()
r := io.TeeReader(src, h)
data, err := io.ReadAll(r)
if err != nil {
    return err
}
sum := h.Sum(nil)
// use data and sum
```

### Example 4: LimitReader — cap request size

```go
const maxBody = 1 << 20  // 1 MiB
r := io.LimitReader(req.Body, maxBody)
body, err := io.ReadAll(r)
if err != nil {
    return err
}
```

### Example 5: MultiWriter — log and respond

```go
var buf bytes.Buffer
mw := io.MultiWriter(w, &buf)  // w = http.ResponseWriter
json.NewEncoder(mw).Encode(resp)
log.Println("response:", buf.String())
```

### Example 6: strings.Reader / bytes.Reader

```go
s := strings.NewReader("hello, world")
b, _ := io.ReadAll(s)

r := bytes.NewReader([]byte{1, 2, 3})
io.Copy(dst, r)
```

---

## Best Practices

- **Prefer `io.Reader` / `io.Writer`** in function signatures over concrete types (`*os.File`, `*bytes.Buffer`, etc.) so you can test with strings, buffers, or mocks.
- Use **`io.ReadAll`** for small streams; use **`io.Copy`** or a **buffered loop** for large ones to avoid big allocations.
- **Always handle `io.EOF`** when reading in a loop; it’s normal, not a failure.
- Use **`io.LimitReader`** (or similar) to cap input size and avoid unbounded reads.
- Use **`defer ... Close()`** for readers/writers that own resources (files, HTTP bodies, etc.).
- **`io.Pipe`** is useful to connect reader-based and writer-based APIs (e.g. encode JSON into a pipe, read from the other end).

---

## Summary

| Concept | Purpose |
|--------|---------|
| `io.Reader` | Read bytes from a stream |
| `io.Writer` | Write bytes to a stream |
| `io.ReadAll` | Slurp entire reader |
| `io.Copy` | Copy reader → writer |
| `io.MultiReader` | Chain readers |
| `io.MultiWriter` | Fan-out writes |
| `io.LimitReader` | Cap read size |
| `io.TeeReader` | Read and mirror to writer |
| `io.Pipe` | Connect reader and writer |

Using these interfaces and helpers keeps I/O code consistent, testable, and reusable across files, HTTP, JSON, and other packages.
