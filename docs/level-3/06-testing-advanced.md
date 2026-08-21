# 06 · Testing, Advanced

Level 1/2 covered `t.Run` table tests and basic assertions. This module adds
the three tools that separate "tests exist" from "tests actually find bugs
and stay fast": subtests with table-driven cases done properly, fuzzing,
and benchmarks — plus `testing.T` helpers that make failures readable.

## The package under test

```go
package calc

import "errors"

var ErrDivByZero = errors.New("division by zero")

func Add(a, b int) int { return a + b }

func Divide(a, b int) (int, error) {
	if b == 0 {
		return 0, ErrDivByZero
	}
	return a / b, nil
}
```

## Table-driven tests with named subtests

```go
package calc

import (
	"errors"
	"testing"
)

func TestAdd(t *testing.T) {
	cases := []struct {
		a, b, want int
	}{
		{2, 3, 5},
		{-1, 1, 0},
		{0, 0, 0},
	}
	for _, c := range cases {
		t.Run("", func(t *testing.T) {
			got := Add(c.a, c.b)
			if got != c.want {
				t.Errorf("Add(%d,%d) = %d, want %d", c.a, c.b, got, c.want)
			}
		})
	}
}

func TestDivide(t *testing.T) {
	got, err := Divide(10, 2)
	if err != nil || got != 5 {
		t.Fatalf("Divide(10,2) = %d, %v; want 5, nil", got, err)
	}

	_, err = Divide(1, 0)
	if !errors.Is(err, ErrDivByZero) {
		t.Fatalf("Divide(1,0) err = %v; want ErrDivByZero", err)
	}
}
```

```console
$ go test -v ./...
=== RUN   TestAdd
=== RUN   TestAdd/#00
=== RUN   TestAdd/#01
=== RUN   TestAdd/#02
--- PASS: TestAdd (0.00s)
    --- PASS: TestAdd/#00 (0.00s)
    --- PASS: TestAdd/#01 (0.00s)
    --- PASS: TestAdd/#02 (0.00s)
=== RUN   TestDivide
--- PASS: TestDivide (0.00s)
PASS
ok  	m06	0.636s
```

`t.Run("", ...)` with an empty name still gets an auto-numbered subtest
(`#00`, `#01`, ...) — naming each case (`t.Run(fmt.Sprintf("%d+%d", c.a,
c.b), ...)`) makes `go test -run TestAdd/2\+3` target one row, which the
anonymous version can't do. `t.Fatalf` stops that subtest immediately;
`t.Errorf` records the failure and keeps running the rest of the table —
use `Fatalf` only when a later assertion would panic (like indexing into a
result that's `nil` because the previous check failed).

## Fuzzing

Go's built-in fuzzer generates inputs to try to break a property you assert
holds for *all* inputs, not just the ones you thought to write down:

```go
func FuzzAdd(f *testing.F) {
	f.Add(2, 3) // seed corpus -- a known-good starting input
	f.Fuzz(func(t *testing.T, a, b int) {
		got := Add(a, b)
		want := a + b
		if got != want {
			t.Errorf("Add(%d,%d) = %d, want %d", a, b, got, want)
		}
	})
}
```

```console
$ go test -run=^$ -fuzz=FuzzAdd -fuzztime=3s .
fuzz: elapsed: 0s, gathering baseline coverage: 0/1 completed
fuzz: elapsed: 0s, gathering baseline coverage: 1/1 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 3218456 (1072818/sec), new interesting: 2 (total: 3)
PASS
ok  	m06	3.104s
```

`-run=^$` skips regular tests so only the fuzz target runs; a plain `go
test` (no `-fuzz` flag) instead runs `FuzzAdd`'s seed corpus once as an
ordinary test, which is why `FuzzAdd/seed#0` shows up in the first `go test
-v` output above. A crashing input gets saved under
`testdata/fuzz/FuzzAdd/` and replayed automatically on every future `go
test`, turning a fuzz-found bug into a permanent regression test for free.

## Benchmarks

```go
func BenchmarkAdd(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Add(2, 3)
	}
}
```

```console
$ go test -bench=. -run=^$ .
goos: darwin
goarch: arm64
pkg: m06
cpu: Apple M1
BenchmarkAdd-8   	  100000	         0.9546 ns/op
PASS
ok  	m06	0.312s
```

`b.N` is chosen by the testing framework, growing until the benchmark runs
long enough to measure reliably — never hardcode a loop count. `-run=^$`
again suppresses ordinary tests so the reported time isn't diluted by
unrelated work; `-8` in `BenchmarkAdd-8` is `GOMAXPROCS`, useful context
when comparing runs across machines.

## Go-specific traps

- **`t.Fatalf` inside a goroutine spawned by a test does nothing safe** —
  `FailNow` (which `Fatalf` calls) must run on the goroutine executing the
  test function; calling it elsewhere is documented as producing undefined
  behavior. Send results back over a channel and fail from the main test
  goroutine instead.
- **Forgetting `-run=^$` when benchmarking** mixes test output into the
  benchmark run and can skew timing if `TestMain` does expensive setup.
- **A fuzz corpus entry that fails must be triaged, not deleted** — the
  saved file under `testdata/fuzz/` is the minimal reproducing input;
  removing it just hides the bug from CI.
- **`t.Parallel()` inside a table-test loop variable capture** — before Go
  1.22, forgetting `c := c` inside the loop before calling `t.Run(...,
  func(t *testing.T) { t.Parallel(); use(c) })` meant every subtest saw the
  *last* row's value once parallel execution actually ran (Go 1.22+ scopes
  `for` loop variables per iteration, which fixes this by default).
- **Benchmarks that allocate setup data inside the timed loop** skew
  results — use `b.ResetTimer()` after one-time setup, or `b.StopTimer()` /
  `b.StartTimer()` to bracket the part you don't want measured.

## Cheat sheet

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

## Related lessons

- Profiling what a benchmark reveals: [Module 7](07-profiling-benchmarking.md).
- Race detection alongside tests: [Module 1](01-concurrency-patterns.md)'s
  `go run -race` exercise, and `go test -race ./...` for the same check
  across a whole test suite.
- The [Level 3 project](10-project-rest-api-sqlite.md) uses an in-memory
  fake `Store` in its tests via the strategy pattern from
  [Module 5](05-design-patterns.md).

## Exercise

Write `FuzzDivide` targeting `Divide(a, b int)`, seeding it with `f.Add(10,
2)`. Inside `f.Fuzz`, guard against `b == 0` (that's the known error case,
not a bug) and instead assert the invariant `Divide(a*b, b) == a` for
`b != 0` — pick inputs that avoid overflow if the fuzzer finds one, and note
in a comment why. Then add `BenchmarkDivide` and compare its `ns/op` against
`BenchmarkAdd`'s from this module — explain in one sentence why they're
close despite `Divide` doing a comparison and a branch that `Add` doesn't.
