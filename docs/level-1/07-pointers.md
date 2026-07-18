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

## Exercise

Write a function `increment(n *int)` that adds 1 to the int a pointer
points to. Then write a function `resetScores(scores *[]int)` that sets the
slice a pointer points to back to an empty slice (`*scores = []int{}`). Call
both from `main` and print the results to confirm the mutations are visible
to the caller.
