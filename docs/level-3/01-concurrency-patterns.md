# 01 · Concurrency Patterns

[Level 2](../level-2/02-goroutines-channels.md) covered goroutines, channels
and `select` in isolation. Real programs combine them into a handful of
recurring shapes — worker pools, pipelines, fan-out/fan-in, and one-time
initialization. Knowing these patterns by name means you reach for a proven
shape instead of improvising synchronization from scratch.

## Worker pools: bounded concurrency

Launching one goroutine per unit of work is fine for a handful of items, but
for thousands of jobs — each hitting a database or a rate-limited API — you
want a *fixed* number of workers pulling from a shared queue.

```go
package main

import (
	"fmt"
	"sync"
)

type job struct{ n int }
type result struct{ n, square int }

func worker(id int, jobs <-chan job, results chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()
	for j := range jobs { // exits automatically when jobs is closed and drained
		results <- result{n: j.n, square: j.n * j.n}
	}
}

func main() {
	const numWorkers = 3
	jobs := make(chan job, 10)
	results := make(chan result, 10)

	var wg sync.WaitGroup
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	for n := 1; n <= 8; n++ {
		jobs <- job{n: n}
	}
	close(jobs) // signal: no more jobs -- workers finish their current item and exit

	go func() {
		wg.Wait()      // wait for every worker to stop sending
		close(results) // only safe to close AFTER all senders are done
	}()

	sum := 0
	for r := range results {
		sum += r.square
	}
	fmt.Println("sum of squares:", sum)
	// Output: sum of squares: 204
}
```

The number of workers is independent of the number of jobs — three workers
happily process eight jobs, each picking up a new one the instant it finishes.
Bump `numWorkers` to control how much concurrent load you put on a downstream
resource like a database connection pool.

**The trap**: closing `results` *before* all workers finish sending panics
with "send on closed channel". That is why the close happens in its own
goroutine gated by `wg.Wait()` — never close a channel from a goroutine that
might still write to it.

## Fan-out, fan-in

*Fan-out* means multiple goroutines reading from one channel to parallelize
work; *fan-in* means multiple channels being merged into one.

```go
package main

import (
	"fmt"
	"sync"
)

func generator(n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 1; i <= n; i++ {
			out <- i
		}
	}()
	return out
}

// square is a pipeline stage. Running several of these against the SAME
// input channel is what makes this "fan-out".
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

// merge fans multiple channels back into one -- "fan-in".
func merge(cs ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	wg.Add(len(cs))
	for _, c := range cs {
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(c)
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

func main() {
	in := generator(10)

	// three squarers pulling from the same channel -- fan-out
	c1 := square(in)
	c2 := square(in)
	c3 := square(in)

	sum := 0
	for v := range merge(c1, c2, c3) { // fan-in
		sum += v
	}
	fmt.Println("sum of squares 1..10:", sum)
	// Output: sum of squares 1..10: 385
}
```

Because three goroutines pull from `in` concurrently, work items get
distributed across them in whatever order the scheduler grants — the *sum* is
still deterministic, but do not assume any particular item lands on any
particular squarer.

## Pipelines

A pipeline chains stages together, each stage its own goroutine connected by
a channel — the same shape `io.Reader`/`io.Writer` chains achieve for data,
but for concurrent computation:

```go
package main

import "fmt"

func generate(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

func double(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * 2
		}
	}()
	return out
}

func addOne(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n + 1
		}
	}()
	return out
}

func main() {
	// Each stage runs concurrently -- addOne can start on item 1 while
	// double is still working on item 3.
	pipeline := addOne(double(generate(1, 2, 3, 4, 5)))
	for v := range pipeline {
		fmt.Print(v, " ")
	}
	fmt.Println()
	// Output: 3 5 7 9 11
}
```

Each function takes a channel and returns a channel, so stages compose by
nesting calls. Adding a stage is one more wrapping function — no change to
the ones on either side.

## `sync.Once`: exactly-once initialization

Lazy, thread-safe initialization that must run exactly once no matter how
many goroutines call it concurrently:

```go
package main

import (
	"fmt"
	"sync"
)

var (
	once   sync.Once
	config string
)

func loadConfig() {
	fmt.Println("loading config from disk...") // must print exactly once
	config = "db=prod;timeout=30s"
}

func getConfig() string {
	once.Do(loadConfig) // subsequent calls are no-ops, even from other goroutines
	return config
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_ = getConfig()
		}()
	}
	wg.Wait()
	fmt.Println("final config:", getConfig())
	// Output:
	// loading config from disk...
	// final config: db=prod;timeout=30s
}
```

Five goroutines call `getConfig()` concurrently but "loading config from
disk..." prints exactly once. Compare this to a `sync.Mutex` plus a boolean
flag — `sync.Once` is the idiomatic, race-free version of that pattern and
handles the flag itself.

## Rate limiting with `time.Tick`

Throttle how fast you process a channel of work by gating on a ticker:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	requests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		requests <- i
	}
	close(requests)

	limiter := time.Tick(50 * time.Millisecond) // one tick every 50ms
	start := time.Now()
	for req := range requests {
		<-limiter // block until the next tick arrives
		fmt.Printf("request %d handled at %v\n", req, time.Since(start).Round(10*time.Millisecond))
	}
	// Output:
	// request 1 handled at 50ms
	// request 2 handled at 100ms
	// request 3 handled at 150ms
	// request 4 handled at 200ms
	// request 5 handled at 250ms
}
```

`time.Tick` leaks its underlying ticker forever (it has no way to stop it) —
fine for a long-lived limiter in `main`, but in a function that returns
repeatedly, use `time.NewTicker` and `defer ticker.Stop()` instead.

## Cancelling a worker pool early

None of the patterns above stop workers if the caller loses interest halfway
through. Real pools need an exit signal — [Module 4](04-context-package.md)
covers `context.Context` in depth, but the shape to recognize now:

```go
select {
case jobs <- j:
	// sent
case <-ctx.Done():
	return ctx.Err() // caller cancelled; stop trying to send
}
```

Every blocking channel operation in a long-running goroutine should have a
`ctx.Done()` escape hatch next to it, or that goroutine can outlive the work
it was meant to do — a goroutine leak, as introduced in
[Level 2, Module 2](../level-2/02-goroutines-channels.md).

## Cheat sheet

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

## Related lessons

- Goroutines, channels and `select` fundamentals:
  [Level 2, Module 2](../level-2/02-goroutines-channels.md).
- Cancellation and deadlines with `context`: [Module 4](04-context-package.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) uses a worker-pool
  shaped background job to write logs without blocking requests.

## Exercise

Build a pipeline that reads integers 1–20 from a generator, fans them out to
four worker goroutines that each check primality (a simple trial-division
`isPrime` is fine), and fans the results back in. Collect the primes into a
slice, sort it, and print it — it should be `[2 3 5 7 11 13 17 19]`. Then add
a `sync.Once`-guarded "starting worker pool..." log message that only prints
once no matter how many workers start. Finally, run the whole program with
`go run -race .` and confirm it is clean.
