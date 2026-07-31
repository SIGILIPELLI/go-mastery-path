# 01 · Interfaces

## What an interface actually is

An interface in Go is a named set of method signatures. Any type that has
those methods **automatically** satisfies the interface — there is no
`implements` keyword, no inheritance, no registration step. This is called
*implicit satisfaction*, and it is the single most important design idea in
Go: the interface belongs to the code that *consumes* it, not to the type
that implements it.

```go
package main

import "fmt"

// Shape declares behaviour, not data.
type Shape interface {
	Area() float64
	Perimeter() float64
}

type Rectangle struct {
	Width, Height float64
}

// Rectangle satisfies Shape simply by having both methods.
func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64      { return 3.14159 * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * 3.14159 * c.Radius }

func describe(s Shape) {
	fmt.Printf("%T -> area %.2f, perimeter %.2f\n", s, s.Area(), s.Perimeter())
}

func main() {
	shapes := []Shape{
		Rectangle{Width: 3, Height: 4},
		Circle{Radius: 2},
	}
	for _, s := range shapes {
		describe(s)
	}
	// Output:
	// main.Rectangle -> area 12.00, perimeter 14.00
	// main.Circle -> area 12.57, perimeter 12.57
}
```

Notice that `Rectangle` and `Circle` never mention `Shape`. You could define
`Shape` in a completely different package, months later, and both types would
still satisfy it.

## Small interfaces win

The Go standard library's most-used interfaces have exactly one method:

```go
type Stringer interface{ String() string }            // fmt
type error interface{ Error() string }                // builtin
type Reader interface{ Read(p []byte) (int, error) }  // io
type Writer interface{ Write(p []byte) (int, error) } // io
```

Implementing `fmt.Stringer` changes how your type prints anywhere `%v`,
`%s`, or `fmt.Println` is used:

```go
package main

import "fmt"

type Temperature float64

// String satisfies fmt.Stringer.
func (t Temperature) String() string {
	return fmt.Sprintf("%.1f°C", float64(t))
}

func main() {
	t := Temperature(21.456)
	fmt.Println(t)               // 21.5°C
	fmt.Printf("today: %v\n", t) // today: 21.5°C
}
```

The design rule Go programmers repeat: *the bigger the interface, the weaker
the abstraction*. Accept the smallest interface that does the job. A function
that only reads should take `io.Reader`, not `*os.File` — then it works with
files, network connections, `strings.Reader`, and test buffers alike.

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"strings"
)

// countLines works with ANY io.Reader: a file, an HTTP body, a string.
func countLines(r io.Reader) (int, error) {
	scanner := bufio.NewScanner(r)
	n := 0
	for scanner.Scan() {
		n++
	}
	return n, scanner.Err()
}

func main() {
	text := "alpha\nbeta\ngamma\n"
	n, err := countLines(strings.NewReader(text))
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println("lines:", n) // lines: 3
}
```

## The empty interface and `any`

`interface{}` has zero methods, so *every* type satisfies it. Since Go 1.18
the alias `any` means exactly the same thing and is the preferred spelling.

```go
package main

import "fmt"

func printAnything(v any) {
	fmt.Printf("value=%v type=%T\n", v, v)
}

func main() {
	printAnything(42)          // value=42 type=int
	printAnything("hello")     // value=hello type=string
	printAnything([]int{1, 2}) // value=[1 2] type=[]int
}
```

`any` is an escape hatch, not a design tool. Once a value is stored in `any`
the compiler knows nothing about it, so you must recover the concrete type
before you can do anything useful with it.

## Type assertions and type switches

A **type assertion** pulls a concrete type back out. Always prefer the
two-value form — the single-value form panics on failure.

```go
package main

import "fmt"

func main() {
	var v any = "gopher"

	s, ok := v.(string)
	fmt.Println(s, ok) // gopher true

	n, ok := v.(int)
	fmt.Println(n, ok) // 0 false -- zero value, no panic

	// n = v.(int)  // this form would panic: interface conversion
}
```

A **type switch** handles several possibilities at once:

```go
package main

import "fmt"

func classify(v any) string {
	switch x := v.(type) {
	case nil:
		return "nil value"
	case int:
		return fmt.Sprintf("int, doubled = %d", x*2)
	case string:
		return fmt.Sprintf("string of length %d", len(x))
	case []int:
		return fmt.Sprintf("int slice with %d elements", len(x))
	case error:
		return "an error: " + x.Error()
	default:
		return fmt.Sprintf("unhandled type %T", v)
	}
}

func main() {
	fmt.Println(classify(7))              // int, doubled = 14
	fmt.Println(classify("hi"))           // string of length 2
	fmt.Println(classify([]int{1, 2, 3})) // int slice with 3 elements
	fmt.Println(classify(3.14))           // unhandled type float64
	fmt.Println(classify(nil))            // nil value
}
```

Inside each `case`, `x` already has that case's type — no extra assertion
needed.

## The nil interface trap

This is the classic Go gotcha, and it bites nearly everyone once. An
interface value is a *pair*: `(type, value)`. It equals `nil` only when
**both** halves are nil. Storing a nil pointer of a concrete type produces a
non-nil interface.

```go
package main

import "fmt"

type MyError struct{ msg string }

func (e *MyError) Error() string { return e.msg }

// BUG: returns a typed nil pointer as an error.
func doWorkBad(fail bool) error {
	var e *MyError // nil pointer
	if fail {
		e = &MyError{msg: "it failed"}
	}
	return e // interface is (*MyError, nil) -- NOT nil!
}

// FIXED: return a literal nil on the success path.
func doWorkGood(fail bool) error {
	if fail {
		return &MyError{msg: "it failed"}
	}
	return nil
}

func main() {
	if err := doWorkBad(false); err != nil {
		fmt.Println("bad version reports failure:", err)
	}
	if err := doWorkGood(false); err != nil {
		fmt.Println("never reached")
	} else {
		fmt.Println("good version: success")
	}
	// Output:
	// bad version reports failure: <nil>
	// good version: success
}
```

Rule of thumb: never declare a concrete pointer variable and return it as an
interface. Return `nil` explicitly on the success path.

## Interface embedding

Interfaces can be composed out of other interfaces:

```go
package main

import "fmt"

type Reader interface{ Read() string }
type Closer interface{ Close() error }

// ReadCloser is the union of both method sets.
type ReadCloser interface {
	Reader
	Closer
}

type File struct{ name string }

func (f File) Read() string { return "contents of " + f.name }
func (f File) Close() error { return nil }

func use(rc ReadCloser) {
	fmt.Println(rc.Read())
	if err := rc.Close(); err != nil {
		fmt.Println("close error:", err)
	}
}

func main() {
	use(File{name: "notes.txt"}) // contents of notes.txt
}
```

This is exactly how `io.ReadWriter` and `io.ReadWriteCloser` are defined in
the standard library.

## Compile-time satisfaction checks

To be sure a type implements an interface — and get a clear error at build
time if someone later removes a method — add a blank assignment:

```go
var _ Shape = (*Rectangle)(nil) // fails to compile if Rectangle drops a method
```

It costs nothing at runtime and documents the intent.

## Cheat sheet

| Task | Syntax |
|------|--------|
| Declare an interface | `type S interface { Area() float64 }` |
| Satisfy an interface | just define the methods — nothing to declare |
| Empty interface | `any` (or `interface{}`) |
| Safe type assertion | `v, ok := x.(Concrete)` |
| Type switch | `switch v := x.(type) { case int: ... }` |
| Embed interfaces | `type RC interface { Reader; Closer }` |
| Compile-time check | `var _ Iface = (*Impl)(nil)` |
| Print concrete type | `fmt.Printf("%T", x)` |

## Related lessons

- Methods are what satisfy interfaces — see [Module 3](03-methods-receivers.md)
  for the value-vs-pointer receiver rules that decide *which* type satisfies.
- `error` is an interface; [Module 4](04-custom-errors-wrapping.md) builds
  custom error types on top of it.
- Structs and their methods were introduced in
  [Level 1, Module 6](../level-1/06-structs.md).

## Exercise

Define an interface `Notifier` with a single method `Notify(message string) error`.
Implement it with two types: `EmailNotifier` (holds an address) and
`SMSNotifier` (holds a phone number); each should print where the message went
and return `nil`. Write `broadcast(ns []Notifier, msg string)` that loops over
all notifiers and reports any error. Then add a `LogNotifier` that returns an
error when `message` is empty, and confirm `broadcast` surfaces it. Finally, add
a compile-time check (`var _ Notifier = (*EmailNotifier)(nil)`) for each type.
