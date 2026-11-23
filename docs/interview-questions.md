---
title: "Go Interview Questions and Answers (179 Questions)"
description: "179 Go interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Go Mastery Path."
---

# Go Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Go Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What is Go?

**Answer:** A statically typed, compiled language with garbage collection and built-in concurrency

Fast compile, single binary.

**Q2.** What does `go run main.go` do?

**Answer:** Compiles and runs the program

go build produces a binary.

**Q3.** What is the entry point of a Go program?

**Answer:** func main() in package main

Package main.

**Q4.** What does `go fmt` do?

**Answer:** Formats code to the canonical style

gofmt.

### [02 · Variables, Types &amp; Operators](level-1/02-variables-types-operators.md)

**Q5.** What does this print?

```go
package main
import ("fmt")

func main() {
x := 7
var y float64 = 2
var z int
fmt.Println(x/2, float64(x)/y, z)
}
```

**Answer:** 3 3.5 0

Integer division truncates; zero values default to 0. Verified by compiling and running it.

**Q6.** What does this print?

```go
package main
import ("fmt")

func main() {
var s string
var b bool
var p *int
fmt.Println(s == "", b, p == nil)
}
```

**Answer:** true false true

Zero values: "", false, nil. Verified by compiling and running it.

**Q7.** What's the short declaration operator?

**Answer:** := declares and initializes (inside functions)

Type is inferred.

**Q8.** Are Go's numeric types implicitly converted?

**Answer:** No: you must convert explicitly (float64(x))

Strict typing.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```go
package main
import ("fmt")

func main() {
for i := 0; i < 5; i++ {
 if i == 1 { continue }
 if i == 4 { break }
 fmt.Print(i, " ")
}
}
```

**Answer:** 0 2 3

continue skips, break exits. Verified by compiling and running it.

**Q10.** What does this print?

```go
package main
import ("fmt")

func main() {
switch x := 5; {
case x > 3:
 fmt.Println("big")
 fallthrough
case x > 10:
 fmt.Println("huge")
default:
 fmt.Println("other")
}
}
```

**Answer:** big
huge

Go switch doesn't fall through unless asked. Verified by compiling and running it.

**Q11.** What's Go's only loop keyword?

**Answer:** for

for covers all forms.

**Q12.** What does `for range` over a string yield?

**Answer:** Runes (Unicode code points) with byte indexes

UTF-8 aware.

### [04 · Functions &amp; Multiple Returns](level-1/04-functions-multiple-returns.md)

**Q13.** What does this print?

```go
package main
import ("fmt")
func divmod(a, b int) (int, int) { return a / b, a % b }
func main() {
fmt.Println(divmod(17, 5))
}
```

**Answer:** 3 2

Multiple return values. Verified by compiling and running it.

**Q14.** What does this print?

```go
package main
import ("fmt")
func sum(n ...int) int { t := 0; for _, v := range n { t += v }; return t }
func main() {
fmt.Println(sum(1, 2, 3), sum())
}
```

**Answer:** 6 0

Variadic parameters. Verified by compiling and running it.

**Q15.** What does this print?

```go
package main
import ("fmt")
func adder(base int) func(int) int { return func(x int) int { return base + x } }
func main() {
f := adder(10)
fmt.Println(f(5), f(1))
}
```

**Answer:** 15 11

Closures capture variables. Verified by compiling and running it.

**Q16.** What's a named return value?

**Answer:** A result variable declared in the signature that a bare return returns

Use sparingly.

### [05 · Arrays, Slices &amp; Maps](level-1/05-arrays-slices-maps.md)

**Q17.** What does this print?

```go
package main
import ("fmt")

func main() {
a := []int{1, 2, 3}
b := a[:2]
b = append(b, 99)
fmt.Println(a, b, len(b), cap(b))
}
```

**Answer:** [1 2 99] [1 2 99] 3 3

Slices share the backing array until capacity is exceeded. Verified by compiling and running it.

**Q18.** What does this print?

```go
package main
import ("fmt")

func main() {
m := map[string]int{"a": 1}
m["b"]++
v, ok := m["z"]
delete(m, "a")
fmt.Println(len(m), v, ok)
}
```

**Answer:** 1 0 false

Missing key returns the zero value; use the comma-ok idiom. Verified by compiling and running it.

**Q19.** What does this print?

```go
package main
import ("fmt")

func main() {
var s []int
fmt.Println(s == nil, len(s))
s = append(s, 1)
fmt.Println(s)
}
```

**Answer:** true 0
[1]

A nil slice is usable with append. Verified by compiling and running it.

**Q20.** What's the difference between an array and a slice?

**Answer:** Arrays have a fixed size part of their type; slices are dynamic views of arrays

[3]int vs []int.

### [06 · Structs](level-1/06-structs.md)

**Q21.** What does this print?

```go
package main
import ("fmt")
type Point struct{ X, Y int }
func main() {
p := Point{1, 2}
q := p
q.X = 9
fmt.Println(p, q)
}
```

**Answer:** {1 2} {9 2}

Structs are copied on assignment. Verified by compiling and running it.

**Q22.** What does this print?

```go
package main
import ("fmt")
type Person struct { Name string; Age int }
func main() {
p := Person{Name: "Ada"}
fmt.Printf("%+v\n", p)
}
```

**Answer:** {Name:Ada Age:0}

Unset fields get zero values. Verified by compiling and running it.

**Q23.** How do you embed a type in a struct?

**Answer:** List the type without a field name (composition)

Promotes methods.

**Q24.** What are struct tags used for?

**Answer:** Metadata like `json:"name"` read via reflection

encoding/json.

### [07 · Pointers](level-1/07-pointers.md)

**Q25.** What does this print?

```go
package main
import ("fmt")

func main() {
x := 5
p := &x
*p = 9
fmt.Println(x, *p, p == nil)
}
```

**Answer:** 9 9 false

Pointers refer to the variable. Verified by compiling and running it.

**Q26.** What does this print?

```go
package main
import ("fmt")
func inc(n int) { n++ }
func incP(n *int) { *n++ }
func main() {
a := 1
inc(a)
b := 1
incP(&b)
fmt.Println(a, b)
}
```

**Answer:** 1 2

Go passes by value; pass a pointer to modify. Verified by compiling and running it.

**Q27.** Does Go have pointer arithmetic?

**Answer:** No

Safer than C.

**Q28.** What happens dereferencing a nil pointer?

**Answer:** A runtime panic

Check for nil.

### [08 · Error Handling](level-1/08-error-handling.md)

**Q29.** What does this print?

```go
package main
import ("fmt"; "strconv")

func main() {
_, err := strconv.Atoi("12x")
fmt.Println(err != nil, err)
}
```

**Answer:** true strconv.Atoi: parsing "12x": invalid syntax

Errors are values returned from functions. Verified by compiling and running it.

**Q30.** What's the idiomatic error check?

**Answer:** if err != nil { return err } after each call

Explicit handling.

**Q31.** When use panic?

**Answer:** For unrecoverable programmer errors, not ordinary errors

recover rarely.

**Q32.** What does this print?

```go
package main
import ("fmt")

func main() {
defer fmt.Println("deferred")
fmt.Println("body")
}
```

**Answer:** body
deferred

defer runs when the function returns. Verified by compiling and running it.

### [09 · Packages &amp; Modules](level-1/09-packages-modules.md)

**Q33.** What's a Go module?

**Answer:** A collection of packages with go.mod defining its path and dependencies

go mod init.

**Q34.** What does `go mod tidy` do?

**Answer:** Adds missing and removes unused dependencies in go.mod/go.sum

Keep in sync.

**Q35.** What controls whether an identifier is exported?

**Answer:** Capitalization: Uppercase names are exported

Package visibility.

**Q36.** What file locks dependency hashes?

**Answer:** go.sum

Integrity.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-cli.md)

**Q37.** What does the TODO CLI project combine?

**Answer:** Structs, slices, JSON/file persistence and flag parsing

Level 1 project.

**Q38.** How do you persist tasks?

**Answer:** Marshal to JSON and write to a file

os.WriteFile.

**Q39.** How parse CLI args?

**Answer:** os.Args or the flag package

Standard library.

**Q40.** How handle a missing file?

**Answer:** Check errors.Is(err, os.ErrNotExist) and start empty

Graceful.

## Level 2 · Intermediate

### [01 · Interfaces](level-2/01-interfaces.md)

**Q41.** What does this print?

```go
package main
import ("fmt"; "math")
type Shape interface{ Area() float64 }
type Rect struct{ W, H float64 }
func (r Rect) Area() float64 { return r.W * r.H }
type Circle struct{ R float64 }
func (c Circle) Area() float64 { return math.Pi * c.R * c.R }
func main() {
var s Shape = Rect{2, 3}
fmt.Println(s.Area())
var t Shape = Circle{1}
fmt.Printf("%.2f\n", t.Area())
}
```

**Answer:** 6
3.14

Interfaces are satisfied implicitly. Verified by compiling and running it.

**Q42.** How does a type implement an interface in Go?

**Answer:** Implicitly by having the methods; no implements keyword

Structural typing.

**Q43.** What's the empty interface?

**Answer:** any (interface{}), satisfied by every type

Use sparingly.

**Q44.** What does this print?

```go
package main
import ("fmt")

func main() {
var i any = "hi"
s, ok := i.(string)
n, ok2 := i.(int)
fmt.Println(s, ok, n, ok2)
}
```

**Answer:** hi true 0 false

Type assertions with comma-ok. Verified by compiling and running it.

### [02 · Goroutines &amp; Channels Basics](level-2/02-goroutines-channels.md)

**Q45.** What does this print?

```go
package main
import ("fmt")

func main() {
ch := make(chan int)
go func() { for i := 1; i <= 3; i++ { ch <- i }; close(ch) }()
sum := 0
for v := range ch { sum += v }
fmt.Println(sum)
}
```

**Answer:** 6

Range over a channel until closed. Verified by compiling and running it.

**Q46.** What does this print?

```go
package main
import ("fmt"; "sync")

func main() {
var wg sync.WaitGroup
var mu sync.Mutex
count := 0
for i := 0; i < 100; i++ {
 wg.Add(1)
 go func() { defer wg.Done(); mu.Lock(); count++; mu.Unlock() }()
}
wg.Wait()
fmt.Println(count)
}
```

**Answer:** 100

The mutex protects the shared counter. Verified by compiling and running it.

**Q47.** What does an unbuffered channel do on send?

**Answer:** Blocks until a receiver is ready

Synchronization.

**Q48.** What's the Go concurrency motto?

**Answer:** Don't communicate by sharing memory; share memory by communicating

Channels.

### [03 · Methods &amp; Receivers](level-2/03-methods-receivers.md)

**Q49.** What does this print?

```go
package main
import ("fmt")
type Counter struct{ n int }
func (c *Counter) Inc() { c.n++ }
func main() {
c := &Counter{}
c.Inc()
c.Inc()
fmt.Println(c.n)
v := Counter{}
v.Inc()
fmt.Println(v.n)
}
```

**Answer:** 2
1

Pointer receivers modify the value (addressable vars get auto &amp;). Verified by compiling and running it.

**Q50.** When use a pointer receiver?

**Answer:** When the method modifies the receiver or the struct is large

Be consistent.

**Q51.** Can you define methods on built-in types?

**Answer:** Not directly; define a new named type first

type MyInt int.

**Q52.** What's a method value?

**Answer:** A method bound to a receiver, usable as a function

f := obj.Method.

### [04 · Custom Errors &amp; Error Wrapping](level-2/04-custom-errors-wrapping.md)

**Q53.** What does this print?

```go
package main
import ("errors"; "fmt")
var ErrNotFound = errors.New("not found")
func main() {
e := fmt.Errorf("read: %w", ErrNotFound)
fmt.Println(e, errors.Is(e, ErrNotFound))
}
```

**Answer:** read: not found true

%w wraps errors for errors.Is/As. Verified by compiling and running it.

**Q54.** What does errors.As do?

**Answer:** Finds a specific error type in the chain and assigns it

Typed errors.

**Q55.** What's a sentinel error?

**Answer:** A package-level error value compared with errors.Is

io.EOF.

**Q56.** Why wrap errors with context?

**Answer:** To add where/why while preserving the cause

%w.

### [05 · JSON Encoding/Decoding](level-2/05-json-encoding-decoding.md)

**Q57.** What does this print?

```go
package main
import ("encoding/json"; "fmt")

func main() {
type U struct { Name string `json:"name"`; Age int `json:"age,omitempty"`; Pass string `json:"-"` }
b, _ := json.Marshal(U{"Ada", 0, "x"})
fmt.Println(string(b))
}
```

**Answer:** {"name":"Ada"}

omitempty skips zero values; "-" skips the field. Verified by compiling and running it.

**Q58.** What does this print?

```go
package main
import ("encoding/json"; "fmt")

func main() {
var m map[string]any
json.Unmarshal([]byte(`{"a":1,"b":[true,"x"]}`), &m)
fmt.Println(m["a"], m["b"])
}
```

**Answer:** 1 [true x]

JSON numbers decode to float64 in any (printed as 1). Verified by compiling and running it.

**Q59.** What types can't be JSON encoded?

**Answer:** Channels, functions and complex numbers

Marshal returns an error.

**Q60.** What does json.NewDecoder suit?

**Answer:** Streaming from an io.Reader like an HTTP body

Efficient.

### [06 · Testing with the testing Package](level-2/06-testing-package.md)

**Q61.** How do you run tests in Go?

**Answer:** go test ./...

Files end with _test.go.

**Q62.** What's a table-driven test?

**Answer:** A slice of cases iterated in one test function, often with t.Run subtests

Idiomatic.

**Q63.** What does t.Fatal do?

**Answer:** Fails and stops the test immediately

t.Error continues.

**Q64.** What does go test -race do?

**Answer:** Detects data races at runtime

Run in CI.

### [07 · File I/O](level-2/07-file-io.md)

**Q65.** What reads a whole file in one call?

**Answer:** os.ReadFile

Small files.

**Q66.** What should you do after os.Open?

**Answer:** defer f.Close()

Resource cleanup.

**Q67.** What does bufio.Scanner do?

**Answer:** Reads input line by line (or by token) efficiently

Large files.

**Q68.** What's the permission 0644?

**Answer:** Owner read/write, others read

File mode.

### [08 · Working with net/http](level-2/08-net-http-client.md)

**Q69.** What does http.Client need besides defaults?

**Answer:** A timeout, since the default client has none

Avoid hangs.

**Q70.** What must you always do with a response body?

**Answer:** Close it (defer resp.Body.Close())

Prevent leaks.

**Q71.** How check success?

**Answer:** resp.StatusCode (non-2xx isn't an error from Get)

Handle codes.

**Q72.** How decode JSON from a response?

**Answer:** json.NewDecoder(resp.Body).Decode(&amp;v)

Stream.

### [09 · Dependency Management](level-2/09-dependency-management.md)

**Q73.** What does `go get` do?

**Answer:** Adds or updates a dependency in go.mod

Module-aware.

**Q74.** What's semantic import versioning?

**Answer:** Major versions v2+ are part of the module path

github.com/x/y/v2.

**Q75.** What does `go mod vendor` do?

**Answer:** Copies dependencies into a vendor directory

Reproducible builds.

**Q76.** Why commit go.sum?

**Answer:** Verifies dependency integrity

Supply chain.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** What does the weather CLI combine?

**Answer:** HTTP client, JSON decoding, error handling, flags and tests

Level 2 project.

**Q78.** How do you test HTTP code?

**Answer:** httptest.NewServer returning canned responses

Deterministic.

**Q79.** How should API keys be supplied?

**Answer:** Environment variables or flags, never hard-coded

Security.

**Q80.** How handle rate limits?

**Answer:** Respect 429/Retry-After with backoff

Be polite.

## Level 3 · Advanced

### [01 · Concurrency Patterns](level-3/01-concurrency-patterns.md)

**Q81.** What does this print?

```go
package main
import ("fmt"; "sync")

func main() {
jobs := make(chan int, 5)
results := make(chan int, 5)
var wg sync.WaitGroup
for w := 0; w < 3; w++ {
 wg.Add(1)
 go func() { defer wg.Done(); for j := range jobs { results <- j * j } }()
}
for i := 1; i <= 5; i++ { jobs <- i }
close(jobs)
wg.Wait()
close(results)
s := 0
for r := range results { s += r }
fmt.Println(s)
}
```

**Answer:** 55

Worker pool: sum of squares 1..5. Verified by compiling and running it.

**Q82.** What's fan-out/fan-in?

**Answer:** Distributing work to many goroutines and merging their results

Pipeline pattern.

**Q83.** What does select do?

**Answer:** Waits on multiple channel operations, choosing a ready one

Timeouts.

**Q84.** How avoid goroutine leaks?

**Answer:** Ensure goroutines can exit (close channels, use context cancellation)

Lifecycle.

### [02 · Building REST APIs](level-3/02-building-rest-apis.md)

**Q85.** Which package serves HTTP in the standard library?

**Answer:** net/http

ServeMux, Handler.

**Q86.** What's a Handler?

**Answer:** An interface with ServeHTTP(w, r)

HandlerFunc adapts functions.

**Q87.** What does Go 1.22+ ServeMux support?

**Answer:** Method and path-parameter patterns like "GET /items/{id}"

No router needed for simple APIs.

**Q88.** What's middleware in Go?

**Answer:** A function wrapping a Handler to add behavior

Chain them.

### [03 · Databases &amp; SQL](level-3/03-databases-sql.md)

**Q89.** What's database/sql?

**Answer:** A generic interface; you import a driver for your DB

sql.Open.

**Q90.** How avoid SQL injection?

**Answer:** Use placeholders with args (db.Query("... ?", v))

Parameterized queries.

**Q91.** What must you do with rows?

**Answer:** Close them and check rows.Err()

Resource handling.

**Q92.** What's sql.DB?

**Answer:** A connection pool safe for concurrent use

Create once.

### [04 · The `context` Package](level-3/04-context-package.md)

**Q93.** What does this print?

```go
package main
import ("context"; "fmt"; "time")

func main() {
ctx, cancel := context.WithTimeout(context.Background(), 20*time.Millisecond)
defer cancel()
select {
case <-time.After(time.Second):
 fmt.Println("done")
case <-ctx.Done():
 fmt.Println(ctx.Err())
}
}
```

**Answer:** context deadline exceeded

Timeouts via context. Verified by compiling and running it.

**Q94.** What should context be in function signatures?

**Answer:** The first parameter, named ctx

Convention.

**Q95.** What does cancel() do?

**Answer:** Signals all derived contexts to stop and releases resources

Always defer it.

**Q96.** What shouldn't go in context values?

**Answer:** Optional parameters or business data; use it for request-scoped metadata

Misuse.

### [05 · Design Patterns, the Go Way](level-3/05-design-patterns.md)

**Q97.** What does the functional options pattern do?

**Answer:** Configures constructors with variadic option funcs (New(opts ...Option))

Flexible APIs.

**Q98.** What's idiomatic dependency injection in Go?

**Answer:** Pass interfaces via constructors

Simple.

**Q99.** What does the 'accept interfaces, return structs' proverb mean?

**Answer:** Parameters should be small interfaces; return concrete types

Flexibility.

**Q100.** Why avoid singletons?

**Answer:** Hidden global state hampers testing

Inject instead.

### [06 · Testing, Advanced](level-3/06-testing-advanced.md)

**Q101.** What's a mock/fake approach in Go?

**Answer:** Define small interfaces and substitute implementations in tests

Interface-driven.

**Q102.** What does t.Helper do?

**Answer:** Marks a function as a helper so failures report the caller's line

Cleaner output.

**Q103.** What's httptest.ResponseRecorder?

**Answer:** An in-memory ResponseWriter to test handlers

No network.

**Q104.** What does t.Parallel do?

**Answer:** Runs the test in parallel with other parallel tests

Speed.

### [07 · Profiling &amp; Benchmarking](level-3/07-profiling-benchmarking.md)

**Q105.** What do Go benchmarks look like?

**Answer:** func BenchmarkX(b *testing.B) { for i := 0; i &lt; b.N; i++ { ... } }

go test -bench.

**Q106.** What does pprof show?

**Answer:** CPU and memory profiles to find hotspots

go tool pprof.

**Q107.** What's -benchmem?

**Answer:** Reports allocations per operation

Reduce allocs.

**Q108.** What's first in optimization?

**Answer:** Measure with benchmarks/profiles

Evidence.

### [08 · Building CLIs](level-3/08-building-clis.md)

**Q109.** What package parses command-line flags in the stdlib?

**Answer:** flag

Subcommands via FlagSet.

**Q110.** What do cobra/urfave offer?

**Answer:** Subcommands, help and completions

CLI frameworks.

**Q111.** How exit with a failure code?

**Answer:** os.Exit(1) after printing to stderr

Scripts check status.

**Q112.** Where should errors print?

**Answer:** stderr, with results on stdout

Unix convention.

### [09 · Interfaces &amp; Generics](level-3/09-interfaces-generics.md)

**Q113.** What does this print?

```go
package main
import ("fmt")
type Ordered interface{ ~int | ~string }
func Max[T Ordered](a, b T) T { if a > b { return a }; return b }
func Map[T, U any](s []T, f func(T) U) []U { r := make([]U, 0, len(s)); for _, v := range s { r = append(r, f(v)) }; return r }
func main() {
fmt.Println(Max(3, 7), Max("a", "b"), Map([]int{1, 2, 3}, func(x int) string { return fmt.Sprint(x * 2) }))
}
```

**Answer:** 7 b [2 4 6]

Generics with constraints. Verified by compiling and running it.

**Q114.** What does `comparable` constrain?

**Answer:** Types usable with == and !=

Map keys.

**Q115.** When use generics?

**Answer:** For reusable containers/algorithms across types, not everywhere

Prefer simple code.

**Q116.** What's type inference in generics?

**Answer:** The compiler deduces type arguments from the call

Max(3,7).

### [10 · Project: REST API with SQLite](level-3/10-project-rest-api-sqlite.md)

**Q117.** What does the REST API + SQLite project combine?

**Answer:** net/http handlers, database/sql, JSON and tests

Level 3 project.

**Q118.** How should handlers get the DB?

**Answer:** Through a struct holding dependencies injected at startup

Testability.

**Q119.** How test handlers?

**Answer:** httptest with an in-memory or temp SQLite DB

Isolation.

**Q120.** How migrate schemas?

**Answer:** Versioned SQL migrations applied at startup or by a tool

Repeatable.

## Level 4 · Master

### [01 · Advanced Concurrency](level-4/01-advanced-concurrency.md)

**Q121.** What's errgroup for?

**Answer:** Running goroutines and collecting the first error, with context cancellation

golang.org/x/sync.

**Q122.** What does sync.Once do?

**Answer:** Runs an initialization exactly once, safely

Lazy init.

**Q123.** What's the race detector's cost?

**Answer:** About 5-10x slower and more memory; use in tests/CI

-race.

**Q124.** What are atomic operations for?

**Answer:** Lock-free updates to simple shared values

sync/atomic.

### [02 · Microservices Patterns](level-4/02-microservices-patterns.md)

**Q125.** What's a circuit breaker?

**Answer:** Stops calling a failing dependency for a period to let it recover

Resilience.

**Q126.** What do timeouts and retries need?

**Answer:** Jitter/backoff and budgets to avoid retry storms

Careful.

**Q127.** What's idempotency for APIs?

**Answer:** Repeating a request has the same effect as one

Safe retries.

**Q128.** What's service discovery?

**Answer:** How services find each other's addresses

Registry/DNS.

### [03 · gRPC Basics](level-4/03-grpc-basics.md)

**Q129.** What does gRPC use?

**Answer:** Protocol Buffers over HTTP/2 with generated clients/servers

Typed contracts.

**Q130.** What's a .proto file?

**Answer:** The schema defining messages and services

protoc generates code.

**Q131.** What are gRPC streaming modes?

**Answer:** Unary, server, client and bidirectional streaming

Flexibility.

**Q132.** When prefer REST over gRPC?

**Answer:** Public/browser APIs needing simple HTTP tooling

Trade-offs.

### [04 · Production-Grade APIs](level-4/04-production-grade-apis.md)

**Q133.** What makes an API production-grade?

**Answer:** Timeouts, graceful shutdown, logging, metrics, validation and tests

Operational.

**Q134.** How do you shut down an HTTP server gracefully?

**Answer:** server.Shutdown(ctx) on SIGTERM

Finish in-flight requests.

**Q135.** What should you set on http.Server?

**Answer:** Read/Write/Idle timeouts

Prevent slowloris.

**Q136.** How validate input?

**Answer:** Explicitly decode with limits and validate fields

http.MaxBytesReader.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** What should CI run for Go?

**Answer:** go vet, go test -race ./... and linting (golangci-lint)

Gate merges.

**Q138.** What does go vet do?

**Answer:** Finds suspicious constructs

Static checks.

**Q139.** How measure coverage?

**Answer:** go test -cover / -coverprofile

Not a goal.

**Q140.** How speed CI?

**Answer:** Cache the module and build caches

GOCACHE.

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

**Q141.** Why multi-stage Docker builds for Go?

**Answer:** Build in a Go image, copy the static binary to a minimal image

Small images.

**Q142.** What does CGO_ENABLED=0 give?

**Answer:** A static binary without C dependencies

Scratch images.

**Q143.** Why run as non-root in containers?

**Answer:** Limit damage if compromised

Least privilege.

**Q144.** What's a health endpoint for?

**Answer:** Orchestrator liveness/readiness checks

/healthz.

### [07 · Observability](level-4/07-observability.md)

**Q145.** What are the observability pillars?

**Answer:** Logs, metrics and traces

Together.

**Q146.** What does log/slog provide?

**Answer:** Structured leveled logging in the standard library

JSON handlers.

**Q147.** What does OpenTelemetry offer?

**Answer:** Vendor-neutral tracing and metrics instrumentation

Standard.

**Q148.** What's a request ID for?

**Answer:** Correlating logs across services

Middleware.

### [08 · Security Best Practices](level-4/08-security-best-practices.md)

**Q149.** How hash passwords?

**Answer:** bcrypt/argon2 (golang.org/x/crypto), never plain or fast hashes

Slow salted.

**Q150.** How prevent SQL injection?

**Answer:** Parameterized queries

Placeholders.

**Q151.** What does `govulncheck` do?

**Answer:** Reports known vulnerabilities in dependencies you actually call

Supply chain.

**Q152.** Where store secrets?

**Answer:** Environment/secret manager, not in code or images

Rotate.

### [09 · Performance Optimization](level-4/09-performance-optimization.md)

**Q153.** What reduces allocations?

**Answer:** Preallocating slices/maps and reusing buffers (sync.Pool)

Escape analysis.

**Q154.** What does `strings.Builder` do?

**Answer:** Builds strings efficiently without repeated allocation

Avoid +=.

**Q155.** How find allocation sources?

**Answer:** go build -gcflags=-m and pprof alloc profiles

Tools.

**Q156.** What's escape analysis?

**Answer:** The compiler deciding whether values live on the stack or heap

Stack is cheaper.

### [10 · Capstone: A Production-Shaped URL Shortener](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** API, persistence, concurrency, observability, security and deployment

Production service.

**Q158.** How prove readiness?

**Answer:** Tests, load checks, monitoring and rollback

Operate.

**Q159.** What documentation helps?

**Answer:** README, architecture notes, API docs and runbooks

Onboarding.

**Q160.** What guards regressions?

**Answer:** CI with tests, race detector and vulnerability scan

Automation.

## Mixed questions across all levels

**Q161.** What does this print?

```go
package main
import ("fmt")

func main() {
x := 7
fmt.Println(x/2, float64(x)/2)
}
```

**Answer:** 3 3.5

Integer division truncates; convert explicitly. Verified by compiling and running it.

**Q162.** What does this print?

```go
package main
import ("fmt")

func main() {
a := []int{1, 2, 3}
b := a[:2]
b = append(b, 99)
fmt.Println(a, b)
}
```

**Answer:** [1 2 99] [1 2 99]

Appending within capacity overwrites the shared array. Verified by compiling and running it.

**Q163.** What does this print?

```go
package main
import ("fmt")

func main() {
m := map[string]int{}
v, ok := m["x"]
m["x"]++
fmt.Println(v, ok, m["x"])
}
```

**Answer:** 0 false 1

Missing keys give zero values. Verified by compiling and running it.

**Q164.** What does this print?

```go
package main
import ("fmt")
type Point struct{ X, Y int }
func main() {
p := Point{1, 2}
q := p
q.X = 9
fmt.Println(p, q)
}
```

**Answer:** {1 2} {9 2}

Structs copy on assignment. Verified by compiling and running it.

**Q165.** What does this print?

```go
package main
import ("fmt")
func inc(n int) { n++ }
func incP(n *int) { *n++ }
func main() {
a := 1
inc(a)
b := 1
incP(&b)
fmt.Println(a, b)
}
```

**Answer:** 1 2

Pass a pointer to modify. Verified by compiling and running it.

**Q166.** What does this print?

```go
package main
import ("errors"; "fmt")
var ErrNotFound = errors.New("not found")
func main() {
e := fmt.Errorf("load: %w", ErrNotFound)
fmt.Println(errors.Is(e, ErrNotFound), e)
}
```

**Answer:** true load: not found

%w keeps the chain for errors.Is. Verified by compiling and running it.

**Q167.** What does this print?

```go
package main
import ("fmt")

func main() {
defer fmt.Println("one")
defer fmt.Println("two")
fmt.Println("body")
}
```

**Answer:** body
two
one

Deferred calls run last-in first-out. Verified by compiling and running it.

**Q168.** What does this print?

```go
package main
import ("fmt")
type Shape interface{ Area() int }
type Square struct{ S int }
func (q Square) Area() int { return q.S * q.S }
func main() {
var s Shape = Square{3}
fmt.Println(s.Area())
}
```

**Answer:** 9

Implicit interface satisfaction. Verified by compiling and running it.

**Q169.** What does this print?

```go
package main
import ("fmt"; "sync")

func main() {
var wg sync.WaitGroup
var mu sync.Mutex
n := 0
for i := 0; i < 50; i++ {
 wg.Add(1)
 go func() { defer wg.Done(); mu.Lock(); n++; mu.Unlock() }()
}
wg.Wait()
fmt.Println(n)
}
```

**Answer:** 50

A mutex protects shared state. Verified by compiling and running it.

**Q170.** What does this print?

```go
package main
import ("fmt")

func main() {
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)
for v := range ch { fmt.Print(v, " ") }
v, ok := <-ch
fmt.Println(v, ok)
}
```

**Answer:** 1 2 0 false

Receiving from a closed channel yields the zero value with ok=false. Verified by compiling and running it.

**Q171.** What does this print?

```go
package main
import ("context"; "fmt"; "time")

func main() {
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
defer cancel()
<-ctx.Done()
fmt.Println(ctx.Err())
}
```

**Answer:** context deadline exceeded

Contexts carry deadlines and cancellation. Verified by compiling and running it.

**Q172.** What does this print?

```go
package main
import ("fmt")
type Ordered interface{ ~int | ~string }
func Max[T Ordered](a, b T) T { if a > b { return a }; return b }
func main() {
fmt.Println(Max(3, 9), Max("pear", "apple"))
}
```

**Answer:** 9 pear

Generic functions with a constraint. Verified by compiling and running it.

**Q173.** A `for range` loop starts goroutines that all print the final loop value in older Go. What's the standard fix?

**Answer:** Pass the variable as an argument or copy it per iteration (Go 1.22+ scopes it per iteration)

Loop variable capture.

**Q174.** How do you make an HTTP client call safe against hangs?

**Answer:** Set a timeout (client Timeout or request context) and always close the response body

The default client has no timeout.

**Q175.** Which tool finds data races?

**Answer:** go test -race

Run it in CI.

**Q176.** Where should the DB handle live in a handler-based service?

**Answer:** In a struct injected into handlers, created once at startup

sql.DB is a pool.

**Q177.** How do you shut an HTTP server down gracefully?

**Answer:** Call server.Shutdown(ctx) when SIGTERM arrives so in-flight requests finish

Production readiness.

**Q178.** A container image for a Go service is 900 MB. What's the fix?

**Answer:** Multi-stage build copying a static binary into a minimal base image

CGO_ENABLED=0.

**Q179.** How do you find a performance bottleneck?

**Answer:** Benchmark and profile (pprof) before optimizing

Measure first.

## Practical coding questions

### A worker pool with errors and cancellation

Complete squareAll: results are collected by original index, negative inputs produce per-item errors ("item N: negative input") combined with errors.Join in sorted order, and a cancelled context returns ctx.Err(). Three runs show success, partial failure and cancellation.

Starter code:

```
package main

import (
	"context"
	"errors"
	"fmt"
	"sort"
	"sync"
)

type result struct {
	id  int
	val int
	err error
}

// squareAll squares each number using `workers` goroutines. It stops early when ctx is cancelled.
func squareAll(ctx context.Context, nums []int, workers int) ([]int, error) {
	jobs := make(chan int)
	results := make(chan result)
	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := range jobs {
				if nums[i] < 0 {
					results <- result{id: i, err: errors.New("negative input")}
					continue
				}
				results <- result{id: i, val: nums[i] * nums[i]}
			}
		}()
	}
	go func() {
		defer close(jobs)
		for i := range nums {
			select {
			case jobs <- i:
			case <-ctx.Done():
				return
			}
		}
	}()
	go func() { wg.Wait(); close(results) }()

	// collect results into out (index = original position); gather per-item errors as
	// fmt.Errorf("item %d: %w", id, err), sort them, and return errors.Join(...);
	// when there are no item errors, return out and ctx.Err()
	return nil, nil
}

func main() {
	out, err := squareAll(context.Background(), []int{1, 2, 3, 4, 5}, 3)
	fmt.Println(out, err)
	out, err = squareAll(context.Background(), []int{2, -3, 4, -1}, 2)
	fmt.Println(out, err)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()
	_, err = squareAll(ctx, []int{1, 2, 3}, 2)
	fmt.Println(err)
}
```

Expected output:

```text
[1 4 9 16 25] <nil>
[4 0 16 0] item 1: negative input
item 3: negative input
context canceled
```

??? success "Show a solution"

    ```
    package main

    import (
    	"context"
    	"errors"
    	"fmt"
    	"sort"
    	"sync"
    )

    type result struct {
    	id  int
    	val int
    	err error
    }

    // squareAll squares each number using `workers` goroutines. It stops early when ctx is cancelled.
    func squareAll(ctx context.Context, nums []int, workers int) ([]int, error) {
    	jobs := make(chan int)
    	results := make(chan result)
    	var wg sync.WaitGroup
    	for w := 0; w < workers; w++ {
    		wg.Add(1)
    		go func() {
    			defer wg.Done()
    			for i := range jobs {
    				if nums[i] < 0 {
    					results <- result{id: i, err: errors.New("negative input")}
    					continue
    				}
    				results <- result{id: i, val: nums[i] * nums[i]}
    			}
    		}()
    	}
    	go func() {
    		defer close(jobs)
    		for i := range nums {
    			select {
    			case jobs <- i:
    			case <-ctx.Done():
    				return
    			}
    		}
    	}()
    	go func() { wg.Wait(); close(results) }()

    	out := make([]int, len(nums))
    	var errs []error
    	for r := range results {
    		if r.err != nil {
    			errs = append(errs, fmt.Errorf("item %d: %w", r.id, r.err))
    			continue
    		}
    		out[r.id] = r.val
    	}
    	if len(errs) > 0 {
    		sort.Slice(errs, func(i, j int) bool { return errs[i].Error() < errs[j].Error() })
    		return out, errors.Join(errs...)
    	}
    	return out, ctx.Err()
    }

    func main() {
    	out, err := squareAll(context.Background(), []int{1, 2, 3, 4, 5}, 3)
    	fmt.Println(out, err)
    	out, err = squareAll(context.Background(), []int{2, -3, 4, -1}, 2)
    	fmt.Println(out, err)
    	ctx, cancel := context.WithCancel(context.Background())
    	cancel()
    	_, err = squareAll(ctx, []int{1, 2, 3}, 2)
    	fmt.Println(err)
    }
    ```

    Workers read indexes from a jobs channel, results carry their original index so order is preserved, a goroutine closes results after the WaitGroup, and the producer stops on ctx.Done(). Verified by running the solution with go run.

### Test an HTTP handler with httptest

Implement the handler: POST /items (body size limit, 400 on invalid or empty name, 201 with the stored item) and GET /items/{id} (200 or 404 for unknown/non-numeric ids), guarded by a mutex. The do() helper drives it with httptest.

Starter code:

```
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strconv"
	"strings"
	"sync"
)

type Item struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

type Store struct {
	mu    sync.Mutex
	items map[int]Item
	next  int
}

func NewStore() *Store { return &Store{items: map[int]Item{}, next: 1} }

func (s *Store) Handler() http.Handler {
	mux := http.NewServeMux()
	// POST /items: decode {"name": ...} with a body size limit; 400 {"error":"name is required"} when
	//   decoding fails or the trimmed name is empty; otherwise store under the next id and reply 201 + JSON
	// GET /items/{id}: 200 + JSON, or 404 {"error":"not found"} for unknown or non-numeric ids
	// (use r.PathValue("id") and guard the map with the mutex)
	return mux
}

func do(h http.Handler, method, path, body string) string {
	req := httptest.NewRequest(method, path, strings.NewReader(body))
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	return fmt.Sprintf("%d %s", rec.Code, strings.TrimSpace(rec.Body.String()))
}

func main() {
	h := NewStore().Handler()
	fmt.Println(do(h, "POST", "/items", `{"name":"pen"}`))
	fmt.Println(do(h, "POST", "/items", `{"name":"  "}`))
	fmt.Println(do(h, "GET", "/items/1", ""))
	fmt.Println(do(h, "GET", "/items/9", ""))
	fmt.Println(do(h, "GET", "/items/abc", ""))
	fmt.Println(do(h, "DELETE", "/items/1", ""))
}
```

Expected output:

```text
201 {"id":1,"name":"pen"}
400 {"error":"name is required"}
200 {"id":1,"name":"pen"}
404 {"error":"not found"}
404 {"error":"not found"}
405 Method Not Allowed
```

??? success "Show a solution"

    ```
    package main

    import (
    	"encoding/json"
    	"fmt"
    	"net/http"
    	"net/http/httptest"
    	"strconv"
    	"strings"
    	"sync"
    )

    type Item struct {
    	ID   int    `json:"id"`
    	Name string `json:"name"`
    }

    type Store struct {
    	mu    sync.Mutex
    	items map[int]Item
    	next  int
    }

    func NewStore() *Store { return &Store{items: map[int]Item{}, next: 1} }

    func (s *Store) Handler() http.Handler {
    	mux := http.NewServeMux()
    	mux.HandleFunc("POST /items", func(w http.ResponseWriter, r *http.Request) {
    		var in struct{ Name string `json:"name"` }
    		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<10)).Decode(&in); err != nil || strings.TrimSpace(in.Name) == "" {
    			http.Error(w, `{"error":"name is required"}`, http.StatusBadRequest)
    			return
    		}
    		s.mu.Lock()
    		it := Item{ID: s.next, Name: in.Name}
    		s.items[it.ID] = it
    		s.next++
    		s.mu.Unlock()
    		w.Header().Set("Content-Type", "application/json")
    		w.WriteHeader(http.StatusCreated)
    		json.NewEncoder(w).Encode(it)
    	})
    	mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
    		id, err := strconv.Atoi(r.PathValue("id"))
    		s.mu.Lock()
    		it, ok := s.items[id]
    		s.mu.Unlock()
    		if err != nil || !ok {
    			http.Error(w, `{"error":"not found"}`, http.StatusNotFound)
    			return
    		}
    		w.Header().Set("Content-Type", "application/json")
    		json.NewEncoder(w).Encode(it)
    	})
    	return mux
    }

    func do(h http.Handler, method, path, body string) string {
    	req := httptest.NewRequest(method, path, strings.NewReader(body))
    	rec := httptest.NewRecorder()
    	h.ServeHTTP(rec, req)
    	return fmt.Sprintf("%d %s", rec.Code, strings.TrimSpace(rec.Body.String()))
    }

    func main() {
    	h := NewStore().Handler()
    	fmt.Println(do(h, "POST", "/items", `{"name":"pen"}`))
    	fmt.Println(do(h, "POST", "/items", `{"name":"  "}`))
    	fmt.Println(do(h, "GET", "/items/1", ""))
    	fmt.Println(do(h, "GET", "/items/9", ""))
    	fmt.Println(do(h, "GET", "/items/abc", ""))
    	fmt.Println(do(h, "DELETE", "/items/1", ""))
    }
    ```

    httptest lets you exercise handlers in-process with no network. The Go 1.22 ServeMux patterns ("POST /items", "GET /items/{id}") route by method and path, and unmatched methods get 405 automatically. Verified by running the solution with go run.

---

More free practice: the [Go Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
