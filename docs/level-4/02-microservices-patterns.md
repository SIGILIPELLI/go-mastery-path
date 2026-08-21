# 02 · Microservices Patterns

A service that calls other services over the network needs defenses a
single-process program doesn't: timeouts on every outbound call, and a way
to stop hammering a dependency that's already failing. This module builds
both against a real HTTP server (via `httptest.NewServer`, no mocking
framework needed) so the failure modes are genuine network behavior, not
simulated.

## A slow downstream dependency

```go
package main

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"time"
)

// pricingService simulates a downstream microservice.
func startPricing() *httptest.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /price/{sku}", func(w http.ResponseWriter, r *http.Request) {
		sku := r.PathValue("sku")
		if sku == "SKU-999" {
			time.Sleep(300 * time.Millisecond) // simulate a slow/misbehaving dependency
		}
		json.NewEncoder(w).Encode(map[string]any{"sku": sku, "price": 19.99})
	})
	return httptest.NewServer(mux)
}
```

`httptest.NewServer` starts a real listener on a real (random, local) port
— every request in this module is a genuine HTTP round trip, just to a
process running in the same test binary instead of across a network.

## Client timeouts: never trust the default

```go
client := &http.Client{Timeout: 100 * time.Millisecond}
```

That single line is the most important one in this module. The zero-value
`http.Client{}` has **no timeout at all** — a hung dependency hangs your
service's request-handling goroutine forever, one per stuck request, until
it exhausts memory or file descriptors. `Timeout` here bounds the *entire*
round trip (connect + write + read), not just the connect phase.

## Circuit breaker: stop calling a dependency that's already down

```go
type circuitBreaker struct {
	failures  int
	threshold int
	open      bool
}

func (cb *circuitBreaker) call(fn func() error) error {
	if cb.open {
		return fmt.Errorf("circuit open: refusing call")
	}
	err := fn()
	if err != nil {
		cb.failures++
		if cb.failures >= cb.threshold {
			cb.open = true
		}
		return err
	}
	cb.failures = 0
	return nil
}
```

Putting it together against the slow `SKU-999` endpoint:

```go
func fetchPrice(client *http.Client, base, sku string) (float64, error) {
	req, _ := http.NewRequest(http.MethodGet, base+"/price/"+sku, nil)
	resp, err := client.Do(req)
	if err != nil {
		return 0, err
	}
	defer resp.Body.Close()
	var out struct{ Price float64 `json:"price"` }
	if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
		return 0, err
	}
	return out.Price, nil
}

func main() {
	pricing := startPricing()
	defer pricing.Close()

	client := &http.Client{Timeout: 100 * time.Millisecond}
	cb := &circuitBreaker{threshold: 2}

	for _, sku := range []string{"SKU-1", "SKU-999", "SKU-999", "SKU-999", "SKU-1"} {
		err := cb.call(func() error {
			price, err := fetchPrice(client, pricing.URL, sku)
			if err != nil {
				return err
			}
			fmt.Printf("%s price=%.2f\n", sku, price)
			return nil
		})
		if err != nil {
			fmt.Printf("%s error: %v\n", sku, err)
		}
	}
}
```

```console
$ go run .
SKU-1 price=19.99
SKU-999 error: Get "http://127.0.0.1:52578/price/SKU-999": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
SKU-999 error: Get "http://127.0.0.1:52578/price/SKU-999": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
SKU-999 error: circuit open: refusing call
SKU-1 error: circuit open: refusing call
```

`SKU-1` succeeds instantly. The first two `SKU-999` calls each burn a full
100ms timing out — that's real time spent, not simulated — and after the
second consecutive failure hits `threshold: 2`, the breaker opens. From
that point on, *every* call fails instantly with `"circuit open"`,
including the third `SKU-999` attempt and, notably, the final `SKU-1` call
that would have succeeded — a real circuit breaker in production needs a
half-open state (allow one trial call after a cooldown) so it can recover,
which this minimal version deliberately omits to keep the mechanism
visible.

## Go-specific traps

- **`http.Client` is safe for concurrent use and meant to be reused** —
  creating a new `http.Client` per request throws away connection pooling
  (`http.Transport`'s keep-alive pool) and can exhaust ephemeral ports
  under load.
- **A circuit breaker without a half-open/reset path never recovers** — the
  version above is permanently open once tripped; real implementations
  (or a library like `sony/gobreaker`) reset `failures` after a cooldown
  and probe with a single trial request before fully closing again.
- **`defer resp.Body.Close()` must run even on non-2xx status codes** — a
  4xx/5xx response still has a body that needs draining and closing, or the
  underlying connection can't be reused from the pool.
- **This breaker isn't goroutine-safe** — `cb.failures`/`cb.open` read and
  written from multiple goroutines needs a `sync.Mutex` around `call`,
  exactly like the `store` type in
  [Level 3, Module 2](../level-3/02-building-rest-apis.md).
- **A single global `threshold` conflates different failure types** — a
  404 (client error, not the dependency's fault) shouldn't usually count
  toward the same threshold as a timeout or 500; production breakers often
  only count specific error classes.

## Cheat sheet

| Concern | Pattern |
|---|---|
| Never hang forever on a dependency | `&http.Client{Timeout: d}` |
| Fake a downstream service for tests | `httptest.NewServer(mux)` |
| Stop calling a failing dependency | Circuit breaker: count failures, `open` after threshold |
| Recover after cooldown | Half-open state — one trial call after a wait, not shown above |
| Bound total time across N dependency calls | `errgroup.WithContext` ([Module 1](01-advanced-concurrency.md)) |
| Reuse connections | One shared `*http.Client` per destination, not one per call |

## Related lessons

- `errgroup` for fanning out to several dependencies with shared
  cancellation: [Module 1](01-advanced-concurrency.md).
- `context.Context` deadlines as an alternative/complement to
  `http.Client.Timeout`: [Level 3, Module 4](../level-3/04-context-package.md).
- gRPC as an alternative transport between services:
  [Module 3](03-grpc-basics.md).

## Exercise

Add a retry-with-backoff wrapper around `fetchPrice` that retries up to 3
times with `50ms * attempt` between attempts, but only for network-level
errors (not for a successfully-received non-2xx status). Wire it *inside*
the circuit breaker's `call` closure so retries count toward — but don't
bypass — the breaker's failure threshold. Run it against `SKU-999` and
confirm the total elapsed time roughly matches `100ms timeout + 50ms +
100ms timeout + 100ms = ~350ms` for the first failing call alone, before
the breaker opens.
