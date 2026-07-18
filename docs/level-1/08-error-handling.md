# 08 · Error Handling

## Errors are values, not exceptions

Go has no `try`/`catch`. Instead, any function that can fail returns an
`error` as its last return value — `nil` means success, non-nil means
failure. The caller is expected to check it immediately.

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("cannot divide by zero")
	}
	return a / b, nil
}

func main() {
	result, err := divide(10, 2)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	fmt.Println("Result:", result) // Result: 5

	result, err = divide(10, 0)
	if err != nil {
		fmt.Println("Error:", err) // Error: cannot divide by zero
		return
	}
	fmt.Println("Result:", result)
}
```

## The `error` interface

`error` is just an interface with one method:

```go
type error interface {
	Error() string
}
```

Anything implementing `Error() string` can be returned as an `error` — this
is what lets you build custom, structured error types (explored fully in
[Level 2](../level-2/04-custom-errors-wrapping.md)). For now, two standard
ways to build a plain error message:

```go
package main

import (
	"errors"
	"fmt"
)

func validateAge(age int) error {
	if age < 0 {
		return errors.New("age cannot be negative")
	}
	if age > 150 {
		return fmt.Errorf("age %d is unrealistically high", age)
	}
	return nil
}

func main() {
	for _, age := range []int{30, -5, 200} {
		if err := validateAge(age); err != nil {
			fmt.Println("invalid:", err)
			continue
		}
		fmt.Println("valid age:", age)
	}
}
```

`fmt.Errorf` works like `fmt.Sprintf` but returns an `error` — the go-to
choice whenever the message needs to include a value.

## The if err != nil idiom

This four-line shape appears after almost every fallible call in Go code:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	inputs := []string{"42", "abc", "17"}

	for _, s := range inputs {
		n, err := strconv.Atoi(s)
		if err != nil {
			fmt.Printf("skipping %q: %v\n", s, err)
			continue
		}
		fmt.Println("parsed:", n)
	}
	// parsed: 42
	// skipping "abc": strconv.Atoi: parsing "abc": invalid syntax
	// parsed: 17
}
```

It looks repetitive coming from exception-based languages, but the payoff is
that every point where something can fail is visible directly in the
control flow — nothing is thrown silently across stack frames.

## panic and recover (rare, not your default tool)

`panic` stops normal execution immediately; `recover` (used inside a
deferred function) can catch a panic and let the program continue. Reserve
these for truly unrecoverable programmer errors (e.g. an invariant that
should be impossible) — never as a substitute for returning an `error`.

```go
package main

import "fmt"

func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recovered from panic: %v", r)
		}
	}()

	result = a / b // dividing by zero panics for integers
	return result, nil
}

func main() {
	result, err := safeDivide(10, 0)
	if err != nil {
		fmt.Println("error:", err) // error: recovered from panic: runtime error: integer divide by zero
		return
	}
	fmt.Println(result)
}
```

## Cheat sheet

| Task | Syntax |
|------|--------|
| Return a plain error | `errors.New("message")` |
| Return a formatted error | `fmt.Errorf("bad value: %d", n)` |
| Check for failure | `if err != nil { ... }` |
| Custom error type | `type X struct{}; func (X) Error() string { ... }` |
| Stop execution | `panic("message")` |
| Catch a panic | `defer func() { recover() }()` |

## Exercise

Write a function `safeSqrt(x float64) (float64, error)` that returns an
error (`"cannot take sqrt of a negative number"`) if `x < 0`, and otherwise
returns `math.Sqrt(x)`. Call it in a loop over `[]float64{16, -4, 25}`,
printing either the result or the error using the `if err != nil` idiom.
