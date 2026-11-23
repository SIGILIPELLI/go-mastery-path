---
title: "Go Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Go cheat sheet: 36 quick-reference tables from every level of the Go Mastery Path, each linked to its full lesson."
---

# Go Cheat Sheet

Quick-reference tables from **36 lessons** of the free Go Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables, Types &amp; Operators](level-1/02-variables-types-operators.md)

| Concept | Syntax |
|---------|--------|
| Declare with inference | `x := 5` |
| Declare with explicit type | `var x int = 5` |
| Zero value | `var x int` → `0` |
| Constant | `const Pi = 3.14` |
| Enum-like constants | `const ( A = iota; B; C )` |
| Convert types | `float64(x)`, `int(y)` |
| Multiple assignment | `a, b := 1, 2` |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Example |
|-----------|---------|
| If | `if x > 0 { ... }` |
| If/else if/else | `if ... else if ... else { ... }` |
| If with initializer | `if v, err := f(); err == nil { ... }` |
| Classic for | `for i := 0; i < n; i++ { ... }` |
| While-style | `for cond { ... }` |
| Infinite | `for { ... break }` |
| Range (index, value) | `for i, v := range slice { ... }` |
| Range (value only) | `for _, v := range slice { ... }` |
| Switch | `switch x { case 1: ...; default: ... }` |
| Switch, no expression | `switch { case cond: ... }` |

### [04 · Functions &amp; Multiple Returns](level-1/04-functions-multiple-returns.md)

| Feature | Syntax |
|---------|--------|
| Basic function | `func add(a, b int) int { return a + b }` |
| Multiple returns | `func f() (int, error) { ... }` |
| Named returns | `func f() (x int) { x = 1; return }` |
| Variadic | `func f(nums ...int) { ... }` |
| Spread a slice | `f(values...)` |
| Discard a return | `_, err := f()` |
| Function value | `var f func(int) int = double` |
| Closure | `func() func() int { ... }` |
| Deferred cleanup | `defer file.Close()` |

### [05 · Arrays, Slices &amp; Maps](level-1/05-arrays-slices-maps.md)

| Operation | Syntax |
|-----------|--------|
| Array (fixed size) | `var a [3]int` |
| Slice literal | `s := []int{1, 2, 3}` |
| Slice with make | `s := make([]int, len, cap)` |
| Append | `s = append(s, 4, 5)` |
| Slice a slice | `s[1:3]` |
| Copy independently | `copy(dst, src)` |
| Map literal | `m := map[string]int{"a": 1}` |
| Map with make | `m := make(map[string]int)` |
| Read + existence check | `v, ok := m[key]` |
| Delete a key | `delete(m, key)` |
| Length (slice or map) | `len(s)` |

### [06 · Structs](level-1/06-structs.md)

| Feature | Syntax |
|---------|--------|
| Define | `type Person struct { Name string; Age int }` |
| Create (named fields) | `Person{Name: "A", Age: 1}` |
| Create (positional) | `Person{"A", 1}` |
| Zero value | `var p Person` |
| Access/set field | `p.Name`, `p.Name = "X"` |
| Nested access | `e.Address.City` |
| Embedding | `type Dog struct { Animal; Breed string }` |
| Compare (comparable fields) | `a == b` |
| Struct tag | `` `json:"host"` `` |

### [07 · Pointers](level-1/07-pointers.md)

| Concept | Syntax |
|---------|--------|
| Address-of | `p := &x` |
| Dereference | `*p` |
| Pointer type | `var p *int` |
| Pointer to struct literal | `&Point{X: 1, Y: 2}` |
| Allocate zeroed | `p := new(Point)` |
| Auto-deref field access | `p.X` (shorthand for `(*p).X`) |
| Nil check | `if p == nil { ... }` |

### [08 · Error Handling](level-1/08-error-handling.md)

| Task | Syntax |
|------|--------|
| Return a plain error | `errors.New("message")` |
| Return a formatted error | `fmt.Errorf("bad value: %d", n)` |
| Check for failure | `if err != nil { ... }` |
| Custom error type | `type X struct{}; func (X) Error() string { ... }` |
| Stop execution | `panic("message")` |
| Catch a panic | `defer func() { recover() }()` |

### [09 · Packages &amp; Modules](level-1/09-packages-modules.md)

| Task | Command / syntax |
|------|-------------------|
| Create a module | `go mod init <module-name>` |
| Add a dependency | `go get <import-path>@<version>` |
| Clean up dependencies | `go mod tidy` |
| Import a package | `import "todoapp/task"` |
| Import stdlib package | `import "fmt"` |
| Import with alias | `import f "fmt"` |
| Exported identifier | Starts with an uppercase letter |
| Unexported identifier | Starts with a lowercase letter |

## Level 2 · Intermediate

### [01 · Interfaces](level-2/01-interfaces.md)

| Task | Syntax |
|------|--------|
| Declare an interface | `type S interface { Area() float64 }` |
| Satisfy an interface | just define the methods — nothing to declare |
| Empty interface | `any` (or `interface{}`) |
| Safe type assertion | `v, ok := x.(Concrete)` |
| Type switch | `switch v := x.(type) { case int: ... }` |
| Embed interfaces | `type RC interface { Reader; Closer }` |
| Compile-time check | `var _ Iface = (*Impl)(nil)` |
| Print concrete type | `fmt.Printf("%T", x)` |

### [02 · Goroutines &amp; Channels Basics](level-2/02-goroutines-channels.md)

| Task | Syntax |
|------|--------|
| Start a goroutine | `go doWork(arg)` |
| Unbuffered channel | `ch := make(chan int)` |
| Buffered channel | `ch := make(chan int, 10)` |
| Send / receive | `ch <- v` / `v := <-ch` |
| Receive with closed check | `v, ok := <-ch` |
| Close a channel | `close(ch)` (sender only) |
| Loop until closed | `for v := range ch { ... }` |
| Send-only / receive-only | `chan<- int` / `<-chan int` |
| Wait for N goroutines | `wg.Add(1)` / `defer wg.Done()` / `wg.Wait()` |
| Multiplex channels | `select { case v := <-a: ... }` |
| Timeout | `case <-time.After(2 * time.Second):` |
| Mutual exclusion | `mu.Lock(); defer mu.Unlock()` |
| Detect data races | `go run -race .` |

### [03 · Methods &amp; Receivers](level-2/03-methods-receivers.md)

| Task | Syntax |
|------|--------|
| Value-receiver method | `func (t T) M() { ... }` |
| Pointer-receiver method | `func (t *T) M() { ... }` |
| Call on a value | `v.M()` — auto `&v` if needed and addressable |
| Call on a pointer | `p.M()` — auto `*p` if needed |
| Method on a named non-struct | `type ID int; func (i ID) String() string` |
| Satisfy an interface with pointer methods | pass `&T{}`, not `T{}` |
| Method value | `f := v.M; f()` |
| Method expression | `f := T.M; f(v)` |
| Nil-safe method | `if t == nil { return zero }` |

### [04 · Custom Errors &amp; Error Wrapping](level-2/04-custom-errors-wrapping.md)

| Task | Syntax |
|------|--------|
| Sentinel error | `var ErrX = errors.New("x")` |
| Formatted error | `fmt.Errorf("bad id %d", id)` |
| Wrap with context | `fmt.Errorf("loading: %w", err)` |
| Compare to a sentinel | `errors.Is(err, ErrX)` |
| Extract a typed error | `var e *MyErr; errors.As(err, &e)` |
| Make a type unwrappable | `func (e *MyErr) Unwrap() error` |
| Peel one layer | `errors.Unwrap(err)` |
| Combine errors | `errors.Join(err1, err2)` |
| Custom error type | `func (e *MyErr) Error() string` |

### [05 · JSON Encoding/Decoding](level-2/05-json-encoding-decoding.md)

| Task | Syntax |
|------|--------|
| Struct → JSON | `data, err := json.Marshal(v)` |
| Pretty JSON | `json.MarshalIndent(v, "", "  ")` |
| JSON → struct | `json.Unmarshal(data, &v)` |
| Read from a stream | `json.NewDecoder(r).Decode(&v)` |
| Write to a stream | `json.NewEncoder(w).Encode(v)` |
| Reject unknown keys | `dec.DisallowUnknownFields()` |
| Rename a key | `` `json:"user_id"` `` |
| Omit zero values | `` `json:"bio,omitempty"` `` |
| Skip a field | `` `json:"-"` `` |
| Unknown shape | `var m map[string]any` |
| Validate JSON text | `json.Valid(data)` |

### [06 · Testing with the testing Package](level-2/06-testing-package.md)

| Task | Syntax |
|------|--------|
| Test function | `func TestX(t *testing.T)` |
| Report and continue | `t.Errorf("got %v; want %v", got, want)` |
| Report and stop | `t.Fatalf(...)` |
| Named subtest | `t.Run(name, func(t *testing.T){ ... })` |
| Skip | `t.Skip("needs network")` |
| Temp dir | `dir := t.TempDir()` |
| Teardown | `t.Cleanup(func(){ ... })` |
| Mark a helper | `t.Helper()` |
| Run in parallel | `t.Parallel()` |
| Compare slices | `slices.Equal(got, want)` |
| Compare anything | `reflect.DeepEqual(got, want)` |
| Run all tests | `go test ./...` |
| Coverage | `go test -cover ./...` |
| Race detection | `go test -race ./...` |

### [07 · File I/O](level-2/07-file-io.md)

| Task | Syntax |
|------|--------|
| Read a whole file | `data, err := os.ReadFile(path)` |
| Write a whole file | `os.WriteFile(path, data, 0644)` |
| Open for reading | `f, err := os.Open(path)` |
| Create / truncate | `f, err := os.Create(path)` |
| Open for appending | `os.OpenFile(p, os.O_APPEND\|os.O_CREATE\|os.O_WRONLY, 0644)` |
| Close reliably | `defer f.Close()` |
| Read line by line | `sc := bufio.NewScanner(f); for sc.Scan() { sc.Text() }` |
| Check scanner errors | `if err := sc.Err(); err != nil` |
| Buffered writing | `w := bufio.NewWriter(f)` … `w.Flush()` |
| Formatted write | `fmt.Fprintf(w, "x = %d\n", x)` |
| File metadata | `info, err := os.Stat(path)` |
| Missing-file check | `errors.Is(err, os.ErrNotExist)` |
| Delete / rename | `os.Remove(p)` / `os.Rename(old, new)` |
| Make directories | `os.MkdirAll(dir, 0755)` |
| List a directory | `os.ReadDir(dir)` |
| Walk a tree | `filepath.WalkDir(root, fn)` |
| Join paths | `filepath.Join("a", "b")` |
| Stream copy | `io.Copy(dst, src)` |

### [08 · Working with net/http](level-2/08-net-http-client.md)

| Task | Syntax |
|------|--------|
| Quick GET | `resp, err := http.Get(url)` |
| Always close the body | `defer resp.Body.Close()` |
| Read the whole body | `io.ReadAll(resp.Body)` |
| Bounded read | `io.ReadAll(io.LimitReader(resp.Body, 1<<20))` |
| Decode JSON | `json.NewDecoder(resp.Body).Decode(&v)` |
| Client with a timeout | `&http.Client{Timeout: 10 * time.Second}` |
| Build a request | `http.NewRequest(method, url, body)` |
| With cancellation | `http.NewRequestWithContext(ctx, ...)` |
| Send it | `resp, err := client.Do(req)` |
| Set a header | `req.Header.Set("Authorization", "Bearer "+t)` |
| JSON POST body | `bytes.NewReader(jsonBytes)` |
| Query parameters | `u.RawQuery = url.Values{...}.Encode()` |
| Status constants | `http.StatusOK`, `http.StatusNotFound` |
| Stream to disk | `io.Copy(file, resp.Body)` |

### [09 · Dependency Management](level-2/09-dependency-management.md)

| Task | Command |
|------|---------|
| Start a module | `go mod init github.com/you/proj` |
| Add / update a dependency | `go get github.com/x/y@v1.2.3` |
| Sync `go.mod` with imports | `go mod tidy` |
| Upgrade everything | `go get -u ./...` |
| Patch-only upgrades | `go get -u=patch ./...` |
| Remove a dependency | `go get github.com/x/y@none` |
| List the build's modules | `go list -m all` |
| Show available updates | `go list -m -u all` |
| Explain a dependency | `go mod why github.com/x/y` |
| Verify checksums | `go mod verify` |
| Vendor dependencies | `go mod vendor` |
| Local override | `replace github.com/x/y => ../y` |
| Multi-module workspace | `go work init ./a ./b` |
| Download without building | `go mod download` |
| Scan for vulnerabilities | `govulncheck ./...` |

## Level 3 · Advanced

### [01 · Concurrency Patterns](level-3/01-concurrency-patterns.md)

| Pattern | Shape |
|---|---|
| Worker pool | N goroutines `range` over a shared `jobs` channel |
| Fan-out | Multiple goroutines reading the *same* input channel |
| Fan-in | `merge()` combines several channels into one with a `WaitGroup` |
| Pipeline | `stage2(stage1(source()))` — each stage its own goroutine |
| Exactly-once init | `var once sync.Once; once.Do(fn)` |
| Rate limit | `limiter := time.Tick(d); <-limiter` before each unit of work |
| Safe channel close | Close only from the sender, only after all sends are done |
| Cancellable send | `select { case ch <- v: case <-ctx.Done(): return }` |

### [02 · Building REST APIs](level-3/02-building-rest-apis.md)

| Task | API |
|---|---|
| Serve JSON | `w.Header().Set("Content-Type","application/json")` then `json.NewEncoder(w).Encode(v)` |
| Parse JSON body | `json.NewDecoder(r.Body).Decode(&v)` |
| Path parameter (Go 1.22+) | `mux.HandleFunc("GET /x/{id}", ...)`, `r.PathValue("id")` |
| Set status code | `w.WriteHeader(http.StatusCreated)` — before any `Write` |
| Method-not-allowed | `http.Error(w, "...", http.StatusMethodNotAllowed)` |
| Wrap handlers | `func(next http.Handler) http.Handler` |
| Concurrency-safe state | Guard shared maps/slices with `sync.Mutex` |

### [03 · Databases &amp; SQL](level-3/03-databases-sql.md)

| Task | API |
|---|---|
| Register a driver | `import _ "modernc.org/sqlite"` (blank import for its `init()`) |
| Open a pooled handle | `db, err := sql.Open(driverName, dsn)` |
| Run DDL/DML, no rows back | `db.Exec(query, args...)` |
| Reusable prepared statement | `db.Prepare(query)` then `stmt.Exec(args...)` |
| Query multiple rows | `rows, err := db.Query(query, args...)`; `defer rows.Close()` |
| Query exactly one row | `db.QueryRow(query, args...).Scan(&dest...)` |
| No matching row | compare scan error to `sql.ErrNoRows` |
| Multi-statement atomicity | `tx, _ := db.Begin()`; `tx.Exec(...)`; `tx.Commit()` / `tx.Rollback()` |

### [04 · The `context` Package](level-3/04-context-package.md)

| Need | API |
|---|---|
| Root context | `context.Background()` (real programs) / `context.TODO()` (not yet wired up) |
| Cancel manually | `ctx, cancel := context.WithCancel(parent)` |
| Cancel after a duration | `ctx, cancel := context.WithTimeout(parent, d)` |
| Cancel at a wall-clock time | `ctx, cancel := context.WithDeadline(parent, t)` |
| Wait for cancellation | `select { case <-ctx.Done(): ... }` |
| Why it ended | `ctx.Err()` → `context.Canceled` or `context.DeadlineExceeded` |
| Attach request-scoped data | `context.WithValue(ctx, privateKeyType, v)` |
| Read it back | `v, ok := ctx.Value(privateKeyType).(T)` |

### [05 · Design Patterns, the Go Way](level-3/05-design-patterns.md)

| GoF-flavored need | Go idiom |
|---|---|
| Configurable constructor | Functional options: `func(*T) ` closures, `opts ...Option` |
| Interchangeable algorithms | Small interface + multiple implementing types |
| Add behavior without subclassing | Wrap the interface in another type holding `next` |
| "Abstract base class" | An interface with no default implementation |
| Singleton-ish shared state | Package-level var + `sync.Once` (see [Module 1](level-3/01-concurrency-patterns.md)) |
| Builder-style construction | Return `*T` from each step, or use functional options |

### [06 · Testing, Advanced](level-3/06-testing-advanced.md)

| Task | Command / API |
|---|---|
| Run all tests, verbose | `go test -v ./...` |
| Run one named test/subtest | `go test -run TestAdd/2\+3` |
| Table test with subtests | `for _, c := range cases { t.Run(name, func(t *testing.T){...}) }` |
| Fail, keep running rest of table | `t.Errorf(...)` |
| Fail, stop this test now | `t.Fatalf(...)` |
| Fuzz a function | `func FuzzX(f *testing.F)`, seed with `f.Add(...)`, `f.Fuzz(func(t *testing.T, ...){...})` |
| Run fuzzing | `go test -run=^$ -fuzz=FuzzX -fuzztime=30s` |
| Benchmark | `func BenchmarkX(b *testing.B) { for i:=0;i<b.N;i++ {...} }` |
| Run benchmarks only | `go test -bench=. -run=^$` |

### [07 · Profiling &amp; Benchmarking](level-3/07-profiling-benchmarking.md)

| Task | Command |
|---|---|
| Benchmark with allocation stats | `go test -bench=. -run=^$ -benchmem` |
| Write a CPU profile | `go test -bench=X -run=^$ -cpuprofile=cpu.prof` |
| Write a memory profile | `go test -bench=X -run=^$ -memprofile=mem.prof` |
| Inspect a profile, top functions | `go tool pprof -top -nodecount=N cpu.prof` |
| Interactive profile browser | `go tool pprof cpu.prof` then `top`, `list <func>`, `web` |
| Live profiling for a running server | `import _ "net/http/pprof"`, hit `/debug/pprof/profile` |
| Statistically compare benchmark runs | `benchstat old.txt new.txt` |

### [08 · Building CLIs](level-3/08-building-clis.md)

| Task | API |
|---|---|
| Boolean flag | `f := flag.Bool("name", false, "usage")` |
| String flag | `f := flag.String("name", "", "usage")` |
| Parse `os.Args` | `flag.Parse()` (call before reading any flag value) |
| Non-flag positional args | `flag.Args()`, `flag.Arg(i)`, `flag.NArg()` |
| Per-subcommand flags | `fs := flag.NewFlagSet("name", flag.ExitOnError)` |
| Read stdin line by line | `bufio.NewScanner(os.Stdin)`, `for scanner.Scan() { scanner.Text() }` |
| Report an error | `fmt.Fprintln(os.Stderr, "...", err)` |
| Signal failure to the shell | `os.Exit(1)` (or `2` for usage errors) |

### [09 · Interfaces &amp; Generics](level-3/09-interfaces-generics.md)

| Need | Syntax |
|---|---|
| Generic function | `func F[T any](v T) T` |
| Multiple type params | `func Map[T, U any](in []T, f func(T) U) []U` |
| Numeric-only constraint | `type Number interface { ~int \| ~int64 \| ~float64 }` |
| Generic struct | `type Stack[T any] struct { items []T }` |
| Method on generic struct | `func (s *Stack[T]) Push(v T)` |
| Zero value of a type param | `var zero T` |
| Explicit type argument | `Sum[int](level-3/vals)` (when inference can't determine it) |
| Interface composition | Embed interfaces inside another `interface { A; B }` |

### [10 · Project: REST API with SQLite](level-3/10-project-rest-api-sqlite.md)

| Piece | Where |
|---|---|
| Interface for testability | `Store` in `store.go` (Module 5's strategy pattern) |
| Context-aware DB calls | `db.ExecContext`/`QueryRowContext` (Module 3 + 4) |
| Method+wildcard routing | `mux.HandleFunc("PATCH /tasks/{id}", ...)` (Module 2) |
| Request logging | `withLogging` middleware wraps the whole mux |
| Fast tests, no real DB | `fakeStore` + `httptest.NewRequest`/`NewRecorder` |
| Sentinel not-found error | `ErrNotFound`, checked via `errors.Is` |

## Level 4 · Master

### [01 · Advanced Concurrency](level-4/01-advanced-concurrency.md)

| Need | Tool |
|---|---|
| Run N goroutines, fail-fast, propagate first error | `errgroup.WithContext` + `g.Go(func() error {...})` |
| Bound concurrent goroutines in a group | `g.SetLimit(n)` |
| Concurrent-safe counter | `atomic.Int64` (typed) or `atomic.AddInt64(&x, ...)` |
| Concurrent-safe map, disjoint keys | `sync.Map` — `Store`, `Load`, `Range`, `Delete` |
| Concurrent-safe map, general case | `map[K]V` + `sync.RWMutex` |
| Wait for goroutines, no error propagation needed | `sync.WaitGroup` ([Level 3](level-3/01-concurrency-patterns.md)) |

### [02 · Microservices Patterns](level-4/02-microservices-patterns.md)

| Concern | Pattern |
|---|---|
| Never hang forever on a dependency | `&http.Client{Timeout: d}` |
| Fake a downstream service for tests | `httptest.NewServer(mux)` |
| Stop calling a failing dependency | Circuit breaker: count failures, `open` after threshold |
| Recover after cooldown | Half-open state — one trial call after a wait, not shown above |
| Bound total time across N dependency calls | `errgroup.WithContext` ([Module 1](level-4/01-advanced-concurrency.md)) |
| Reuse connections | One shared `*http.Client` per destination, not one per call |

### [03 · gRPC Basics](level-4/03-grpc-basics.md)

| Task | Command / API |
|---|---|
| Install tooling | `brew install protobuf`; `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`; `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest` |
| Generate Go code | `protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative x.proto` |
| Start a server | `grpc.NewServer()`; `pb.RegisterXServer(s, impl)`; `s.Serve(lis)` |
| Connect a client | `grpc.NewClient(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))` |
| Bound an RPC call | `context.WithTimeout(ctx, d)`, pass `ctx` as the call's first argument |
| Nil-safe field access | Generated getters: `msg.GetField()`, never `msg.Field` on a possibly-nil message |
| Typed error with status code | `status.Errorf(codes.NotFound, "...")` |

### [04 · Production-Grade APIs](level-4/04-production-grade-apis.md)

| Concern | Setting / pattern |
|---|---|
| Protect against slow header writers | `http.Server{ReadHeaderTimeout: d}` |
| Bound full request/response time | `ReadTimeout`, `WriteTimeout` |
| Idle keep-alive connections | `IdleTimeout` |
| Structured JSON error response | Custom `apiError` type + a `writeError` helper |
| Survive a handler panic | `recoverMiddleware` wrapping the whole mux |
| Stop accepting new work, finish in-flight | `srv.Shutdown(ctx)` on `SIGINT`/`SIGTERM` |
| Distinguish clean shutdown from real failure | `errors.Is(err, http.ErrServerClosed)` |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Task | Command |
|---|---|
| Run tests with coverage | `go test -coverprofile=cover.out ./...` |
| Per-function coverage report | `go tool cover -func=cover.out` |
| Browsable HTML coverage | `go tool cover -html=cover.out -o cover.html` |
| Gate a test behind a build tag | `//go:build integration` (first line, blank line after) |
| Run tagged tests | `go test -tags=integration ./...` |
| Race detector in CI | `go test -race ./...` |
| Lint beyond `go vet` | `golangci-lint run` (`golangci-lint-action` in Actions) |

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

| Task | Command / directive |
|---|---|
| Static cross-compile for Linux | `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ...` |
| Strip debug info, shrink binary | `-ldflags="-s -w"` |
| Minimal build stage | `FROM golang:1.23 AS build` |
| Minimal final image (CGO-free, no TLS needed) | `FROM scratch` |
| Minimal final image, TLS or `/etc/passwd` needed | `FROM gcr.io/distroless/static` |
| Cache module downloads across builds | `COPY go.mod go.sum ./` then `go mod download`, *before* `COPY . .` |
| Copy the built binary into the final stage | `COPY --from=build /app /app` |

### [07 · Observability](level-4/07-observability.md)

| Need | API |
|---|---|
| Structured JSON logs | `slog.New(slog.NewJSONHandler(os.Stdout, nil))` |
| Log a field | `slog.String(k, v)`, `slog.Int(k, v)`, `slog.Duration(k, d)` |
| Counter metric | `promauto.NewCounterVec(prometheus.CounterOpts{...}, labels)` |
| Histogram (latency) metric | `promauto.NewHistogramVec(prometheus.HistogramOpts{...}, labels)` |
| Increment/observe | `.WithLabelValues(...).Inc()` / `.Observe(seconds)` |
| Expose a scrape endpoint | `mux.Handle("GET /metrics", promhttp.Handler())` |
| Capture the response status in middleware | Wrap `http.ResponseWriter`, override `WriteHeader` |

### [08 · Security Best Practices](level-4/08-security-best-practices.md)

| Need | API |
|---|---|
| Hash a password | `bcrypt.GenerateFromPassword([]byte(pw), bcrypt.DefaultCost)` |
| Verify a password | `bcrypt.CompareHashAndPassword(hash, pw) == nil` |
| Sign a message | `hmac.New(sha256.New, secret)` then `.Write`/`.Sum(nil)` |
| Compare secrets safely | `subtle.ConstantTimeCompare(a, b) == 1` |
| Cryptographically secure random bytes | `crypto/rand.Read(buf)` |
| Parameterized SQL | `db.Query("... WHERE x = ?", val)` — never string-concatenate |
| Per-endpoint rate limiting | `rate.NewLimiter(eventsPerSecond, burst)`, `.Allow()` |

### [09 · Performance Optimization](level-4/09-performance-optimization.md)

| Optimization | How |
|---|---|
| Avoid slice regrowth | `make([]T, 0, n)` when `n` is known ahead of time |
| Avoid a misleading zero-allocation benchmark | Assign the result to a package-level `sink` var after the loop |
| See why something escapes to the heap | `go build -gcflags="-m" .` |
| Prefer values over pointers for small structs | `[]T` in hot paths, `[]*T` only for large/shared/mutable elements |
| Confirm a benchmark isn't dead code | Compare with and without a sink; a real workload's numbers shouldn't collapse to zero |

---

Go deeper with the full [Go Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
