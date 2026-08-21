# 05 · Design Patterns, the Go Way

Go deliberately lacks classes, inheritance, and constructors — so classic
Gang-of-Four patterns show up in different clothes: small interfaces,
closures, and struct embedding do the work that other languages hand to
class hierarchies. This module covers the three patterns that come up
constantly in idiomatic Go code.

## Functional options

Go has no constructor overloading and no default-argument syntax, so
configurable constructors use a variadic slice of functions that mutate the
value being built:

```go
package main

import "fmt"

type Server struct {
	host    string
	port    int
	timeout int
}

type Option func(*Server)

func WithPort(p int) Option    { return func(s *Server) { s.port = p } }
func WithTimeout(t int) Option { return func(s *Server) { s.timeout = t } }

func NewServer(host string, opts ...Option) *Server {
	s := &Server{host: host, port: 8080, timeout: 30} // sane defaults first
	for _, opt := range opts {
		opt(s)
	}
	return s
}

func main() {
	s1 := NewServer("localhost")
	s2 := NewServer("localhost", WithPort(9090), WithTimeout(5))
	fmt.Printf("s1: %+v\n", *s1)
	fmt.Printf("s2: %+v\n", *s2)
}
```

```console
$ go run .
s1: {host:localhost port:8080 timeout:30}
s2: {host:localhost port:9090 timeout:5}
```

`NewServer("localhost")` and `NewServer("localhost", WithPort(9090),
WithTimeout(5))` are both valid calls to the *same* constructor — no
overloads, no giant options struct with a dozen optional pointer fields.
Adding a new option later (`WithTLS(cert)`) never breaks existing call
sites, which is the real payoff over a positional-argument constructor.

## Strategy via interfaces

Where other languages might reach for a `Strategy` base class, Go just
defines a one-method interface and lets any type satisfy it implicitly:

```go
type Discount interface {
	Apply(price float64) float64
}

type NoDiscount struct{}

func (NoDiscount) Apply(price float64) float64 { return price }

type PercentOff struct{ Pct float64 }

func (p PercentOff) Apply(price float64) float64 { return price * (1 - p.Pct/100) }

func checkout(price float64, d Discount) float64 { return d.Apply(price) }
```

```console
$ go run .
no discount: 100
20% off: 80
```

`checkout` never imports `NoDiscount` or `PercentOff` by name — it only
knows about `Discount`. Adding a `BuyOneGetOne` strategy later requires
touching nothing in `checkout`, which is the whole point of coding against
an interface instead of a concrete type.

## Decorator via composition

Wrapping one implementation of an interface inside another, each adding
behavior, replaces the inheritance-based decorator pattern:

```go
type Notifier interface {
	Notify(msg string) string
}

type baseNotifier struct{}

func (baseNotifier) Notify(msg string) string { return "sent: " + msg }

type withPrefix struct {
	next   Notifier
	prefix string
}

func (w withPrefix) Notify(msg string) string { return w.next.Notify(w.prefix + msg) }

func main() {
	var n Notifier = baseNotifier{}
	n = withPrefix{next: n, prefix: "[ALERT] "}
	fmt.Println(n.Notify("disk full"))
}
```

```console
$ go run .
sent: [ALERT] disk full
```

`withPrefix` holds a `Notifier` and *is* a `Notifier` — stacking another
decorator (say, one that adds a timestamp) is just wrapping again:
`withTimestamp{next: n}`. Each layer only knows about the interface, never
about what's underneath it.

## Go-specific traps

- **A functional option's zero value must still be usable.** If `opts` is
  empty, `NewServer` must produce something sane — options are for
  *overrides*, not for filling in mandatory fields.
- **Interface satisfaction is implicit and structural** — `NoDiscount`
  never writes `implements Discount` anywhere. This is a feature (decouples
  packages) but means a typo in a method name doesn't fail until you try to
  assign the type where the interface is expected, sometimes far from the
  typo itself.
- **Embedding is not inheritance.** `type withPrefix struct { next Notifier
  }` composes by holding a reference and delegating explicitly — Go has no
  mechanism where a struct "automatically" overrides one method of an
  embedded type while inheriting the rest polymorphically; each wrapper
  must implement the whole interface itself (even if just by forwarding).
- **Value vs. pointer receivers change what satisfies the interface.** If
  `Apply` were defined on `*PercentOff` instead of `PercentOff`, only
  `&PercentOff{...}`, not `PercentOff{...}`, would satisfy `Discount` —
  worth checking when an assignment mysteriously fails to compile.

## Cheat sheet

| GoF-flavored need | Go idiom |
|---|---|
| Configurable constructor | Functional options: `func(*T) ` closures, `opts ...Option` |
| Interchangeable algorithms | Small interface + multiple implementing types |
| Add behavior without subclassing | Wrap the interface in another type holding `next` |
| "Abstract base class" | An interface with no default implementation |
| Singleton-ish shared state | Package-level var + `sync.Once` (see [Module 1](01-concurrency-patterns.md)) |
| Builder-style construction | Return `*T` from each step, or use functional options |

## Related lessons

- `sync.Once` as Go's answer to lazy singleton initialization:
  [Module 1](01-concurrency-patterns.md).
- Interfaces and generics in more depth: [Module 9](09-interfaces-generics.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) uses a `Store`
  interface (strategy pattern) so the SQLite backend can be swapped for an
  in-memory fake in tests.

## Exercise

Add a `Middleware` decorator pattern around an `http.Handler` (reusing the
shape from [Module 2](02-building-rest-apis.md)'s `withLogging`), then use
functional options to configure a `Router` struct that holds a list of
middlewares (`WithMiddleware(m)`) and a default 404 handler
(`WithNotFound(h)`). Wire two middlewares — logging and a fake auth check
that rejects requests missing an `Authorization` header — and confirm with
`curl` that the ordering you registered them in is the order they run.
