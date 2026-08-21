# 05 · Testing at Scale & CI

[Level 3, Module 6](../level-3/06-testing-advanced.md) covered fuzzing and
benchmarks for a single package. Scaling that to a real codebase means
tracking coverage, separating fast unit tests from slow integration tests
so CI stays useful, and wiring both into a pipeline that runs on every
push. This module builds a real `go.mod`, generates a real coverage
report, and ends with a GitHub Actions workflow that does the same.

## Measuring coverage

```go
package calc

func Add(a, b int) int { return a + b }
func Sub(a, b int) int { return a - b }
```

```go
package calc

import "testing"

func TestAdd(t *testing.T) {
	if Add(2, 3) != 5 {
		t.Fatal("bad add")
	}
}

func TestSub(t *testing.T) {
	if Sub(5, 3) != 2 {
		t.Fatal("bad sub")
	}
}
```

```console
$ go test -coverprofile=cover.out ./...
ok  	l4m05	0.619s	coverage: 100.0% of statements

$ go tool cover -func=cover.out
l4m05/calc.go:3:	Add		100.0%
l4m05/calc.go:4:	Sub		100.0%
total:			(statements)	100.0%
```

`-coverprofile` writes per-statement coverage data to a file; `go tool
cover -func` turns it into a per-function summary, and `go tool cover
-html=cover.out -o cover.html` renders a browsable, line-by-line view —
genuinely useful for spotting an entire error-handling branch nobody
exercises, not just a percentage to chase. 100% coverage on two one-line
functions is a toy result — real coverage targets (commonly 70-80% for a
service, higher for a pure library) are a floor to catch untested code
paths, not a proxy for "the tests are good."

## Separating fast unit tests from slow integration tests

A build tag gates a test file out of the default `go test ./...` run:

```go
//go:build integration

package calc

import "testing"

func TestAddAgainstRealService(t *testing.T) {
	// Placeholder: real integration tests hit a live dependency
	// (a database, another service) and are gated behind this build tag
	// so `go test ./...` stays fast by default.
	if Add(10, 5) != 15 {
		t.Fatal("bad add")
	}
}
```

```console
$ go test ./... -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestSub
--- PASS: TestSub (0.00s)
PASS
ok  	l4m05	0.518s

$ go test -tags=integration ./... -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestSub
--- PASS: TestSub (0.00s)
=== RUN   TestAddAgainstRealService
--- PASS: TestAddAgainstRealService (0.00s)
PASS
ok  	l4m05	0.530s
```

Without `-tags=integration`, `TestAddAgainstRealService` doesn't even
compile into the test binary — it's not skipped at runtime, it's absent
entirely. This is the standard way to keep `go test ./...` fast enough to
run on every save while a separate, slower CI job runs the tagged
integration suite against real dependencies (a database container, a
staging service).

## A real CI pipeline

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.23"
          cache: true

      - name: Vet
        run: go vet ./...

      - name: Unit tests with race detector and coverage
        run: go test -race -coverprofile=cover.out ./...

      - name: Enforce coverage floor
        run: |
          pct=$(go tool cover -func=cover.out | tail -1 | awk '{print $3}' | tr -d '%')
          echo "coverage: ${pct}%"
          awk -v p="$pct" 'BEGIN { exit !(p+0 >= 70) }'

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6

  integration:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.23"
      - name: Integration tests
        run: go test -tags=integration ./...
```

`-race` in the unit-test step catches data races
([Level 3, Module 1](../level-3/01-concurrency-patterns.md)) on every push,
not just when someone remembers to run it locally. The coverage-floor step
turns "coverage: 62%" from a number nobody looks at into a build that
actually fails when it drops. Splitting `integration` into its own job with
`needs: test` means slow, potentially flaky integration tests only run
after the fast unit suite passes, and a failure there doesn't block merges
on unrelated unit-test problems being fixed first.

## Go-specific traps

- **`go test -race` is significantly slower and more memory-hungry** —
  running it on every local save is often impractical, but it belongs in
  CI on every push precisely because CI machines can absorb the cost and
  humans forget to run it manually.
- **A coverage percentage says nothing about assertion quality** — a test
  that calls a function and asserts nothing about the result still counts
  as "covered." Coverage tooling answers "was this code executed," never
  "was this code correctly checked."
- **Build tags must be the very first thing in the file**, with a blank
  line separating `//go:build ...` from the `package` clause — get the
  blank line wrong and the tag silently fails to apply, and the file
  compiles unconditionally.
- **`go vet` is not a linter** — it catches real bugs (`Printf` format
  mismatches, unreachable code) but not style issues; `golangci-lint`
  wraps `go vet` alongside dozens of style/bug-pattern linters and is what
  most real Go CI pipelines run instead.
- **Caching `go build`/`go test` artifacts incorrectly across CI runs** can
  hide a failure that would show up on a clean checkout — `actions/setup-go`'s
  built-in `cache: true` handles the module cache correctly; don't hand-roll
  a cache key that also captures build output as if it were still valid
  after a source change.

## Cheat sheet

| Task | Command |
|---|---|
| Run tests with coverage | `go test -coverprofile=cover.out ./...` |
| Per-function coverage report | `go tool cover -func=cover.out` |
| Browsable HTML coverage | `go tool cover -html=cover.out -o cover.html` |
| Gate a test behind a build tag | `//go:build integration` (first line, blank line after) |
| Run tagged tests | `go test -tags=integration ./...` |
| Race detector in CI | `go test -race ./...` |
| Lint beyond `go vet` | `golangci-lint run` (`golangci-lint-action` in Actions) |

## Related lessons

- Table tests, subtests, fuzzing, and benchmarks this module scales up:
  [Level 3, Module 6](../level-3/06-testing-advanced.md).
- The race detector applied to concurrency patterns directly:
  [Level 3, Module 1](../level-3/01-concurrency-patterns.md).
- Containerizing the service these tests protect: [Module 6](06-deployment-docker.md).

## Exercise

Add a `Makefile` (or a `justfile`) with `test`, `test-integration`,
`cover`, and `lint` targets that wrap the commands above. Then extend the
CI workflow with a matrix (`strategy: matrix: go-version: ["1.22", "1.23"]`)
so unit tests run against two Go versions in parallel jobs, and confirm in
the Actions UI (or by reading the generated workflow file carefully) that
both versions' jobs are independent — a failure in one doesn't cancel the
other.
