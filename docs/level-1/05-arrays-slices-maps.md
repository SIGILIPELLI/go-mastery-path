# 05 · Arrays, Slices & Maps

## Arrays: fixed size, rarely used directly

An array's length is part of its type — `[3]int` and `[5]int` are different
types entirely. Because of this rigidity, arrays are uncommon in everyday Go
code; slices (below) are what you'll actually reach for.

```go
package main

import "fmt"

func main() {
	var nums [3]int          // [0 0 0] -- zero-valued
	nums[0] = 10
	nums[1] = 20
	nums[2] = 30

	fruits := [3]string{"apple", "banana", "cherry"}
	sized := [...]string{"a", "b"} // let the compiler count: [2]string

	fmt.Println(nums, fruits, sized, len(fruits))
}
```

## Slices: the workhorse collection

A slice is a flexible, growable view over an underlying array — three
words under the hood: a pointer, a length, and a capacity.

```go
package main

import "fmt"

func main() {
	// Slice literal
	fruits := []string{"apple", "banana", "cherry"}
	fmt.Println(fruits, len(fruits)) // [apple banana cherry] 3

	// make(type, length, capacity)
	scores := make([]int, 0, 10) // len 0, capacity 10 -- avoids reallocation

	// append grows the slice, returns a (possibly new) slice
	scores = append(scores, 90, 85, 77)
	fmt.Println(scores, len(scores), cap(scores))

	// slicing: [low:high), high is exclusive
	fmt.Println(fruits[0:2]) // [apple banana]
	fmt.Println(fruits[1:])  // [banana cherry]
	fmt.Println(fruits[:2])  // [apple banana]
}
```

### Slices share underlying arrays

Slicing does **not** copy data — two slices can point at the same
backing array, so mutating one can affect the other:

```go
package main

import "fmt"

func main() {
	original := []int{1, 2, 3, 4, 5}
	view := original[1:4] // [2 3 4], shares memory with original

	view[0] = 99
	fmt.Println(original) // [1 99 3 4 5] -- original changed too!

	// To get an independent copy, use copy()
	independent := make([]int, len(view))
	copy(independent, view)
	independent[0] = -1
	fmt.Println(view) // unaffected: [99 3 4]
}
```

### Two-dimensional slices

```go
package main

import "fmt"

func main() {
	grid := make([][]int, 3) // 3 rows
	for i := range grid {
		grid[i] = make([]int, 3) // each row: 3 columns
	}
	grid[1][1] = 5

	for _, row := range grid {
		fmt.Println(row)
	}
	// [0 0 0]
	// [0 5 0]
	// [0 0 0]
}
```

## Maps: key-value pairs

```go
package main

import "fmt"

func main() {
	// Map literal
	ages := map[string]int{
		"Alice": 30,
		"Bob":   25,
	}

	// make() for an empty map
	scores := make(map[string]int)
	scores["Carol"] = 88
	scores["Dave"] = 92

	fmt.Println(ages["Alice"]) // 30

	// the "comma ok" idiom -- check whether a key exists
	value, ok := ages["Eve"]
	fmt.Println(value, ok) // 0 false -- zero value if missing, ok tells you why

	// delete a key
	delete(scores, "Dave")

	// iterate -- order is NOT guaranteed
	for name, score := range scores {
		fmt.Println(name, score)
	}

	fmt.Println(len(scores))
}
```

The comma-ok idiom (`value, ok := m[key]`) is essential: reading a missing
key returns the zero value silently, so `ok` is the only reliable way to
distinguish "key present with zero value" from "key absent."

## Cheat sheet

| Operation | Syntax |
|-----------|--------|
| Array (fixed size) | `var a [3]int` |
| Slice literal | `s := []int{1, 2, 3}` |
| Slice with make | `s := make([]int, len, cap)` |
| Append | `s = append(s, 4, 5)` |
| Slice a slice | `s[1:3]` |
| Copy independently | `copy(dst, src)` |
| Map literal | `m := map[string]int{"a": 1}` |
| Map with make | `m := make(map[string]int)` |
| Read + existence check | `v, ok := m[key]` |
| Delete a key | `delete(m, key)` |
| Length (slice or map) | `len(s)` |

## Exercise

Write a program that builds a `map[string]int` counting word frequency in a
`[]string` of words (some repeated). Iterate the map to print each word and
its count, then use `append` to build a `[]string` slice containing only the
words that appear more than once, using the comma-ok idiom where useful.
