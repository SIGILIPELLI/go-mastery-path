---
description: "Performance Optimization — Level 3, Module 7 introduced benchmarking and pprof. This module applies them to two real Go performance decisions …"
---

# 09 · Performance Optimization

[Level 3, Module 7](../level-3/07-profiling-benchmarking.md) introduced
benchmarking and pprof. This module applies them to two real Go
performance decisions — preallocating slices, and choosing between values
and pointers — and along the way hits a genuine benchmarking trap: the
compiler eliminating work you thought you were measuring.

## Preallocating a slice

```go
type Point struct {
	X, Y int
}

func sumNoPrealloc(n int) []Point {
	var pts []Point
	for i := 0; i < n; i++ {
		pts = append(pts, Point{i, i})
	}
	return pts
}

func sumPrealloc(n int) []Point {
	pts := make([]Point, 0, n)
	for i := 0; i < n; i++ {
		pts = append(pts, Point{i, i})
	}
	return pts
}

func BenchmarkNoPrealloc(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sumNoPrealloc(1000)
	}
}

func BenchmarkPrealloc(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sumPrealloc(1000)
	}
}
```

```console
$ go test -bench=. -run=^$ -benchmem .
BenchmarkNoPrealloc-8   	  285818	      4189 ns/op	   50416 B/op	      12 allocs/op
BenchmarkPrealloc-8     	 1327506	       905.3 ns/op	       0 B/op	       0 allocs/op
```

Read literally, `BenchmarkPrealloc` reporting **zero** allocations for a
function that allocates a 1000-element slice looks like an incredible win.
It isn't real — it's a benchmarking bug.

## The trap: dead code elimination

Both benchmarks above call `sumPrealloc(1000)`/`sumNoPrealloc(1000)` and
**discard the return value**. Go's compiler is allowed to notice that the
result of `sumPrealloc` is never used for anything observable and skip
doing the work at all in some cases — which is exactly what happened to
the "0 allocs" result. Fixing it means assigning the result to a
package-level variable (a "sink") so the compiler can't prove the call is
dead:

```go
var sink []Point

func BenchmarkPreallocSink(b *testing.B) {
	var r []Point
	for i := 0; i < b.N; i++ {
		r = sumPrealloc(1000)
	}
	sink = r
}

func BenchmarkNoPreallocSink(b *testing.B) {
	var r []Point
	for i := 0; i < b.N; i++ {
		r = sumNoPrealloc(1000)
	}
	sink = r
}
```

```console
$ go test -bench=Sink -run=^$ -benchmem .
BenchmarkPreallocSink-8     	  775501	      1544 ns/op	   16384 B/op	       1 allocs/op
BenchmarkNoPreallocSink-8   	  280153	      4325 ns/op	   50416 B/op	      12 allocs/op
```

*Now* the numbers reflect real work: preallocating cuts the allocation
count from 12 down to 1 (one single 16KB backing array instead of the
repeated doubling `append` does without a capacity hint) and is roughly
2.8x faster. The `sink` pattern — write the loop's result to a
package-level variable after the timed loop, not inside it — is the
standard defense against this exact class of misleading benchmark result.

## Values vs. pointers in a hot loop

```go
func sumPointers(n int) []*Point {
	pts := make([]*Point, 0, n)
	for i := 0; i < n; i++ {
		pts = append(pts, &Point{i, i})
	}
	return pts
}

func BenchmarkPointers(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sumPointers(1000)
	}
}
```

```console
$ go test -bench=Pointers -run=^$ -benchmem .
BenchmarkPointers-8   	  110227	     10768 ns/op	   16000 B/op	    1000 allocs/op
```

1000 allocations — one per `&Point{i, i}` — versus the 1 allocation the
preallocated value-slice version needed for its entire backing array.
Taking the address of a small struct inside a loop like this forces each
one onto the heap individually (it "escapes" because a pointer to it
survives the loop iteration), where a `[]Point` keeps all 1000 structs
contiguous in one allocation. For small, short-lived structs like `Point`,
storing values directly in a slice is almost always faster than storing
pointers — reach for `[]*T` when the elements are large (copying them
would be expensive) or need to be shared/mutated through multiple
references, not by default.

## Go-specific traps

- **Any benchmark whose result is never used anywhere observable risks
  partial or total dead-code elimination** — always sink the result to a
  package-level variable, exactly as shown, whenever you're not certain the
  compiler can't prove the call has no effect.
- **`go build -gcflags="-m"` shows escape analysis decisions** directly —
  running it against the `sumPointers` example prints a line noting
  `&Point{i, i} escapes to heap`, confirming the allocation source without
  guessing.
- **Preallocating with the wrong capacity is worse than not preallocating**
  — `make([]T, 0, wrongGuess)` that's too small still reallocates partway
  through; too large wastes memory. Preallocate from a known or measured
  size, not a guess.
- **`append` doubles capacity at small sizes but grows more conservatively
  at large ones** (roughly 1.25x past a few thousand elements in recent Go
  versions) — the exact growth factor is a runtime implementation detail,
  not something to hardcode logic around.
- **Micro-benchmark results don't automatically transfer to production
  behavior** — GC pressure, cache effects, and allocator behavior under
  real concurrent load can differ from an isolated `b.N` loop; treat a
  benchmark win as a hypothesis to confirm with production profiling
  ([Level 3, Module 7](../level-3/07-profiling-benchmarking.md)), not a
  final answer.

## How It Actually Works

`sync.Pool` reduces GC pressure by keeping a per-P (recall the GMP model, level-2/02)
free list of previously-used objects that a goroutine can `Get()` without touching
the shared heap allocator at all, and `Put()` back after use; because each P has its
own local pool, Get/Put usually need no locking, only falling back to a
cross-P steal when the local pool is empty. Objects sitting in a `sync.Pool` are not
protected from GC, though — the runtime clears pools at the start of most GC cycles
specifically so pooled memory doesn't become a permanent, invisible leak. Reducing
allocations matters because every heap allocation both costs allocator time and adds
one more object the GC's mark phase has to visit — cutting allocations in a hot loop
shrinks both the allocation-time cost and the GC's per-cycle mark work
simultaneously, which is why `-gcflags="-m"` escape-analysis output (level-1/07) is
the actual profiling tool for this class of optimization, not just wall-clock
benchmarking. CPU cache-line effects (false sharing between fields two goroutines
each write to) can dominate over algorithmic complexity at high concurrency, which
is why padding hot struct fields to separate 64-byte cache lines is a real, if
rarely needed, optimization in Go same as in C.


## Cheat sheet

| Optimization | How |
|---|---|
| Avoid slice regrowth | `make([]T, 0, n)` when `n` is known ahead of time |
| Avoid a misleading zero-allocation benchmark | Assign the result to a package-level `sink` var after the loop |
| See why something escapes to the heap | `go build -gcflags="-m" .` |
| Prefer values over pointers for small structs | `[]T` in hot paths, `[]*T` only for large/shared/mutable elements |
| Confirm a benchmark isn't dead code | Compare with and without a sink; a real workload's numbers shouldn't collapse to zero |

## Related lessons

- Benchmark and pprof mechanics this module builds directly on:
  [Level 3, Module 7](../level-3/07-profiling-benchmarking.md).
- The `strings.Builder` allocation-count case study from the same module —
  the identical "count allocations, don't guess" method applied here.

## 🔀 See this in another language

- [TypeScript — 07 · Performance Optimization](https://sigilipelli.github.io/typescript-mastery-path/level-4/07-performance-optimization/)
- [C — 03 · Performance Optimization & Profiling](https://sigilipelli.github.io/c-mastery-path/level-4/03-performance-profiling/)
- [C# — 05 · Performance Profiling & Optimization](https://sigilipelli.github.io/csharp-mastery-path/level-4/05-performance-profiling/)

## Exercise

Add a fourth variant, `sumPreallocPointers`, that preallocates
`make([]*Point, 0, n)` and confirm whether preallocating the *pointer*
slice's backing array closes any of the gap with `sumPointers` above (it
should reduce the slice-growth allocations but not the 1000 per-element
heap allocations). Then run `go build -gcflags="-m" .` on this file and
find the exact line reporting that `&Point{i, i}` escapes to the heap —
paste it as a comment above `sumPointers` explaining, in your own words,
why it escapes and the value version doesn't.
