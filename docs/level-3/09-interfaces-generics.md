# 09 · Interfaces & Generics

Level 1/2 covered basic interfaces. This module goes deeper into how
interfaces are satisfied and combined, then covers generics (added in Go
1.18) — type parameters, constraints, and the generic data structures they
make possible without giving up compile-time type safety.

## Generic functions and constraints

```go
package main

import "fmt"

type Number interface {
	~int | ~int64 | ~float64
}

func Sum[T Number](vals []T) T {
	var total T
	for _, v := range vals {
		total += v
	}
	return total
}

func Map[T, U any](in []T, f func(T) U) []U {
	out := make([]U, len(in))
	for i, v := range in {
		out[i] = f(v)
	}
	return out
}

func main() {
	ints := []int{1, 2, 3, 4}
	fmt.Println("sum ints:", Sum(ints))

	floats := []float64{1.5, 2.5, 3.0}
	fmt.Println("sum floats:", Sum(floats))

	strs := Map(ints, func(n int) string { return fmt.Sprintf("n=%d", n) })
	fmt.Println("mapped:", strs)

	type celsius float64
	temps := []celsius{10, 20, 30}
	fmt.Println("sum celsius:", Sum(temps))
}
```

```console
$ go run .
sum ints: 10
sum floats: 7
mapped: [n=1 n=2 n=3 n=4]
sum celsius: 60
```

`Number` is a constraint, not an ordinary interface — it lists the exact
underlying types `Sum` can add. The `~` before `int` means "any type whose
*underlying* type is `int`", which is why `Sum` also works on `celsius`
(defined as `float64` above) without a `~float64` constraint needing to
name `celsius` specifically. Drop the `~` and only the literal named types
would qualify.

`Map[T, U any]` uses `any` (an alias for `interface{}`) for both type
parameters because it needs no operations on them beyond passing values
through — `any` is the right constraint whenever a generic function only
moves values around rather than computing with them.

Calling `Sum` with a type outside its constraint is a compile error, not a
runtime one:

```console
$ cat main.go
func main() {
	strs := []string{"a", "b"}
	Sum(strs)
}

$ go build .
./main.go:17:5: string does not satisfy Number (string missing in ~int | ~int64 | ~float64)
```

That's the entire value proposition over the pre-generics alternative
(`interface{}` plus a type switch and a runtime panic on the wrong type):
the mistake is caught before the program ever runs.

## Generic data structures

```go
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	v := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return v, true
}

func main() {
	var s Stack[string]
	s.Push("a")
	s.Push("b")
	v, ok := s.Pop()
	fmt.Println("popped:", v, ok)
}
```

```console
$ go run .
popped: b true
```

`Stack[T any]` is one definition that becomes `Stack[string]`,
`Stack[int]`, `Stack[MyStruct]`, etc. at each call site — before generics,
this required either code generation, `interface{}` with type assertions
scattered through the caller, or writing `IntStack`/`StringStack` by hand.
`var zero T` is the idiomatic way to produce "the zero value of whatever T
turns out to be" when there's nothing to return, since you can't write
`return nil` for a `T` that might be a non-pointer type like `int`.

## Interfaces recap: composition and the empty interface

Interfaces compose by embedding, building bigger contracts from small ones —
the same principle behind `io.ReadWriter` combining `io.Reader` and
`io.Writer`:

```go
type Reader interface{ Read() string }
type Writer interface{ Write(string) }
type ReadWriter interface {
	Reader
	Writer
}
```

Anything satisfying both one-method interfaces automatically satisfies
`ReadWriter` — there is no explicit "implements" declaration to write or
keep in sync.

## Go-specific traps

- **Type inference doesn't always work** — `Map(ints, func(n int) string
  {...})` infers `T=int, U=string` from the arguments, but if a generic
  function's type parameters can't be inferred from any argument, you must
  supply them explicitly: `Sum[int](vals)`.
- **A constraint's type set is closed** — adding a case (a new numeric
  wrapper type) later that isn't `~int`/`~int64`/`~float64`-based simply
  won't compile against `Number`; there's no way to "extend" a constraint
  from outside its definition the way you might reopen a class elsewhere.
- **Generics are not a substitute for interfaces** — if `Sum` needed to
  call a *method* on `T` (rather than use `+=`), the constraint would be an
  ordinary method-set interface, not a type-set union; reach for the type
  parameter form only when you need the same code to work across types
  that don't share a method.
- **The empty interface `any` disables the compiler's help** — a function
  taking `any` is exactly where generics or method-set interfaces should
  usually go instead, so you get type errors at compile time instead of a
  type assertion panicking in production.
- **Comparing two `any` values holding uncomparable dynamic types (e.g.
  slices) panics at runtime**, not compile time — `==` on two `any` compiles
  fine regardless of what's inside them.

## Cheat sheet

| Need | Syntax |
|---|---|
| Generic function | `func F[T any](v T) T` |
| Multiple type params | `func Map[T, U any](in []T, f func(T) U) []U` |
| Numeric-only constraint | `type Number interface { ~int \| ~int64 \| ~float64 }` |
| Generic struct | `type Stack[T any] struct { items []T }` |
| Method on generic struct | `func (s *Stack[T]) Push(v T)` |
| Zero value of a type param | `var zero T` |
| Explicit type argument | `Sum[int](vals)` (when inference can't determine it) |
| Interface composition | Embed interfaces inside another `interface { A; B }` |

## Related lessons

- Interfaces used for the strategy/decorator patterns:
  [Module 5](05-design-patterns.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) uses a `Store`
  interface (not generics) because its operations differ per resource type
  rather than sharing a uniform shape.

## Exercise

Write a generic `Filter[T any](in []T, keep func(T) bool) []T` and a
generic `Reduce[T, U any](in []T, init U, f func(U, T) U) U`. Use `Reduce`
to reimplement `Sum` for `[]int` (constraint `any` this time, since the
combining logic lives in the caller's closure, not in `Reduce` itself), and
use `Filter` to keep only even numbers from `1..20` before summing them.
Confirm the result is `110` and that `go vet ./...` reports nothing.
