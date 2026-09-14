# 03 · Control Flow

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/hYyGbYiIuK4" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## if / else — no parentheses required

```go
package main

import "fmt"

func main() {
	score := 82

	if score >= 90 {
		fmt.Println("A")
	} else if score >= 80 {
		fmt.Println("B")
	} else {
		fmt.Println("C or below")
	}
}
```

Braces `{}` are mandatory in Go, even for single-statement bodies —
there is no one-line `if` without braces.

## if with an initializer statement

A very common Go idiom: run a statement, then check its result, scoped only
to the `if`/`else`:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	if value, err := strconv.Atoi("42"); err == nil {
		fmt.Println("parsed:", value)
	} else {
		fmt.Println("parse failed:", err)
	}
	// value and err do not exist out here
}
```

This pattern shows up constantly with functions that return `(result, error)`
— see [Module 8](08-error-handling.md).

## for — Go's only loop keyword

Go has exactly one looping construct, `for`, which covers every case other
languages split across `for`/`while`/`do-while`:

```go
package main

import "fmt"

func main() {
	// Classic three-part for
	for i := 0; i < 5; i++ {
		fmt.Println("count:", i)
	}

	// "while" loop -- condition only
	n := 5
	for n > 0 {
		fmt.Println("n is", n)
		n--
	}

	// Infinite loop -- break to exit
	total := 0
	for {
		total++
		if total == 3 {
			break
		}
	}
	fmt.Println("total:", total)

	// range over a slice: index, value
	fruits := []string{"apple", "banana", "cherry"}
	for i, fruit := range fruits {
		fmt.Println(i, fruit)
	}

	// range when you only want values
	for _, fruit := range fruits {
		fmt.Println("fruit:", fruit)
	}
}
```

## continue and labeled loops

```go
package main

import "fmt"

func main() {
	for i := 1; i <= 10; i++ {
		if i%2 != 0 {
			continue // skip odd numbers
		}
		fmt.Println(i)
	}

outer:
	for i := 0; i < 3; i++ {
		for j := 0; j < 3; j++ {
			if j == 1 {
				continue outer // continue the OUTER loop
			}
			fmt.Println(i, j)
		}
	}
}
```

## switch — no automatic fallthrough

Unlike C or Java, a Go `switch` case does **not** fall through by default —
each case breaks automatically. Use the `fallthrough` keyword to opt in.

```go
package main

import "fmt"

func main() {
	day := "Tue"

	switch day {
	case "Sat", "Sun":
		fmt.Println("Weekend")
	case "Mon", "Tue", "Wed", "Thu", "Fri":
		fmt.Println("Weekday")
	default:
		fmt.Println("Unknown")
	}

	// switch with no expression -- reads like an if/else chain
	score := 82
	switch {
	case score >= 90:
		fmt.Println("A")
	case score >= 80:
		fmt.Println("B")
	default:
		fmt.Println("C or below")
	}

	// switch with an initializer, same idea as if
	switch x := 2; x {
	case 1:
		fmt.Println("one")
	case 2:
		fmt.Println("two")
		fallthrough // explicitly fall into the next case
	case 3:
		fmt.Println("three (reached via fallthrough or directly)")
	}
}
```

## How It Actually Works

Go's `switch` doesn't fall through by default because the compiler generates a jump
table (or a chain of compares for small/sparse cases) that jumps straight to the
matching case and then to the end of the switch — falling through would require an
explicit extra jump instruction, which is exactly what the `fallthrough` keyword
inserts when you ask for it. A `for` loop with no clauses (`for { }`) compiles to an
unconditional backward jump — there's no hidden condition check, which is why it's
the idiomatic infinite loop instead of `for true {}` (both compile the same way, but
the former reads as "obviously infinite" to the compiler's dead-code and unreachable
code analysis too). The classic Go 1.21-and-earlier loop-variable-capture bug
(closures inside a `for range` all seeing the last value) happened because the loop
variable was one stack slot reused every iteration; each closure captured a pointer
to that slot, not a copy of the value at capture time. Go 1.22 changed the spec so
each iteration gets its own variable instance, which is why the fix is "upgrade Go,"
not "learn a workaround."


## Cheat sheet

| Construct | Example |
|-----------|---------|
| If | `if x > 0 { ... }` |
| If/else if/else | `if ... else if ... else { ... }` |
| If with initializer | `if v, err := f(); err == nil { ... }` |
| Classic for | `for i := 0; i < n; i++ { ... }` |
| While-style | `for cond { ... }` |
| Infinite | `for { ... break }` |
| Range (index, value) | `for i, v := range slice { ... }` |
| Range (value only) | `for _, v := range slice { ... }` |
| Switch | `switch x { case 1: ...; default: ... }` |
| Switch, no expression | `switch { case cond: ... }` |

## Exercise

Write a program that loops from 1 to 30 and, using `switch` with no
expression, prints `"FizzBuzz"` for multiples of 15, `"Fizz"` for multiples
of 3, `"Buzz"` for multiples of 5, and the number itself otherwise. Then add
a second loop using `for` with `range` over a slice of your favorite foods
that skips (`continue`) any name shorter than 4 characters.
