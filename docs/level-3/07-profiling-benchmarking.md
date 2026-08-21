# 07 · Profiling & Benchmarking

[Module 6](06-testing-advanced.md) introduced `testing.B` benchmarks. This
module uses them to actually find a real performance bug — string
concatenation with `+=` versus `strings.Builder` — and then reaches for
`pprof` to see *why* the slow version is slow, not just that it is.

## The benchmark

```go
package m07

import (
	"strings"
	"testing"
)

func concatPlus(items []string) string {
	s := ""
	for _, it := range items {
		s += it
	}
	return s
}

func concatBuilder(items []string) string {
	var b strings.Builder
	for _, it := range items {
		b.WriteString(it)
	}
	return b.String()
}

var words = func() []string {
	w := make([]string, 200)
	for i := range w {
		w[i] = "hello"
	}
	return w
}()

func BenchmarkConcatPlus(b *testing.B) {
	for i := 0; i < b.N; i++ {
		concatPlus(words)
	}
}

func BenchmarkConcatBuilder(b *testing.B) {
	for i := 0; i < b.N; i++ {
		concatBuilder(words)
	}
}
```

```console
$ go test -bench=. -run=^$ -benchmem .
goos: darwin
goarch: arm64
pkg: m07
cpu: Apple M1
BenchmarkConcatPlus-8      	   82281	     12407 ns/op	  106440 B/op	     199 allocs/op
BenchmarkConcatBuilder-8   	 1311042	       914.6 ns/op	    3320 B/op	       9 allocs/op
PASS
ok  	m07	3.895s
```

`-benchmem` adds the last two columns — bytes and allocations per
operation — and they tell the real story: `concatPlus` does **199
allocations** joining 200 strings because Go strings are immutable, so
`s += it` allocates a brand-new string every iteration and copies
everything built so far into it. `concatBuilder` allocates 9 times total
(the builder's internal buffer growing in doubling steps) and is over 13x
faster as a direct result. The byte counts confirm it: 106KB moved around
for `concatPlus` versus 3.3KB for the builder.

## Reading a CPU profile

Benchmarks can dump a pprof-format profile straight from `go test`, no
extra instrumentation needed:

```console
$ go test -bench=BenchmarkConcatPlus -run=^$ -cpuprofile=cpu.prof .
goos: darwin
goarch: arm64
pkg: m07
cpu: Apple M1
BenchmarkConcatPlus-8   	   65857	     16513 ns/op
PASS
ok  	m07	1.928s

$ go tool pprof -top -nodecount=8 cpu.prof
File: m07.test
Type: cpu
Duration: 1.41s, Total samples = 1760ms (125.23%)
Showing nodes accounting for 1610ms, 91.48% of 1760ms total
      flat  flat%   sum%        cum   cum%
     340ms 19.32% 19.32%      340ms 19.32%  runtime.pthread_cond_wait
     340ms 19.32% 38.64%      340ms 19.32%  runtime.usleep
     280ms 15.91% 54.55%      280ms 15.91%  runtime.pthread_cond_signal
     260ms 14.77% 69.32%      260ms 14.77%  runtime.kevent
     250ms 14.20% 83.52%      250ms 14.20%  runtime.madvise
      80ms  4.55% 88.07%       80ms  4.55%  runtime.pthread_kill
      40ms  2.27% 90.34%       40ms  2.27%  runtime.pthread_kill
      20ms  1.14% 91.48%       20ms  1.14%  runtime.(*pallocBits).summarize
```

Every single hot frame here is the Go runtime's memory allocator and
scheduler — `madvise` (returning/reserving OS memory for the heap),
`pthread_cond_wait`/`kevent` (goroutine parking and GC coordination) — not
application code. That absence of application frames *is* the diagnosis:
the benchmark is so allocation-heavy that essentially all measured CPU time
went to garbage collection and heap bookkeeping instead of doing `+=`. The
fix is the one already shown above — stop allocating, and this profile's
shape disappears along with the slowdown.

## Go-specific traps

- **Benchmarking without `-benchmem`** hides the exact signal (allocation
  count) that usually explains a slowdown in Go — always pass it when
  comparing two implementations.
- **`go tool pprof` needs a profile written to disk** — `-cpuprofile=`,
  `-memprofile=`, or, for running services, an imported `net/http/pprof`
  serving profiles over HTTP. Forgetting to import it (blank import,
  `_ "net/http/pprof"`) means `/debug/pprof/` 404s even with the mux wired
  up correctly otherwise.
- **Comparing two benchmark runs by eyeballing `ns/op`** is noisy on a busy
  machine — use `benchstat` (`go install
  golang.org/x/perf/cmd/benchstat@latest`) on multiple `-count=10` runs of
  each version for a statistically defensible comparison.
- **String concatenation isn't always the bottleneck it is here** — for two
  or three short strings, `+=` is fine and `strings.Builder` is
  over-engineering; profile before optimizing, don't guess.
- **`b.N` in a benchmark can vary run to run** — never write benchmark code
  that assumes a specific `b.N`, and never benchmark something with
  meaningful side effects that accumulate (e.g. appending to a package-level
  slice) without resetting them each iteration.

## Cheat sheet

| Task | Command |
|---|---|
| Benchmark with allocation stats | `go test -bench=. -run=^$ -benchmem` |
| Write a CPU profile | `go test -bench=X -run=^$ -cpuprofile=cpu.prof` |
| Write a memory profile | `go test -bench=X -run=^$ -memprofile=mem.prof` |
| Inspect a profile, top functions | `go tool pprof -top -nodecount=N cpu.prof` |
| Interactive profile browser | `go tool pprof cpu.prof` then `top`, `list <func>`, `web` |
| Live profiling for a running server | `import _ "net/http/pprof"`, hit `/debug/pprof/profile` |
| Statistically compare benchmark runs | `benchstat old.txt new.txt` |

## Related lessons

- Benchmark mechanics (`testing.B`, `b.N`): [Module 6](06-testing-advanced.md).
- Concurrency patterns whose cost shows up in a profile the same way:
  [Module 1](01-concurrency-patterns.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) wires
  `net/http/pprof` into its server for live profiling under load.

## Exercise

Add a third variant, `concatSlice`, that appends each item to a `[]byte`
via `append` and converts once with `string(buf)` at the end. Benchmark all
three with `-benchmem` and rank them by `ns/op` and `allocs/op`. Then take a
memory profile (`-memprofile=mem.prof`) of `concatPlus` and run `go tool
pprof -top -nodecount=5 mem.prof` — identify which function's allocations
dominate and confirm it matches your expectation from reading the code.
