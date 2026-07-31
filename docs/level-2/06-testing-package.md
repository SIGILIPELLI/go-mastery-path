# 06 · Testing with the testing Package

## Testing is built in

Go ships with a test runner and a testing package in the standard library.
There is no framework to choose, no assertion library to install, no config
file. Write a function called `TestSomething` in a `_test.go` file and
`go test` finds and runs it.

The conventions the toolchain enforces:

- Test files end in `_test.go` and live **beside** the code they test.
- Test functions are `func TestXxx(t *testing.T)` — capital letter after
  `Test`.
- Files ending in `_test.go` are excluded from normal builds.

## Your first test

Given `math.go`:

```go
// mathutil/math.go
package mathutil

import "errors"

var ErrDivideByZero = errors.New("division by zero")

func Add(a, b int) int { return a + b }

func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, ErrDivideByZero
	}
	return a / b, nil
}
```

The test file sits next to it:

```go
// mathutil/math_test.go
package mathutil

import (
	"errors"
	"testing"
)

func TestAdd(t *testing.T) {
	got := Add(2, 3)
	want := 5
	if got != want {
		// t.Errorf reports a failure but keeps the test running.
		t.Errorf("Add(2, 3) = %d; want %d", got, want)
	}
}

func TestDivideByZero(t *testing.T) {
	_, err := Divide(10, 0)
	if !errors.Is(err, ErrDivideByZero) {
		t.Fatalf("Divide(10, 0) error = %v; want %v", err, ErrDivideByZero)
	}
}
```

Run it:

```bash
go test ./...           # every package in the module
go test -v ./mathutil   # verbose: one line per test
go test -run TestAdd    # only tests matching this regexp
```

```text
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestDivideByZero
--- PASS: TestDivideByZero (0.00s)
PASS
ok      example/mathutil    0.002s
```

Go has no `assertEqual`. You compare with plain `if` and report with
`t.Errorf`. The universal message format is
`"Func(input) = got; want expected"` — follow it and failures read clearly
without opening the source.

## `Error` vs `Fatal` vs `Log`

| Method | Marks failure | Stops this test | Use when |
|--------|---------------|-----------------|----------|
| `t.Log` / `t.Logf` | no | no | diagnostics (shown with `-v`, or on failure) |
| `t.Error` / `t.Errorf` | yes | no | a bad value, but the test can keep checking |
| `t.Fatal` / `t.Fatalf` | yes | yes | setup failed; continuing would panic |
| `t.Skip` / `t.Skipf` | no | yes | precondition absent (no network, no fixture) |
| `t.Helper()` | — | — | mark a function as a helper so line numbers point at the caller |

Rule of thumb: `Fatal` when the next line would crash (a nil result, a failed
`os.Open`), `Error` otherwise so one run reports every problem.

## Table-driven tests: the Go idiom

Rather than one function per case, put the cases in a slice and loop. This is
overwhelmingly the dominant style in Go codebases, including the standard
library itself.

```go
package mathutil

import (
	"errors"
	"math"
	"testing"
)

func TestDivide(t *testing.T) {
	tests := []struct {
		name    string
		a, b    float64
		want    float64
		wantErr error
	}{
		{name: "simple", a: 10, b: 2, want: 5},
		{name: "negative", a: -9, b: 3, want: -3},
		{name: "fraction", a: 1, b: 4, want: 0.25},
		{name: "divide by zero", a: 1, b: 0, wantErr: ErrDivideByZero},
	}

	for _, tt := range tests {
		// t.Run creates a named subtest -- failures identify themselves.
		t.Run(tt.name, func(t *testing.T) {
			got, err := Divide(tt.a, tt.b)

			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("Divide(%v, %v) error = %v; want %v", tt.a, tt.b, err, tt.wantErr)
			}
			if tt.wantErr != nil {
				return // error case verified; no value to check
			}
			if math.Abs(got-tt.want) > 1e-9 { // never compare floats with ==
				t.Errorf("Divide(%v, %v) = %v; want %v", tt.a, tt.b, got, tt.want)
			}
		})
	}
}
```

Output identifies the exact case:

```text
--- FAIL: TestDivide (0.00s)
    --- FAIL: TestDivide/fraction (0.00s)
        math_test.go:31: Divide(1, 4) = 0.2; want 0.25
```

Adding a new case is one line. `go test -run 'TestDivide/fraction'` runs just
that subtest.

## Comparing composite values

`==` does not work on slices or maps. Use `reflect.DeepEqual`, or
`slices.Equal` / `maps.Equal` (Go 1.21+) which are faster and type-safe:

```go
package mathutil

import (
	"reflect"
	"slices"
	"testing"
)

func TestEvens(t *testing.T) {
	got := Evens([]int{1, 2, 3, 4, 5, 6})
	want := []int{2, 4, 6}

	if !slices.Equal(got, want) { // preferred for slices
		t.Errorf("Evens() = %v; want %v", got, want)
	}

	gotMap := Tally([]string{"a", "b", "a"})
	wantMap := map[string]int{"a": 2, "b": 1}
	if !reflect.DeepEqual(gotMap, wantMap) { // works for any nested structure
		t.Errorf("Tally() = %v; want %v", gotMap, wantMap)
	}
}
```

A trap worth knowing: `reflect.DeepEqual([]int{}, []int(nil))` is `false`. An
empty slice and a nil slice are different to `DeepEqual` even though both have
length zero — decide which your function should return and test for it.

## Setup, cleanup and temp directories

`t.Cleanup` registers teardown that runs when the test finishes, however it
finishes. `t.TempDir()` creates a directory that is removed automatically:

```go
package store

import (
	"os"
	"path/filepath"
	"testing"
)

func TestSaveAndLoad(t *testing.T) {
	dir := t.TempDir() // auto-deleted after the test
	path := filepath.Join(dir, "data.json")

	s := New(path)
	if err := s.Save([]string{"one", "two"}); err != nil {
		t.Fatalf("Save() error = %v", err)
	}

	got, err := s.Load()
	if err != nil {
		t.Fatalf("Load() error = %v", err)
	}
	if len(got) != 2 {
		t.Errorf("Load() returned %d items; want 2", len(got))
	}

	// Example of manual cleanup when you need it:
	f, _ := os.Create(filepath.Join(dir, "scratch"))
	t.Cleanup(func() { f.Close() })
}
```

`TestMain(m *testing.M)` handles package-wide setup: do the work, call
`os.Exit(m.Run())`, and everything in the package shares it.

## Testing with interfaces instead of mocks

Go rarely needs a mocking library. Depend on a small interface
([Module 1](01-interfaces.md)) and pass a fake in the test:

```go
package notify

import (
	"errors"
	"testing"
)

type Sender interface {
	Send(to, body string) error
}

// Notifier depends on the interface, not a concrete SMTP client.
type Notifier struct{ S Sender }

func (n Notifier) Alert(user string) error { return n.S.Send(user, "alert!") }

// fakeSender records calls instead of sending anything.
type fakeSender struct {
	calls []string
	err   error
}

func (f *fakeSender) Send(to, body string) error {
	f.calls = append(f.calls, to+": "+body)
	return f.err
}

func TestAlert(t *testing.T) {
	fake := &fakeSender{}
	n := Notifier{S: fake}

	if err := n.Alert("ada@example.com"); err != nil {
		t.Fatalf("Alert() error = %v", err)
	}
	if len(fake.calls) != 1 {
		t.Fatalf("got %d sends; want 1", len(fake.calls))
	}
	if fake.calls[0] != "ada@example.com: alert!" {
		t.Errorf("unexpected call: %q", fake.calls[0])
	}

	// Failure path.
	fake.err = errors.New("smtp down")
	if err := n.Alert("x@example.com"); err == nil {
		t.Error("expected an error when the sender fails")
	}
}
```

## Coverage, parallelism and the race detector

```bash
go test -cover ./...                       # coverage percentage per package
go test -coverprofile=c.out ./... \
  && go tool cover -html=c.out             # line-by-line coverage in a browser
go test -race ./...                        # detect data races (see Module 2)
go test -count=1 ./...                     # bypass the test result cache
go test -v -run 'TestDivide/simple'        # one subtest
go test -bench=. ./...                     # run benchmarks too
```

Calling `t.Parallel()` at the top of a test signals it can run alongside other
parallel tests, cutting wall-clock time for I/O-heavy suites. Chase useful
coverage, not a number — 100% coverage of getters proves nothing, while one
good table test of your parsing logic prevents real bugs.

## Example functions double as documentation

A function named `ExampleXxx` with an `// Output:` comment is compiled, run,
and its output compared. It appears in `go doc` output too, so your docs
cannot drift from reality:

```go
func ExampleAdd() {
	fmt.Println(Add(2, 3))
	// Output: 5
}
```

If `Add` ever stops returning 5, the documentation fails the build.

## Cheat sheet

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

## Related lessons

- Fakes rely on small interfaces: [Module 1](01-interfaces.md).
- Asserting error identity with `errors.Is`:
  [Module 4](04-custom-errors-wrapping.md).
- Benchmarks, fuzzing and golden files:
  [Level 3, Module 6](../level-3/06-testing-advanced.md).
- Packages and module layout:
  [Level 1, Module 9](../level-1/09-packages-modules.md).

## Exercise

Take the `NextID` function from the
[Level 1 to-do project](../level-1/10-project-todo-cli.md) and write a
table-driven test for it covering: an empty slice, a slice with sequential
IDs, a slice with gaps, and a slice with unsorted IDs. Use `t.Run` with a
descriptive name per case. Then add a `TestStoreRoundTrip` that uses
`t.TempDir()` to save tasks to a temp file, load them back, and verify with
`reflect.DeepEqual` that nothing changed. Run `go test -v -cover ./...` and
note which branches are still uncovered.
