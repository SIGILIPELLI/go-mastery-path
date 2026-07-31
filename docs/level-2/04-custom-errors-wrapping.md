# 04 · Custom Errors & Error Wrapping

## Beyond `errors.New`

[Level 1, Module 8](../level-1/08-error-handling.md) established that errors
are ordinary values returned as the last result. That works fine until a
caller needs to *react differently* depending on what went wrong — retry a
timeout, prompt the user on a validation failure, give up on a permission
error. A plain string cannot be inspected reliably, and comparing error text
(`if err.Error() == "not found"`) is fragile and breaks the moment someone
rewords the message.

Go gives you two better tools: **sentinel errors** for fixed conditions, and
**custom error types** for errors carrying data. Wrapping ties them together
across layers.

## Sentinel errors

A sentinel is a package-level error variable that callers compare against.
Declare it once, return it everywhere the condition occurs:

```go
package main

import (
	"errors"
	"fmt"
)

// Sentinels are conventionally named Err... and declared at package level.
var (
	ErrNotFound     = errors.New("record not found")
	ErrUnauthorized = errors.New("unauthorized")
)

var db = map[int]string{1: "Ada", 2: "Grace"}

func lookup(id int, admin bool) (string, error) {
	if !admin {
		return "", ErrUnauthorized
	}
	name, ok := db[id]
	if !ok {
		return "", ErrNotFound
	}
	return name, nil
}

func main() {
	for _, tc := range []struct {
		id    int
		admin bool
	}{{1, true}, {99, true}, {1, false}} {
		name, err := lookup(tc.id, tc.admin)
		switch {
		case errors.Is(err, ErrNotFound):
			fmt.Printf("id %d: no such record\n", tc.id)
		case errors.Is(err, ErrUnauthorized):
			fmt.Printf("id %d: access denied\n", tc.id)
		case err != nil:
			fmt.Println("unexpected:", err)
		default:
			fmt.Printf("id %d: %s\n", tc.id, name)
		}
	}
	// Output:
	// id 1: Ada
	// id 99: no such record
	// id 1: access denied
}
```

Standard-library examples you already use: `io.EOF`, `sql.ErrNoRows`,
`os.ErrNotExist`.

## Custom error types

When the error needs to carry *data* — which field failed, which HTTP status,
how long to wait before retrying — define a type implementing `error`:

```go
package main

import (
	"errors"
	"fmt"
)

// ValidationError carries the offending field and value.
type ValidationError struct {
	Field string
	Value any
	Rule  string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation failed on %q (value %v): %s", e.Field, e.Value, e.Rule)
}

type User struct {
	Name  string
	Email string
	Age   int
}

func validate(u User) error {
	if u.Name == "" {
		return &ValidationError{Field: "Name", Value: u.Name, Rule: "must not be empty"}
	}
	if u.Age < 0 || u.Age > 150 {
		return &ValidationError{Field: "Age", Value: u.Age, Rule: "must be between 0 and 150"}
	}
	return nil
}

func main() {
	err := validate(User{Name: "Ada", Age: 200})

	var ve *ValidationError
	if errors.As(err, &ve) {
		// We now have the structured data, not just a string.
		fmt.Println("field:", ve.Field) // field: Age
		fmt.Println("rule:", ve.Rule)   // rule: must be between 0 and 150
	}
	fmt.Println(err)
	// validation failed on "Age" (value 200): must be between 0 and 150
}
```

Use a **pointer** receiver (`*ValidationError`) and return `&ValidationError{}`.
Two different `*ValidationError` values then compare as different errors, which
is what you want, and `errors.As` targets `**ValidationError` naturally.

## Wrapping with `%w`

As an error travels up through layers, each layer wants to add context
("reading config", "loading user 42") without destroying the original. The
`%w` verb in `fmt.Errorf` wraps: the new error's message includes the old
one, and the original stays retrievable.

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func readConfig(path string) ([]byte, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("reading config %s: %w", path, err)
	}
	return data, nil
}

func startup() error {
	if _, err := readConfig("/nonexistent/app.yaml"); err != nil {
		return fmt.Errorf("startup: %w", err)
	}
	return nil
}

func main() {
	err := startup()
	fmt.Println(err)
	// startup: reading config /nonexistent/app.yaml: open /nonexistent/app.yaml: no such file or directory

	// Despite two layers of wrapping, the root cause is still identifiable:
	fmt.Println(errors.Is(err, os.ErrNotExist)) // true
}
```

`%v` would produce the same *text* but sever the chain — `errors.Is` would
return `false`. Choose deliberately:

- `%w` when callers may reasonably want to inspect the cause.
- `%v` when the underlying error is an implementation detail you do not want
  to commit to as part of your API.

## `errors.Is` vs `errors.As`

Both walk the wrap chain; they answer different questions.

```go
errors.Is(err, ErrNotFound)  // "is this specific error value anywhere in the chain?"
errors.As(err, &target)      // "is there an error of this TYPE in the chain? give it to me"
```

Never use `err == ErrNotFound` on a possibly-wrapped error — it compares only
the outermost layer and silently returns `false`.

```go
package main

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("not found")

type QueryError struct {
	Query string
	Err   error
}

func (e *QueryError) Error() string { return "query " + e.Query + ": " + e.Err.Error() }

// Unwrap makes this type participate in the errors.Is/As chain.
func (e *QueryError) Unwrap() error { return e.Err }

func run() error {
	inner := &QueryError{Query: "SELECT * FROM users", Err: ErrNotFound}
	return fmt.Errorf("handler: %w", inner)
}

func main() {
	err := run()

	fmt.Println(err == ErrNotFound)          // false -- naive comparison fails
	fmt.Println(errors.Is(err, ErrNotFound)) // true  -- walks the whole chain

	var qe *QueryError
	if errors.As(err, &qe) {
		fmt.Println("failing query:", qe.Query) // failing query: SELECT * FROM users
	}
}
```

The `Unwrap() error` method is what makes a custom type transparent to
`errors.Is`/`errors.As`. Without it the chain stops at your type.

## Joining multiple errors

Since Go 1.20, `errors.Join` combines several errors into one; `errors.Is`
matches against any of them. Perfect for validating every field and reporting
all problems at once instead of stopping at the first.

```go
package main

import (
	"errors"
	"fmt"
)

var (
	ErrNameRequired  = errors.New("name is required")
	ErrEmailRequired = errors.New("email is required")
)

func validateAll(name, email string) error {
	var errs []error
	if name == "" {
		errs = append(errs, ErrNameRequired)
	}
	if email == "" {
		errs = append(errs, ErrEmailRequired)
	}
	return errors.Join(errs...) // returns nil if errs is empty
}

func main() {
	err := validateAll("", "")
	fmt.Println(err)
	// name is required
	// email is required

	fmt.Println(errors.Is(err, ErrEmailRequired)) // true
}
```

`fmt.Errorf` also accepts multiple `%w` verbs, e.g.
`fmt.Errorf("%w and %w", err1, err2)`.

## Practical guidelines

- Add context at each layer, but do not repeat it. `"reading config: open
  x: no such file"` is useful; `"error: failed: error reading: error"` is not.
- Error strings are lowercase and have no trailing punctuation, because they
  get embedded into larger messages.
- Handle an error *or* return it — never both. Logging and returning produces
  duplicate noise at every level.
- Sentinels and exported error types are part of your public API. Changing
  them breaks callers.

## Cheat sheet

| Task | Syntax |
|------|--------|
| Sentinel error | `var ErrX = errors.New("x")` |
| Formatted error | `fmt.Errorf("bad id %d", id)` |
| Wrap with context | `fmt.Errorf("loading: %w", err)` |
| Compare to a sentinel | `errors.Is(err, ErrX)` |
| Extract a typed error | `var e *MyErr; errors.As(err, &e)` |
| Make a type unwrappable | `func (e *MyErr) Unwrap() error` |
| Peel one layer | `errors.Unwrap(err)` |
| Combine errors | `errors.Join(err1, err2)` |
| Custom error type | `func (e *MyErr) Error() string` |

## Related lessons

- Error basics and the `if err != nil` idiom:
  [Level 1, Module 8](../level-1/08-error-handling.md).
- `error` is an interface — see [Module 1](01-interfaces.md), including the
  nil-interface trap that bites custom error types especially hard.
- Testing error paths with `errors.Is` in table-driven tests:
  [Module 6](06-testing-package.md).

## Exercise

Write a tiny file-based key/value store with two sentinels (`ErrKeyNotFound`,
`ErrStoreClosed`) and one custom type `ParseError{Line int, Err error}` with an
`Unwrap` method. `Get(key string)` should return `ErrKeyNotFound` for a missing
key, and wrap any underlying `os` error with `fmt.Errorf("get %q: %w", key,
err)`. In `main`, exercise all three paths and use `errors.Is` for the
sentinels and `errors.As` to print the failing line number from `ParseError`.
Verify that `errors.Is(err, os.ErrNotExist)` still works through your wrapping.
