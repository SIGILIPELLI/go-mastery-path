# 06 · Structs

## Defining and creating a struct

A `struct` groups related fields into a single named type — Go's equivalent
of a lightweight class body without inheritance.

```go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
	City string
}

func main() {
	// Field names -- order doesn't matter, all fields required unless omitted
	p1 := Person{Name: "Alice", Age: 30, City: "Boston"}

	// Positional -- order MUST match the struct definition exactly
	p2 := Person{"Bob", 25, "Chicago"}

	// Zero-valued struct, filled in afterward
	var p3 Person
	p3.Name = "Carol"
	p3.Age = 28

	fmt.Println(p1, p2, p3)
	fmt.Println(p1.Name, p1.Age) // Alice 30
}
```

## Structs are value types

Assigning or passing a struct copies it — unlike in Java or Python where
objects are references. This matters constantly:

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

func tryToModify(p Point) {
	p.X = 999 // modifies the COPY, not the caller's original
}

func main() {
	original := Point{X: 1, Y: 2}
	tryToModify(original)
	fmt.Println(original) // {1 2} -- unchanged

	copy := original
	copy.X = 100
	fmt.Println(original, copy) // {1 2} {100 2} -- independent
}
```

To actually mutate the caller's struct, pass a pointer (see
[Module 7](07-pointers.md)).

## Nested structs

```go
package main

import "fmt"

type Address struct {
	Street string
	City   string
}

type Employee struct {
	Name    string
	Address Address // struct field inside a struct
}

func main() {
	e := Employee{
		Name: "Dana",
		Address: Address{
			Street: "1 Main St",
			City:   "Denver",
		},
	}

	fmt.Println(e.Address.City) // Denver
	e.Address.City = "Boulder"  // dotted access reaches into nested fields
	fmt.Println(e)
}
```

## Anonymous fields (embedding) and struct methods

Embedding a type gives the outer struct direct access to the embedded
type's fields and methods -- Go's alternative to classical inheritance,
covered in depth in [Level 2](../level-2/03-methods-receivers.md).

```go
package main

import "fmt"

type Animal struct {
	Name string
}

func (a Animal) Speak() string {
	return a.Name + " makes a sound"
}

type Dog struct {
	Animal // embedded -- Dog "has" all of Animal's fields/methods promoted
	Breed  string
}

func main() {
	d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
	fmt.Println(d.Name)    // Rex -- promoted field, no need for d.Animal.Name
	fmt.Println(d.Speak()) // Rex makes a sound -- promoted method
}
```

## Struct tags and comparing structs

```go
package main

import "fmt"

type Config struct {
	Host string `json:"host"` // tags are metadata, read by reflection-based
	Port int    `json:"port"` // packages like encoding/json (Level 2)
}

func main() {
	a := Config{Host: "localhost", Port: 8080}
	b := Config{Host: "localhost", Port: 8080}

	// Structs with only comparable fields support == directly
	fmt.Println(a == b) // true -- compares field by field
}
```

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Define | `type Person struct { Name string; Age int }` |
| Create (named fields) | `Person{Name: "A", Age: 1}` |
| Create (positional) | `Person{"A", 1}` |
| Zero value | `var p Person` |
| Access/set field | `p.Name`, `p.Name = "X"` |
| Nested access | `e.Address.City` |
| Embedding | `type Dog struct { Animal; Breed string }` |
| Compare (comparable fields) | `a == b` |
| Struct tag | `` `json:"host"` `` |

## Exercise

Define a `Book` struct with `Title`, `Author`, and `Pages` fields. Write a
function `isLong(b Book) bool` that returns whether `Pages > 300`. In
`main`, create a slice of three `Book` values and loop over them, printing
each title along with whether it's "long."
