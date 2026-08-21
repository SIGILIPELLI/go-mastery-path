# 04 · The `context` Package

[Module 1](01-concurrency-patterns.md) ended with a bare `select` against
`ctx.Done()` and promised details later. This module covers `context.Context`
properly: deadlines, cancellation, and the narrow, debated case for passing
values through it — the mechanism Go uses to propagate "give up" across
API boundaries and goroutines.

## Timeouts and cancellation

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func slowWork(ctx context.Context, d time.Duration) error {
	select {
	case <-time.After(d):
		fmt.Println("work finished")
		return nil
	case <-ctx.Done():
		fmt.Println("work cancelled:", ctx.Err())
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	err := slowWork(ctx, 300*time.Millisecond)
	fmt.Println("returned err:", err)
	fmt.Println("is DeadlineExceeded:", errors.Is(err, context.DeadlineExceeded))

	ctx2, cancel2 := context.WithCancel(context.Background())
	go func() {
		time.Sleep(50 * time.Millisecond)
		cancel2()
	}()
	err2 := slowWork(ctx2, 500*time.Millisecond)
	fmt.Println("returned err2:", err2)
	fmt.Println("is Canceled:", errors.Is(err2, context.Canceled))
}
```

```console
$ go run .
work cancelled: context deadline exceeded
returned err: context deadline exceeded
is DeadlineExceeded: true
work cancelled: context canceled
returned err2: context canceled
is Canceled: true
```

`slowWork` never checks a boolean flag — it races `time.After(d)` against
`ctx.Done()` in a `select`, so whichever happens first wins. `WithTimeout`
produces `context.DeadlineExceeded` when the clock runs out; an explicit
`cancel()` call (as `cancel2` shows) produces `context.Canceled` instead.
Both satisfy `errors.Is` against the respective sentinel, so callers can
distinguish "ran out of time" from "someone gave up" without string matching.

**The trap**: `context.WithTimeout` and `context.WithCancel` both return a
`cancel` function that *must* be called even if the context expires or is
cancelled naturally — `defer cancel()` immediately after creation. Skipping
it leaks the timer goroutine backing the context until the parent context
itself is done, which in a long-running server can be never.

## Passing values (sparingly)

```go
package main

import (
	"context"
	"fmt"
)

type ctxKey string

const requestIDKey ctxKey = "requestID"

func withRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey, id)
}

func requestID(ctx context.Context) string {
	id, ok := ctx.Value(requestIDKey).(string)
	if !ok {
		return "unknown"
	}
	return id
}

func handle(ctx context.Context) {
	fmt.Println("handling request:", requestID(ctx))
}

func main() {
	ctx := withRequestID(context.Background(), "req-42")
	handle(ctx)
	handle(context.Background())
}
```

```console
$ go run .
handling request: req-42
handling request: unknown
```

The unexported `ctxKey` type is deliberate — a plain `string` key would
collide with any other package's `context.WithValue(ctx, "requestID", ...)`
call using the same literal. Defining a private key type makes collisions
a compile-time impossibility for anyone outside the package.

`context.Value` is for request-scoped metadata that cuts across API
boundaries you don't control — request IDs, auth tokens, tracing spans. It
is *not* a substitute for passing an explicit parameter: if a function
needs a value to do its job, put it in the signature. Reaching into
`ctx.Value` for "regular" arguments makes the dependency invisible and
untyped.

## Go-specific traps

- **A cancelled parent context cancels every context derived from it** —
  `WithTimeout`/`WithCancel`/`WithValue` all build a tree; cancelling the
  root cancels the whole subtree, but cancelling a child never affects its
  parent or siblings.
- **`ctx.Err()` is `nil` until `Done()` fires** — checking it before
  `<-ctx.Done()` has been observed to close is a race; use the channel, not
  a bare `if ctx.Err() != nil` poll, unless you're deliberately sampling.
- **`context.Context` should be the first parameter, named `ctx`**, and
  never stored inside a struct — that's the convention every standard
  library API follows, and deviating from it surprises every caller.
- **`context.TODO()` is not "safe to ignore"** — it signals "this function
  should take a context but the plumbing isn't done yet"; leaving it in
  production code is a marker you forgot to finish the job.
- **Forgetting `defer cancel()`** — flagged above, but worth repeating: `go
  vet` catches the common case (`context.WithTimeout` result's cancel func
  discarded entirely), but not the case where you call it conditionally on
  only some code paths.

## Cheat sheet

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

## Related lessons

- The cancellable-send pattern this module formalizes:
  [Module 1](01-concurrency-patterns.md).
- `net/http` handlers get a per-request context via `r.Context()`, used the
  same way as here: [Module 2](02-building-rest-apis.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) threads request
  contexts into `database/sql` calls so a client disconnect cancels the
  in-flight query.

## Exercise

Write an HTTP handler that calls a simulated slow dependency (`time.Sleep`)
guarded by `context.WithTimeout(r.Context(), 200*time.Millisecond)`. Return
`503 Service Unavailable` when the context's `Done()` fires before the
"dependency" finishes, and `200 OK` otherwise. Test both paths with `curl`
by varying the simulated sleep duration above and below 200ms, and confirm
the response codes match.
