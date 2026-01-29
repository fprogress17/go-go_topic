# Go Topics — Table of Contents

Reference guides for Go: concurrency, stdlib, databases, HTTP, and more.  
**Suggested / not-yet-created** topics are listed at the end; ask to add those next when you're ready.

---

## Created topics

| # | File | Description |
|---|------|-------------|
| 00 | **00-TOC.md** | This table of contents |
| 01 | [01-goroutines-and-channels.md](01-goroutines-and-channels.md) | Goroutines, channels, `go func()`, `<-done`, patterns |
| 02 | [02-select-and-context.md](02-select-and-context.md) | `select`, `context` (cancellation, deadlines, values) |
| 03 | [03-variable-declaration.md](03-variable-declaration.md) | `var` vs `:=`, syntax, zero values |
| 04 | [04-short-declaration-reassignment.md](04-short-declaration-reassignment.md) | `:=` vs `=`, re-assignment, scope |
| 05 | [05-generics.md](05-generics.md) | Type parameters, constraints, generic structs |
| 06 | [06-interfaces-and-structs.md](06-interfaces-and-structs.md) | Structs, interfaces, composition |
| 07 | [07-nil.md](07-nil.md) | `nil` for pointers, slices, maps, channels, interfaces |
| 08 | [08-capitalization-visibility.md](08-capitalization-visibility.md) | Exported vs unexported (visibility) |
| 09 | [09-json-and-metadata.md](09-json-and-metadata.md) | JSON marshal/unmarshal, struct tags |
| 10 | [10-http.md](10-http.md) | HTTP server/client, middleware, REST |
| 11 | [11-postgresql.md](11-postgresql.md) | `database/sql`, pgx, CRUD, transactions |
| 12 | [12-kafka.md](12-kafka.md) | kafka-go, confluent-kafka-go, producers/consumers |
| 13 | [13-grpc.md](13-grpc.md) | Protocol Buffers, gRPC server/client, streaming |
| 14 | [14-context.md](14-context.md) | Context deep dive: timeout, cancel, value, propagation |
| 15 | [15-make.md](15-make.md) | `make` for slices, maps, channels |
| 16 | [16-sync.md](16-sync.md) | `WaitGroup`, `Mutex`, `RWMutex`, `Once`, `sync.Map`, `sync.Pool` |
| 17 | [17-defer-panic-recover.md](17-defer-panic-recover.md) | `defer`, `panic`, `recover` |
| 18 | [18-errors.md](18-errors.md) | `errors` package: `Is`, `As`, wrapping, custom errors |
| 19 | [19-io-reader-writer.md](19-io-reader-writer.md) | `io.Reader`, `io.Writer`, composition, helpers |
| 20 | [20-import.md](20-import.md) | `import`: syntax, paths, alias, blank, side effects, what it is / isn’t |
| 21 | [21-empty-interface-and-struct.md](21-empty-interface-and-struct.md) | `any` / `interface{}`, `struct{}`: usage, sets, signals, JSON |

---

## Suggested topics (not yet created)

Add these as new `XX-topic-name.md` files when you want to expand the set.  
You can say e.g. *“add the next item from the TOC”* or *“add Testing”*.

| # | Topic | Description |
|---|-------|-------------|
| 1 | **Testing** | `testing` package, table-driven tests, benchmarks, `httptest`, mocks |
| 2 | **Pointers** | Value vs reference, when to use, pointer receivers, escaping |
| 3 | **go mod** | Modules, versions, `replace`, `exclude`, vendoring |
| 4 | **Functional options** | Optional config, builder-style APIs |
| 5 | **Reflection (reflect)** | `reflect` basics, when used, limitations |
| 6 | **Profiling (pprof)** | CPU, heap, goroutine profiles, capturing and reading |

---

## Quick links

- Concurrency: [01](01-goroutines-and-channels.md) · [02](02-select-and-context.md) · [14](14-context.md) · [16](16-sync.md)
- Types & basics: [03](03-variable-declaration.md) · [04](04-short-declaration-reassignment.md) · [05](05-generics.md) · [06](06-interfaces-and-structs.md) · [07](07-nil.md) · [08](08-capitalization-visibility.md) · [15](15-make.md) · [20](20-import.md) · [21](21-empty-interface-and-struct.md)
- Stdlib & I/O: [09](09-json-and-metadata.md) · [10](10-http.md) · [17](17-defer-panic-recover.md) · [18](18-errors.md) · [19](19-io-reader-writer.md)
- Data & services: [11](11-postgresql.md) · [12](12-kafka.md) · [13](13-grpc.md)
