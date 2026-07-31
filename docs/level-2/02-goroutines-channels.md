# 02 · Goroutines & Channels Basics

## Goroutines: concurrency that costs almost nothing

A goroutine is a function running independently alongside others, scheduled by
the Go runtime rather than the operating system. Starting one costs a few
kilobytes of stack — you can run hundreds of thousands of them, which is why
Go code reaches for concurrency far more casually than most languages.

Put `go` in front of any function call and it runs concurrently:

```go
package main

import (
	"fmt"
	"time"
)

func worker(name string) {
	for i := 1; i <= 3; i++ {
		fmt.Println(name, "step", i)
		time.Sleep(10 * time.Millisecond)
	}
}

func main() {
	go worker("A")
	go worker("B")

	// main is itself a goroutine -- when it returns, the program exits
	// and any still-running goroutines are killed instantly.
	time.Sleep(100 * time.Millisecond)
	fmt.Println("main done")
}
```

Two things to internalise immediately:

1. **`main` returning kills everything.** There is no "wait for children" by
   default. The `time.Sleep` above is a placeholder for real synchronisation.
2. **Interleaving order is undefined.** `A step 1` may print before or after
   `B step 1`. Never write code that depends on scheduling order.

## `sync.WaitGroup`: waiting properly

`time.Sleep` is never the right answer in real code. A `WaitGroup` is a
counter: `Add` before launching, `Done` when finished, `Wait` blocks until the
counter reaches zero.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1) // increment BEFORE the go statement
		go func(id int) {
			defer wg.Done() // deferred, so a panic still decrements
			fmt.Println("worker", id, "finished")
		}(i)
	}

	wg.Wait() // blocks until all three call Done
	fmt.Println("all workers finished")
}
```

Note `go func(id int){...}(i)` — the loop variable is passed as an argument.
In Go versions before 1.22 every goroutine shared one `i` variable and would
often print the same value three times. Go 1.22 made loop variables
per-iteration, fixing this, but passing explicitly is still the clearest form
and works on every version.

## Channels: typed pipes between goroutines

A channel carries values of one type between goroutines *and* synchronises
them at the same time. The Go proverb: *don't communicate by sharing memory;
share memory by communicating.*

```go
package main

import "fmt"

func main() {
	ch := make(chan string) // unbuffered channel of strings

	go func() {
		ch <- "hello from the goroutine" // send: blocks until someone receives
	}()

	msg := <-ch // receive: blocks until someone sends
	fmt.Println(msg)
	// Output: hello from the goroutine
}
```

An **unbuffered** channel is a rendezvous point: the sender blocks until a
receiver is ready and vice versa. That blocking *is* the synchronisation — no
mutex required.

## Buffered channels

`make(chan T, n)` gives the channel a queue of capacity `n`. Sends block only
when the buffer is full; receives block only when it is empty.

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 3) // capacity 3

	ch <- 1 // no receiver needed -- goes into the buffer
	ch <- 2
	ch <- 3
	// ch <- 4 would block forever: buffer full, nobody receiving

	fmt.Println(len(ch), cap(ch)) // 3 3

	fmt.Println(<-ch) // 1
	fmt.Println(<-ch) // 2
	fmt.Println(<-ch) // 3
}
```

Use a buffer when a producer legitimately runs ahead of a consumer, or when
you know exactly how many results are coming. Do not use it to "make the
deadlock go away" — that only hides the bug until the buffer fills.

## Closing and ranging over channels

The **sender** closes a channel to say "no more values are coming". Receivers
can then `range` over it and the loop ends automatically.

```go
package main

import "fmt"

func produce(n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out) // signal completion however we exit
		for i := 1; i <= n; i++ {
			out <- i * i
		}
	}()
	return out // receive-only channel: callers cannot send or close
}

func main() {
	for v := range produce(5) {
		fmt.Print(v, " ")
	}
	fmt.Println()
	// Output: 1 4 9 16 25
}
```

Receiving from a closed channel returns the zero value immediately. Use the
comma-ok form to tell "closed" from "genuine zero":

```go
v, ok := <-ch // ok == false means the channel is closed and drained
```

Closing rules that matter:

- Only the sender closes. A receiver closing makes the sender panic.
- Closing an already-closed channel panics.
- Sending on a closed channel panics.
- You do **not** have to close every channel. Closing is a signal, not
  cleanup — garbage collection handles unreferenced channels fine.

## Directional channel types

`chan<- int` is send-only, `<-chan int` is receive-only. Using them in
function signatures lets the compiler enforce who does what:

```go
func producer(out chan<- int) { out <- 1 } // can only send
func consumer(in <-chan int)  { <-in }     // can only receive
```

A bidirectional `chan int` converts to either automatically, so callers just
pass their normal channel.

## `select`: waiting on several channels

`select` blocks until one of its cases can proceed. If several are ready, one
is chosen at random.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fast := make(chan string)
	slow := make(chan string)

	go func() { time.Sleep(20 * time.Millisecond); fast <- "fast result" }()
	go func() { time.Sleep(80 * time.Millisecond); slow <- "slow result" }()

	for received := 0; received < 2; received++ {
		select {
		case msg := <-fast:
			fmt.Println("got:", msg)
		case msg := <-slow:
			fmt.Println("got:", msg)
		case <-time.After(200 * time.Millisecond):
			fmt.Println("timed out")
			return
		}
	}
	// Output:
	// got: fast result
	// got: slow result
}
```

`time.After` returns a channel that delivers a value after the duration — the
standard way to bound how long you are willing to wait. Adding a `default:`
case makes the `select` non-blocking: it runs immediately if nothing else is
ready.

## Deadlocks and goroutine leaks

The two failure modes you will actually hit:

**Deadlock** — every goroutine is blocked. The runtime detects this and
crashes with a clear message:

```go
func main() {
	ch := make(chan int) // unbuffered
	ch <- 1              // nobody is receiving; main blocks forever
	fmt.Println(<-ch)
}
// fatal error: all goroutines are asleep - deadlock!
```

**Goroutine leak** — a goroutine blocks forever while `main` carries on. The
runtime *cannot* detect this; the goroutine just sits there holding memory:

```go
func leak() {
	ch := make(chan int) // unbuffered
	go func() {
		ch <- compute() // blocks forever if nobody ever receives
	}()
	// ... function returns without receiving -- goroutine is stranded
}
```

Give every goroutine a guaranteed exit path: a buffered channel with room for
the result, or cancellation via `context` (covered in
[Level 3, Module 4](../level-3/04-context-package.md)). Printing
`runtime.NumGoroutine()` before and after a workload is a quick leak check.

## Protecting shared state with a mutex

Channels are the idiomatic default, but plain shared state guarded by
`sync.Mutex` is simpler for a counter or cache:

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}

func main() {
	c := &Counter{}
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); c.Inc() }()
	}
	wg.Wait()
	fmt.Println(c.Value()) // 1000 -- always, thanks to the mutex
}
```

Without the mutex, `c.n++` is a read-modify-write race and the total comes out
below 1000. Run any concurrent program with `go run -race .` — the race
detector finds these bugs reliably and is the most valuable tool in Go
concurrency work.

## Cheat sheet

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

## Related lessons

- Worker pools, fan-in/fan-out and pipelines build on this in
  [Level 3, Module 1](../level-3/01-concurrency-patterns.md).
- Cancellation and deadlines: [Level 3, Module 4](../level-3/04-context-package.md).
- Functions and closures recap:
  [Level 1, Module 4](../level-1/04-functions-multiple-returns.md).

## Exercise

Write a program that squares the numbers 1–10 concurrently. Launch one
goroutine per number, each sending `n*n` into a shared channel; use a
`sync.WaitGroup` plus a separate goroutine that calls `close(ch)` after
`wg.Wait()`. In `main`, `range` over the channel, sum the values, and print the
total (it should be 385). Then run it with `go run -race .` and confirm it is
clean. Bonus: rewrite it with a `sync.Mutex` guarding a shared `int` instead of
a channel, and decide which version you find easier to read.
