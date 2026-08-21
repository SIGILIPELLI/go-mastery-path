# 01 · Advanced Concurrency

[Level 3, Module 1](../level-3/01-concurrency-patterns.md) covered worker
pools, pipelines, and fan-in/fan-out. This module goes further into
production-grade concurrency: coordinating a group of goroutines that
should all fail together via `errgroup`, `sync.Map` for concurrent-safe
maps without a hand-rolled mutex, and the atomic operations that back both.

## `errgroup`: cancel the group when one goroutine fails

The standard library's `sync.WaitGroup` has no way to propagate an error or
cancel siblings when one goroutine fails. `golang.org/x/sync/errgroup`
adds exactly that on top of `context.Context`:

```go
package main

import (
	"context"
	"fmt"
	"sync/atomic"
	"time"

	"golang.org/x/sync/errgroup"
)

func fetch(ctx context.Context, id int) (int, error) {
	select {
	case <-time.After(time.Duration(id) * 10 * time.Millisecond):
		if id == 4 {
			return 0, fmt.Errorf("fetch %d failed", id)
		}
		return id * id, nil
	case <-ctx.Done():
		return 0, ctx.Err()
	}
}

func main() {
	g, ctx := errgroup.WithContext(context.Background())
	var total int64
	for i := 1; i <= 5; i++ {
		i := i
		g.Go(func() error {
			v, err := fetch(ctx, i)
			if err != nil {
				return err
			}
			atomic.AddInt64(&total, int64(v))
			return nil
		})
	}
	err := g.Wait()
	fmt.Println("total so far:", atomic.LoadInt64(&total))
	fmt.Println("err:", err)
}
```

```console
$ go run .
total so far: 14
err: fetch 4 failed
```

`errgroup.WithContext` returns a `ctx` that gets cancelled the instant *any*
`g.Go` function returns a non-nil error — that's why goroutines 1, 2, 3
(which finish before goroutine 4's error at 40ms) contribute
`1+4+9=14` to `total`, while goroutine 5 (scheduled to finish at 50ms) sees
`ctx.Done()` fire first and never adds `25`. `g.Wait()` blocks until every
goroutine returns, then gives back the *first* error reported — the exact
"stop the whole group, report why" behavior a `WaitGroup` alone can't
express.

**The trap**: mutating `total` with a plain `total += v` from multiple
`g.Go` goroutines is a data race — `atomic.AddInt64` (or a mutex) is not
optional just because `errgroup` is handling the coordination logic.

## `sync.Map`: concurrent-safe map without a mutex

For maps written and read concurrently from many goroutines with mostly
disjoint keys, `sync.Map` avoids a global mutex bottleneck:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		i := i
		wg.Add(1)
		go func() {
			defer wg.Done()
			m.Store(i, i*i)
		}()
	}
	wg.Wait()

	sum := 0
	m.Range(func(k, v any) bool {
		sum += v.(int)
		return true
	})
	fmt.Println("sum of squares via sync.Map:", sum)

	v, ok := m.Load(3)
	fmt.Println("load key 3:", v, ok)
}
```

```console
$ go run .
sum of squares via sync.Map: 30
load key 3: 9 true
```

`m.Range` visits keys in no particular order (like a regular map) and its
callback returning `false` stops iteration early — useful for a "find the
first match" search without building a slice first. Note `Load` returns
`any`, so reading a value back out requires a type assertion (`v.(int)`
above); that lost type safety is the real cost of `sync.Map`, which is why
the standard library doc recommends a plain `map` plus a `sync.RWMutex`
unless you specifically have the disjoint-keys or write-once-read-many
access pattern `sync.Map` is optimized for.

## Go-specific traps

- **`sync.Map` is *not* a drop-in replacement for `map[K]V]` guarded by a
  mutex in the general case** — for a map with heavy read/write contention
  on the *same* keys, a `sync.RWMutex`-guarded map often benchmarks faster;
  measure before reaching for `sync.Map` by default.
- **`errgroup.Group.SetLimit(n)`** (added later to the package) bounds how
  many goroutines run concurrently — without it, `g.Go` inside a loop over
  a large slice launches all of them at once, which for outbound HTTP calls
  can trip rate limits or exhaust file descriptors.
- **`ctx` from `errgroup.WithContext` is cancelled on the *first* error,
  not the last** — goroutines that don't check `ctx.Done()` (like a
  `time.Sleep` instead of `select` against it) keep running to completion
  anyway, wasting the work `errgroup` was meant to cut short.
- **`atomic.AddInt64` requires 64-bit alignment on 32-bit platforms** — a
  struct field aligned incorrectly can panic at runtime; putting the
  atomically-accessed field first in a struct avoids this, and
  `atomic.Int64` (the typed wrapper added in Go 1.19) sidesteps the issue
  entirely by handling alignment itself.
- **`g.Wait()` only returns after every goroutine has returned**, even the
  ones that saw `ctx.Done()` and returned early — it's not "stop as soon as
  the first error appears," it's "stop *launching new work*, still wait for
  in-flight work to unwind."

## Cheat sheet

| Need | Tool |
|---|---|
| Run N goroutines, fail-fast, propagate first error | `errgroup.WithContext` + `g.Go(func() error {...})` |
| Bound concurrent goroutines in a group | `g.SetLimit(n)` |
| Concurrent-safe counter | `atomic.Int64` (typed) or `atomic.AddInt64(&x, ...)` |
| Concurrent-safe map, disjoint keys | `sync.Map` — `Store`, `Load`, `Range`, `Delete` |
| Concurrent-safe map, general case | `map[K]V` + `sync.RWMutex` |
| Wait for goroutines, no error propagation needed | `sync.WaitGroup` ([Level 3](../level-3/01-concurrency-patterns.md)) |

## Related lessons

- Worker pools, pipelines, fan-in/out fundamentals:
  [Level 3, Module 1](../level-3/01-concurrency-patterns.md).
- `context.Context` cancellation mechanics that `errgroup` builds on:
  [Level 3, Module 4](../level-3/04-context-package.md).
- Microservice call fan-out that typically uses `errgroup`:
  [Module 2](02-microservices-patterns.md).

## Exercise

Rewrite the `errgroup` example so `fetch` takes a `[]int` of IDs and
`g.SetLimit(2)` caps concurrency to two in-flight fetches at a time. Add a
`time.Since` measurement around `g.Wait()` and confirm the total wall time
is roughly what you'd expect from two workers processing five staggered
delays instead of five running fully in parallel. Then add a `sync.Map` to
record each fetch's `(id, result)` pair as it completes, and print the map
sorted by key after `g.Wait()` returns.
