# Go `import` — Complete Guide

## Table of Contents
- [What `import` Is](#what-import-is)
- [What `import` Isn't](#what-import-isnt)
- [Syntax & Usage](#syntax--usage)
- [Import Paths](#import-paths)
- [Aliased & Special Imports](#aliased--special-imports)
- [Side-Effect Imports](#side-effect-imports)
- [Modules & go get](#modules--go-get)
- [Best Practices](#best-practices)

---

## What `import` Is

**`import`** brings **packages** into scope so you can use their **exported** (capitalized) names. The import path identifies the package; the **package name** (last segment of the path, usually) is the identifier you use in code.

- **Import path** → where the package lives (e.g. `"fmt"`, `"encoding/json"`, `"github.com/foo/bar"`).
- **Package name** → how you refer to it (e.g. `fmt.Println`, `json.Marshal`).
- **Scope** → imported names are available only in the current file, not across the whole module.

```go
import "fmt"

func main() {
    fmt.Println("hello")  // use package name "fmt"
}
```

---

## What `import` Isn't

| Not this | Reality |
|----------|--------|
| **#include-style text paste** | Imports are not copy-pasted. The compiler resolves packages by path; no textual inclusion. |
| **Importing “modules”** | You import **packages**. A **module** is the unit of versioning (`go.mod`); it contains one or more packages. |
| **Importing files** | You import **packages** (directories). All `.go` files in a directory share one package name. |
| **Global namespace** | Imports are **per-file**. Each file lists its own imports; there is no project-wide import list. |
| **Optional / dynamic** | Imports are **static**. All imports must resolve at compile time. No `import(path)` at runtime. |
| **Exporting symbols** | You don’t “export” via import. **Capitalization** controls export. Import only makes an already-exported package available. |

---

## Syntax & Usage

### Single import

```go
import "fmt"

func main() {
    fmt.Println("hello")
}
```

### Grouped imports (recommended)

```go
import (
    "encoding/json"
    "fmt"
    "net/http"
)

func main() {
    fmt.Println("hi")
    json.Marshal(nil)
    http.Get("https://example.com")
}
```

### Multiple single-line imports (valid but uncommon)

```go
import "fmt"
import "os"
import "strings"
```

Use the **grouped** form for clarity.

### Ordering

By convention, group imports into:

1. **Standard library** (e.g. `fmt`, `encoding/json`, `net/http`).
2. **Third-party** (e.g. `github.com/...`).

Blank lines between groups are optional but common. Tools like `goimports` sort and group them for you.

```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/lib/pq"
    "go.uber.org/zap"
)
```

---

## Import Paths

### Standard library

No host or module prefix. Use the path as documented in [pkg.go.dev/std](https://pkg.go.dev/std).

```go
import (
    "fmt"
    "encoding/json"
    "net/http"
    "context"
)
```

### Third-party (module path)

Usually **module path + package path** within the module. The **module path** is what appears in `go.mod` (e.g. `github.com/user/repo`). Subpackages add path segments.

```go
// Module: github.com/user/myapp
// Package: github.com/user/myapp/pkg/auth

import "github.com/user/myapp/pkg/auth"

auth.Validate(...)
```

### Same-module packages

Import other packages in **your** module using the **module path** from `go.mod`.

```go
// go.mod: module github.com/user/myapp

import "github.com/user/myapp/internal/config"
import "github.com/user/myapp/pkg/auth"
```

### Relative imports

**Not supported.** You cannot `import "./pkg"` or `import "../pkg"`. Use the full module path.

---

## Aliased & Special Imports

### Aliased import

Use a **different local name** than the default package name. Handy for long or duplicate names.

```go
import (
    f "fmt"
    m "math"
    httputil "net/http/httputil"
)

func main() {
    f.Println("hi")
    m.Sqrt(2)
    httputil.NewSingleHostReverseProxy(...)
}
```

### Blank identifier `_` (side-effect import)

Imports the package **only for side effects** (e.g. `init()`). You don’t use the package name in code. Common for **drivers** and **registrations**.

```go
import (
    "database/sql"
    _ "github.com/lib/pq"  // registers PostgreSQL driver with database/sql
)

func main() {
    db, err := sql.Open("postgres", "...")
    // ...
}
```

```go
import _ "image/png"  // registers PNG decoder with image package
```

**Without** `_ "github.com/lib/pq"`, `sql.Open("postgres", ...)` would not have a driver and would fail.

### Dot import `.` (avoid)

```go
import . "fmt"

func main() {
    Println("hello")  // no "fmt." prefix
}
```

**Avoid.** It pulls all exported names into the current file’s namespace, hurts readability, and can cause name clashes. The toolchain allows it, but style guides discourage it.

---

## Side-Effect Imports

Some packages are imported **only** for:

- **`init()`** — runs when the package is loaded (e.g. register a driver, configure a default).
- **Registration** with another package (e.g. `database/sql`, `image`, encoding packages).

Use **blank import** so the compiler doesn’t complain about an “unused” import:

```go
import _ "github.com/lib/pq"
import _ "github.com/jackc/pgx/v5/stdlib"
```

```go
import _ "go.uber.org/automaxprocs"  // sets GOMAXPROCS from env
```

You never reference the package name; the import is purely for side effects.

---

## Modules & go get

### Resolving imports

When you `import "github.com/foo/bar"`:

1. The **module** containing that package is found (via `go.mod` + `go.sum` or `go get`).
2. The **package** is compiled and linked. No copy-paste of source into your project.

### Adding a dependency

```bash
go get github.com/lib/pq
```

This adds the module to `go.mod` and updates `go.sum`. You then import packages from that module as usual.

### go.mod

Your module is defined by `go.mod`:

```text
module github.com/user/myapp

go 1.21

require (
    github.com/lib/pq v1.10.9
)
```

Imports use **module path + package path**; versions come from `go.mod`, not from the import line.

---

## Best Practices

1. **Use grouped imports** and let `goimports` (or your editor) order and format them.
2. **Prefer the default package name.** Use an alias only when necessary (e.g. conflict, very long name).
3. **Use `_` imports only for real side-effect packages** (drivers, registrations, `init`-only packages). Don’t use `_` to “silence” unused imports; remove those imports instead.
4. **Avoid dot imports** (`. "pkg"`).
5. **Use full import paths** — no relative imports. Stick to module-aware paths.
6. **Run `go mod tidy`** to drop unused dependencies and keep `go.mod` clean.

---

## Common Mistakes

### Unused import

```go
import "fmt"  // never use fmt
```

**Fix:** Remove the import (or use it). Don’t switch to `_ "fmt"` just to hide the error; `_` is for side-effect-only packages.

### Wrong path

```go
import "github.com/user/repo"  // repo root may have no package
```

Import **packages**, not necessarily the repo root. Often the path includes a subdir, e.g. `github.com/user/repo/pkg`.

### Relative import

```go
import "./mypkg"   // invalid
import "../other"  // invalid
```

**Fix:** Use the full module path, e.g. `github.com/user/myapp/mypkg`.

### Assuming import exports names

Import doesn’t control what is visible. **Exporting** is done via **capitalization** in the defining package. Import only makes that package (and its exported names) available in the current file.

---

## Summary

| Concept | Meaning |
|--------|---------|
| **import** | Bring a **package** into scope in the **current file** so you can use its exported names. |
| **Import path** | Identifies the package (stdlib path or module path + package path). |
| **Package name** | Identifier used in code (e.g. `fmt`, `json`). Default is usually the last path segment. |
| **Alias** | `alias "path"` — use `alias` instead of the default package name. |
| **Blank `_`** | `_ "path"` — import for **side effects** only (`init`, registration). |
| **Dot `.`** | `."path"` — bring exported names into current namespace; **avoid**. |

**Import** is how you depend on other packages; it’s static, per-file, and works together with **modules** and **export rules** to keep dependencies explicit and predictable.
