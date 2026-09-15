---
description: "Pointers — A pointer holds the memory address of a value instead of the value itself. Go uses pointers for two main reasons: letting a function mutate the…"
---

# 07 · Pointers

## Why Go has pointers (but no pointer arithmetic)

A pointer holds the memory address of a value instead of the value itself.
Go uses pointers for two main reasons: letting a function mutate the
caller's data, and avoiding expensive copies of large structs. Unlike C,
Go pointers are safe — no arithmetic, and the garbage collector tracks what
they point to.

```go
package main

import "fmt"

func main() {
	x := 42
	p := &x // & takes the address of x -- p is a *int

	fmt.Println(x)  // 42
	fmt.Println(p)  // 0xc0000140a0 (some address)
	fmt.Println(*p) // 42 -- * dereferences: "the value p points to"

	*p = 100 // writing through the pointer changes x
	fmt.Println(x) // 100
}
```

| Operator | Meaning |
|----------|---------|
| `&x` | "Address of x" — produces a pointer to `x` |
| `*p` | "Value pointed to by p" — dereferences the pointer |
| `var p *int` | Declares `p` as a pointer to an `int` (zero value: `nil`) |

## Pointers let functions mutate caller state

Recall from [Module 6](06-structs.md) that passing a struct by value copies
it. Passing a pointer instead lets the function reach back into the
caller's original data:

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

// pointer receiver -- modifies the CALLER's struct
func movePoint(p *Point, dx, dy int) {
	p.X += dx // Go auto-dereferences: shorthand for (*p).X
	p.Y += dy
}

func main() {
	pt := Point{X: 1, Y: 2}
	movePoint(&pt, 5, 5)
	fmt.Println(pt) // {6 7} -- actually changed this time
}
```

## new() vs &Type{}

```go
package main

import "fmt"

type Counter struct {
	Count int
}

func main() {
	// new(T) allocates zeroed memory for T, returns *T
	c1 := new(Counter)
	c1.Count = 1

	// &T{} is the more idiomatic way when you also want to set fields
	c2 := &Counter{Count: 10}

	fmt.Println(c1, c2)   // &{1} &{10}
	fmt.Println(*c1, *c2) // {1} {10}
}
```

`&Counter{Count: 10}` is by far the more common pattern in real code; `new`
shows up mostly for primitive types or when you want a bare zero value.

## nil pointers

A pointer's zero value is `nil` — dereferencing a `nil` pointer panics at
runtime, so always check before dereferencing when a pointer might be unset:

```go
package main

import "fmt"

func safePrint(p *int) {
	if p == nil {
		fmt.Println("no value")
		return
	}
	fmt.Println(*p)
}

func main() {
	var p *int    // nil
	safePrint(p)  // no value

	x := 5
	safePrint(&x) // 5
}
```

## Pointers to slices and maps: usually unnecessary

Slices and maps already contain an internal pointer to their underlying
data, so passing them by value already lets a function mutate their
contents (append is the one exception — it can return a new slice header):

```go
package main

import "fmt"

func double(nums []int) {
	for i := range nums {
		nums[i] *= 2 // mutates the shared backing array directly
	}
}

func main() {
	values := []int{1, 2, 3}
	double(values)
	fmt.Println(values) // [2 4 6] -- no pointer needed
}
```

## How It Actually Works

Go decides stack vs. heap allocation through *escape analysis*, a compile-time pass
over the function's data-flow graph: if the compiler can prove a value's address
never leaves the function (no pointer to it is returned, stored in a global, sent on
a channel, or captured by an escaping closure), it stays on the stack and gets freed
for free when the function returns — no GC involvement at all. The moment you return
`&localVar` from a function, the compiler marks it as escaping and allocates it on
the heap instead, because the stack frame it would have lived in is gone once the
function returns. You can see this decision directly with `go build -gcflags="-m"`,
which prints "escapes to heap" or "does not escape" for every allocation. This is
why Go pointers are safe to return from functions (unlike a raw pointer to a local
in C) — the compiler silently promotes the allocation to the heap rather than let it
dangle, and it's why minimizing accidental escapes (e.g. passing an interface where
a concrete type would do) is a real, measurable performance lever.


## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Address-of | `p := &x` |
| Dereference | `*p` |
| Pointer type | `var p *int` |
| Pointer to struct literal | `&Point{X: 1, Y: 2}` |
| Allocate zeroed | `p := new(Point)` |
| Auto-deref field access | `p.X` (shorthand for `(*p).X`) |
| Nil check | `if p == nil { ... }` |

## 🔀 See this in another language

- [Rust — Error Handling Basics (Option, Result)](https://sigilipelli.github.io/rust-mastery-path/level-1/07-error-handling-basics/)
- [Dart — Null Safety Basics](https://sigilipelli.github.io/dart-mastery-path/level-1/07-null-safety-basics/)
- [Python — File I/O Basics](https://sigilipelli.github.io/python-mastery-path/level-1/07-file-io/)

## Exercise

Write a function `increment(n *int)` that adds 1 to the int a pointer
points to. Then write a function `resetScores(scores *[]int)` that sets the
slice a pointer points to back to an empty slice (`*scores = []int{}`). Call
both from `main` and print the results to confirm the mutations are visible
to the caller.
