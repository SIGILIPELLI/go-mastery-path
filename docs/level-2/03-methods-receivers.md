# 03 · Methods & Receivers

## A method is a function with a receiver

Go has no classes. Instead you attach behaviour to a type by declaring a
function with an extra parameter — the *receiver* — written before the
function name.

```go
package main

import "fmt"

type Rectangle struct {
	Width, Height float64
}

// (r Rectangle) is the receiver. Rectangle now "has" an Area method.
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func main() {
	rect := Rectangle{Width: 3, Height: 4}
	fmt.Println(rect.Area()) // 12
}
```

`rect.Area()` is exactly equivalent to calling `Area(rect)` — the receiver is
just a parameter with nicer syntax. That equivalence is worth remembering,
because it explains everything that follows.

## Value receivers get a copy

A value receiver (`func (r Rectangle)`) receives a **copy** of the value.
Mutations inside the method are invisible to the caller.

```go
package main

import "fmt"

type Counter struct{ n int }

// Value receiver: operates on a copy.
func (c Counter) IncBroken() {
	c.n++ // modifies the copy, then throws it away
}

// Pointer receiver: operates on the original.
func (c *Counter) Inc() {
	c.n++
}

func main() {
	c := Counter{}
	c.IncBroken()
	fmt.Println(c.n) // 0 -- nothing happened

	c.Inc()
	c.Inc()
	fmt.Println(c.n) // 2
}
```

This is the same value-vs-reference distinction covered in
[Level 1, Module 7](../level-1/07-pointers.md), just wearing method syntax.

## Pointer receivers modify and avoid copying

Use a pointer receiver (`func (t *T)`) when either is true:

1. The method needs to **modify** the receiver.
2. The receiver is **large** and copying it every call would be wasteful.
3. Any *other* method on the type already uses a pointer receiver — mixing is
   confusing and causes the method-set problems described below.

```go
package main

import "fmt"

type Account struct {
	Owner   string
	Balance float64
}

func (a *Account) Deposit(amount float64) {
	a.Balance += amount
}

func (a *Account) Withdraw(amount float64) error {
	if amount > a.Balance {
		return fmt.Errorf("insufficient funds: have %.2f, need %.2f", a.Balance, amount)
	}
	a.Balance -= amount
	return nil
}

// Read-only, but kept as a pointer receiver for consistency.
func (a *Account) String() string {
	return fmt.Sprintf("%s: $%.2f", a.Owner, a.Balance)
}

func main() {
	acct := &Account{Owner: "Ada", Balance: 100}
	acct.Deposit(50)
	fmt.Println(acct) // Ada: $150.00

	if err := acct.Withdraw(500); err != nil {
		fmt.Println("error:", err)
	}
	// error: insufficient funds: have 150.00, need 500.00
}
```

## Go's automatic address-of and dereference

You almost never write `(&x).Method()` or `(*p).Method()` because the compiler
inserts `&` and `*` for you when the value is *addressable*:

```go
package main

import "fmt"

type Point struct{ X, Y int }

func (p *Point) Scale(f int) { p.X *= f; p.Y *= f }
func (p Point) String() string {
	return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

func main() {
	p := Point{1, 2}
	p.Scale(3) // shorthand for (&p).Scale(3)
	fmt.Println(p) // (3, 6)

	ptr := &Point{5, 5}
	fmt.Println(ptr.String()) // shorthand for (*ptr).String()
}
```

The convenience stops at non-addressable values — map elements, function
return values, and literals cannot have their address taken:

```go
m := map[string]Point{"a": {1, 2}}
// m["a"].Scale(2)  // compile error: cannot call pointer method on m["a"]

// Workaround: pull it out, modify, put it back.
p := m["a"]
p.Scale(2)
m["a"] = p
```

Storing `map[string]*Point` instead avoids the dance entirely.

## Method sets: the rule that trips people up

This is where receiver choice stops being style and starts being semantics.

- The method set of `T` contains only methods with **value** receivers.
- The method set of `*T` contains methods with **both** value and pointer
  receivers.

Since interface satisfaction is checked against the method set, a value of
type `T` does **not** satisfy an interface whose methods use pointer
receivers:

```go
package main

import "fmt"

type Shape interface{ Area() float64 }

type Square struct{ Side float64 }

// Pointer receiver!
func (s *Square) Area() float64 { return s.Side * s.Side }

func main() {
	// var s Shape = Square{Side: 3}
	// compile error: Square does not implement Shape
	//   (method Area has pointer receiver)

	var s Shape = &Square{Side: 3} // works: *Square has Area
	fmt.Println(s.Area())          // 9
}
```

The reason: the compiler can always take the address of an addressable
variable, but an interface holds a *copy* of the value with no address to
take, so it cannot promote `Square` to `*Square` automatically.

| Receiver on the method | `T` satisfies the interface? | `*T` satisfies it? |
|---|---|---|
| value: `func (t T) M()` | yes | yes |
| pointer: `func (t *T) M()` | **no** | yes |

Practical takeaway: **pick one receiver style per type and stick to it.** If
any method needs a pointer, make them all pointers, and pass `&T{}` around.

## Methods on non-struct types

Any type you define in your own package can have methods — not just structs:

```go
package main

import (
	"fmt"
	"strings"
)

type StringSlice []string

func (s StringSlice) Join(sep string) string { return strings.Join(s, sep) }

func (s StringSlice) Upper() StringSlice {
	out := make(StringSlice, len(s))
	for i, v := range s {
		out[i] = strings.ToUpper(v)
	}
	return out
}

type Celsius float64

func (c Celsius) ToFahrenheit() float64 { return float64(c)*9/5 + 32 }
func (c Celsius) String() string        { return fmt.Sprintf("%.1f°C", float64(c)) }

func main() {
	names := StringSlice{"ada", "grace", "alan"}
	fmt.Println(names.Upper().Join(", ")) // ADA, GRACE, ALAN

	temp := Celsius(100)
	fmt.Println(temp, temp.ToFahrenheit()) // 100.0°C 212
}
```

You cannot define methods on a type from another package (`func (s
strings.Builder) Foo()` is illegal) — define your own named type wrapping it
instead.

## Nil receivers are legal

A pointer receiver method can be called on a `nil` pointer. This is not a
crash by itself; it only panics if the method dereferences the nil. Handling
`nil` deliberately can make an API pleasant:

```go
package main

import "fmt"

type List struct {
	Value int
	Next  *List
}

// Len works even on a nil *List -- an empty list has length 0.
func (l *List) Len() int {
	if l == nil {
		return 0
	}
	return 1 + l.Next.Len()
}

func main() {
	var empty *List
	fmt.Println(empty.Len()) // 0 -- no panic

	list := &List{1, &List{2, &List{3, nil}}}
	fmt.Println(list.Len()) // 3
}
```

## Method values and method expressions

A method can be detached from its receiver and passed around like a function:

```go
package main

import "fmt"

type Greeter struct{ Name string }

func (g Greeter) Greet() string { return "Hello, " + g.Name }

func main() {
	g := Greeter{Name: "Ada"}

	// Method value: receiver is bound now.
	f := g.Greet
	g.Name = "Changed"
	fmt.Println(f()) // Hello, Ada -- the copy was captured at bind time

	// Method expression: receiver becomes the first argument.
	h := Greeter.Greet
	fmt.Println(h(Greeter{Name: "Grace"})) // Hello, Grace
}
```

The "bound at bind time" behaviour of method values on *value* receivers
surprises people; with a pointer receiver the method value sees later changes,
because it captured the pointer.

## Cheat sheet

| Task | Syntax |
|------|--------|
| Value-receiver method | `func (t T) M() { ... }` |
| Pointer-receiver method | `func (t *T) M() { ... }` |
| Call on a value | `v.M()` — auto `&v` if needed and addressable |
| Call on a pointer | `p.M()` — auto `*p` if needed |
| Method on a named non-struct | `type ID int; func (i ID) String() string` |
| Satisfy an interface with pointer methods | pass `&T{}`, not `T{}` |
| Method value | `f := v.M; f()` |
| Method expression | `f := T.M; f(v)` |
| Nil-safe method | `if t == nil { return zero }` |

## Related lessons

- Method sets decide interface satisfaction — see
  [Module 1](01-interfaces.md).
- Pointer fundamentals: [Level 1, Module 7](../level-1/07-pointers.md).
- Struct declaration and embedding:
  [Level 1, Module 6](../level-1/06-structs.md).

## Exercise

Build a `Stack` type over `[]int` with methods `Push(v int)`, `Pop() (int,
error)`, `Peek() (int, error)`, `Len() int`, and `IsEmpty() bool`. Decide
which need pointer receivers and which do not, and write a comment explaining
each choice. `Pop` and `Peek` must return an error on an empty stack rather
than panicking. Then define `type Sizer interface{ Len() int }` and try
assigning both `Stack{}` and `&Stack{}` to it — observe which compiles and
explain why using the method-set table above.
