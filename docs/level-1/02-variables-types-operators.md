# 02 · Variables, Types & Operators

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/wGjU6idYM1s" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Declaring variables

Go is statically typed but infers types for you most of the time. Two ways
to declare a variable:

```go
package main

import "fmt"

func main() {
	// var form: explicit type, works anywhere (including package scope)
	var age int = 30
	var name string = "Ada"

	// type inference: var without a type
	var city = "Boston"

	// short declaration :=  -- only inside function bodies
	country := "USA"

	fmt.Println(age, name, city, country)
}
```

`:=` is the idiomatic choice inside functions. `var` is required at package
level, when you want a zero value with no initializer, or when the inferred
type would be wrong (e.g. you want `float64` but wrote a whole number).

## Zero values

Unlike many languages, Go never leaves a declared variable "undefined" — it
gets a **zero value** automatically:

```go
var count int      // 0
var price float64   // 0.0
var label string     // "" (empty string)
var ready bool       // false
var data []int       // nil (nil slice, safe to read, len is 0)
```

## Basic types

| Category | Types |
|----------|-------|
| Integers | `int`, `int8/16/32/64`, `uint`, `uint8` (`byte`), `uint16/32/64` |
| Floating point | `float32`, `float64` |
| Text | `string`, `rune` (an alias for `int32`, holds a Unicode code point) |
| Boolean | `bool` |
| Complex (rare) | `complex64`, `complex128` |

`int` is the general-purpose choice — its size (32 or 64 bit) matches the
platform's native word size. Use a specific width (`int64`, `uint8`) only
when the size matters, e.g. binary protocols or memory-tight structures.

```go
package main

import "fmt"

func main() {
	var temperature float64 = 98.6
	var initial byte = 'A'          // byte is an alias for uint8
	var letter rune = '日'           // rune holds one Unicode code point

	fmt.Println(temperature, initial, string(letter))
	fmt.Printf("%T %T %T\n", temperature, initial, letter)
	// float64 uint8 int32
}
```

## Constants and `iota`

```go
package main

import "fmt"

const Pi = 3.14159 // untyped constant, adapts to context

const (
	StatusPending = iota // 0
	StatusActive         // 1
	StatusDone            // 2
)

func main() {
	fmt.Println(Pi, StatusPending, StatusActive, StatusDone)
}
```

`iota` resets to `0` at each `const (...)` block and increments by one per
line — the standard idiom for defining a small enum-like set of related
constants.

## Type conversion

Go **never** converts types implicitly. Mixing an `int` and a `float64` in
an expression is a compile error — you must convert explicitly:

```go
package main

import "fmt"

func main() {
	var i int = 10
	var f float64 = 3.0

	// result := i + f          // compile error: mismatched types
	result := float64(i) + f
	fmt.Println(result) // 13

	back := int(result) // truncates, does not round: 13
	fmt.Println(back)
}
```

## Operators

```go
package main

import "fmt"

func main() {
	a, b := 17, 5

	fmt.Println(a + b)  // 22
	fmt.Println(a - b)  // 12
	fmt.Println(a * b)  // 85
	fmt.Println(a / b)  // 3   (integer division truncates)
	fmt.Println(a % b)  // 2   (remainder)

	// Comparison
	fmt.Println(a > b, a == b, a != b) // true false true

	// Logical
	fmt.Println(a > 10 && b < 10) // true
	fmt.Println(a < 10 || b < 10) // true

	// Increment/decrement are statements, not expressions
	a++
	b--
	fmt.Println(a, b) // 18 4
}
```

## How It Actually Works

Every Go value carries a *static type* known at compile time — the compiler uses it
to pick the machine instructions for `+`, decide how many bytes to allocate, and
reject mismatched operations before the binary is even built. This is different from
Python or JS, where a value carries its type at runtime and the interpreter checks it
on every operation. `var x int` allocates 8 bytes (on a 64-bit target) on the stack
if escape analysis proves `x` never outlives the function, or on the heap otherwise —
the compiler decides this, not you (see level-1/07 for more). Numeric conversions
like `float64(i)` are never implicit in Go specifically because implicit widening
conversions are a classic source of silent precision-loss bugs in C; the language
designers traded convenience for a compiler error you see immediately. Untyped
constants (`const x = 5`) are a special case: they live in the compiler's constant
table with arbitrary precision until they're assigned to a typed variable, which is
why `const x = 1 << 100` compiles fine but `var x int64 = 1 << 100` doesn't.


## Cheat sheet

| Concept | Syntax |
|---------|--------|
| Declare with inference | `x := 5` |
| Declare with explicit type | `var x int = 5` |
| Zero value | `var x int` → `0` |
| Constant | `const Pi = 3.14` |
| Enum-like constants | `const ( A = iota; B; C )` |
| Convert types | `float64(x)`, `int(y)` |
| Multiple assignment | `a, b := 1, 2` |

## Exercise

Write a program that declares a rectangle's `width` and `height` as `float64`
using `:=`, computes its area and perimeter, and prints both with `fmt.Printf`
using the `%.2f` verb for two decimal places. Then add an `int` constant
block using `iota` for three shape types (`Rectangle`, `Circle`, `Triangle`)
and print the constant matching your rectangle.
