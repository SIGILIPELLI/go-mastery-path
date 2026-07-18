# 04 · Functions & Multiple Returns

## Basic function syntax

```go
package main

import "fmt"

// name(params) returnType
func add(a int, b int) int {
	return a + b
}

// consecutive parameters of the same type can share one annotation
func multiply(a, b int) int {
	return a * b
}

func main() {
	fmt.Println(add(2, 3))      // 5
	fmt.Println(multiply(4, 5)) // 20
}
```

## Multiple return values

This is one of Go's signature features — functions routinely return more
than one value, most commonly a result **and** an error:

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}

func main() {
	result, err := divide(10, 2)
	if err != nil {
		fmt.Println("error:", err)
	} else {
		fmt.Println("result:", result)
	}

	_, err = divide(10, 0)
	if err != nil {
		fmt.Println("error:", err) // error: division by zero
	}
}
```

The blank identifier `_` discards a return value you don't need — Go forces
you to either use every declared variable or explicitly discard it.

## Named return values

Return values can be named up front; a bare `return` sends back their
current values. Useful for short functions and for documenting intent, but
overusing it can hurt readability in longer functions.

```go
package main

import "fmt"

func divmod(a, b int) (quotient, remainder int) {
	quotient = a / b
	remainder = a % b
	return // "naked" return -- sends back quotient, remainder
}

func main() {
	q, r := divmod(17, 5)
	fmt.Println(q, r) // 3 2
}
```

## Variadic functions

A parameter prefixed with `...` accepts zero or more arguments, collected
into a slice inside the function:

```go
package main

import "fmt"

func sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

func main() {
	fmt.Println(sum())           // 0
	fmt.Println(sum(1, 2, 3))    // 6

	values := []int{10, 20, 30}
	fmt.Println(sum(values...)) // spread a slice with ...
}
```

## Functions as values, closures

Functions are first-class values in Go — assign them to variables, pass
them as arguments, return them from other functions.

```go
package main

import "fmt"

// takes a function as a parameter
func applyTwice(f func(int) int, x int) int {
	return f(f(x))
}

// returns a closure that captures "start"
func counterFrom(start int) func() int {
	count := start
	return func() int {
		count++
		return count
	}
}

func main() {
	double := func(x int) int { return x * 2 }
	fmt.Println(applyTwice(double, 3)) // 12

	next := counterFrom(10)
	fmt.Println(next()) // 11
	fmt.Println(next()) // 12
	fmt.Println(next()) // 13
}
```

## defer — run code on function exit

`defer` schedules a call to run right before the surrounding function
returns, regardless of how it returns. It's the idiomatic way to guarantee
cleanup (closing files, unlocking mutexes) sits right next to the code that
acquired the resource.

```go
package main

import "fmt"

func process() {
	fmt.Println("start")
	defer fmt.Println("cleanup (runs last)")
	fmt.Println("middle")
	// "cleanup" prints after "middle", right before process() returns
}

func main() {
	process()
	// start
	// middle
	// cleanup (runs last)
}
```

Multiple `defer` calls run in **LIFO** order (last deferred, first executed).

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Basic function | `func add(a, b int) int { return a + b }` |
| Multiple returns | `func f() (int, error) { ... }` |
| Named returns | `func f() (x int) { x = 1; return }` |
| Variadic | `func f(nums ...int) { ... }` |
| Spread a slice | `f(values...)` |
| Discard a return | `_, err := f()` |
| Function value | `var f func(int) int = double` |
| Closure | `func() func() int { ... }` |
| Deferred cleanup | `defer file.Close()` |

## Exercise

Write a function `minMax(nums ...int) (min, max int)` using named return
values that finds the smallest and largest numbers in a variadic list of
ints (return `0, 0` for an empty call). In `main`, call it with a slice
spread using `...`, and add a `defer` statement that prints `"done"` after
the results are printed.
