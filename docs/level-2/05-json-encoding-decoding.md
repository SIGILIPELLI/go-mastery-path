# 05 · JSON Encoding/Decoding

## Why `encoding/json` matters

JSON is the lingua franca of web APIs, config files and log pipelines. Go's
`encoding/json` package converts between Go values and JSON text using
reflection, driven by struct tags — no code generation, no schema files. Get
comfortable here and you can consume any REST API (which is exactly what
[Module 8](08-net-http-client.md) and the
[Level 2 project](10-project-weather-cli.md) do).

Two directions, two verbs:

- **Marshal** — Go value → JSON bytes (also called encoding, serialising).
- **Unmarshal** — JSON bytes → Go value (decoding, deserialising).

## Marshalling: Go to JSON

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	Name  string
	Email string
	Age   int
	admin bool // unexported -- INVISIBLE to encoding/json
}

func main() {
	u := User{Name: "Ada", Email: "ada@example.com", Age: 36, admin: true}

	data, err := json.Marshal(u)
	if err != nil {
		fmt.Println("marshal error:", err)
		return
	}
	fmt.Println(string(data))
	// {"Name":"Ada","Email":"ada@example.com","Age":36}

	// MarshalIndent for human-readable output.
	pretty, _ := json.MarshalIndent(u, "", "  ")
	fmt.Println(string(pretty))
	// {
	//   "Name": "Ada",
	//   "Email": "ada@example.com",
	//   "Age": 36
	// }
}
```

Two rules to lock in right now:

1. **Only exported fields are encoded.** `admin` is lowercase, so
   `encoding/json` cannot see it — no error, it just silently disappears.
   This is the single most common JSON bug in Go.
2. `Marshal` returns `[]byte`, so `string(data)` is needed to print it.

## Struct tags: controlling the JSON shape

Go field names are capitalised; JSON keys usually are not. Struct tags bridge
the gap:

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Product struct {
	ID          int      `json:"id"`
	Name        string   `json:"name"`
	Price       float64  `json:"price"`
	Description string   `json:"description,omitempty"` // dropped when ""
	Tags        []string `json:"tags,omitempty"`        // dropped when nil/empty
	Internal    string   `json:"-"`                     // never encoded or decoded
	StockCount  int      `json:"stock,string"`          // encoded as a JSON string
}

func main() {
	p := Product{ID: 1, Name: "Mug", Price: 9.99, Internal: "secret", StockCount: 42}

	data, _ := json.MarshalIndent(p, "", "  ")
	fmt.Println(string(data))
	// {
	//   "id": 1,
	//   "name": "Mug",
	//   "price": 9.99,
	//   "stock": "42"
	// }
}
```

| Tag | Effect |
|-----|--------|
| `json:"name"` | use `name` as the JSON key |
| `json:"name,omitempty"` | omit when the field is the zero value |
| `json:"-"` | never encode or decode this field |
| `json:"-,"` | use the literal key `-` |
| `json:",omitempty"` | keep the Go field name, but allow omission |
| `json:"n,string"` | encode a number/bool as a JSON string |

Watch out: `omitempty` treats `0`, `false` and `""` as empty. A `Quantity int`
with `omitempty` will vanish when it is legitimately zero. Use `*int` when you
need to distinguish "zero" from "absent".

## Unmarshalling: JSON to Go

`Unmarshal` needs a **pointer** so it has something to write into:

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

func main() {
	input := `{"id": 7, "name": "Notebook", "price": 3.50, "colour": "blue"}`

	var p Product
	if err := json.Unmarshal([]byte(input), &p); err != nil { // note the &
		fmt.Println("unmarshal error:", err)
		return
	}
	fmt.Printf("%+v\n", p)
	// {ID:7 Name:Notebook Price:3.5}
}
```

Note what did **not** happen: the unknown key `colour` was silently ignored,
and no error was raised. Field matching is case-insensitive, so `"NAME"`,
`"name"` and `"Name"` all fill `Name` — the exact tag match wins if there is
one.

Missing JSON keys leave the Go field at its zero value, again with no error.
If absence must be detectable, use a pointer:

```go
type Settings struct {
	Debug *bool `json:"debug"` // nil = key absent, &false = explicitly false
}
```

## Nested structs and slices

Real API payloads nest. Model them with nested types:

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Address struct {
	City    string `json:"city"`
	Country string `json:"country"`
}

type Employee struct {
	Name    string   `json:"name"`
	Address Address  `json:"address"`
	Skills  []string `json:"skills"`
}

type Company struct {
	Name      string     `json:"name"`
	Employees []Employee `json:"employees"`
}

func main() {
	input := `{
	  "name": "Acme",
	  "employees": [
	    {"name": "Ada", "address": {"city": "London", "country": "UK"},
	     "skills": ["go", "math"]},
	    {"name": "Grace", "address": {"city": "Boston", "country": "US"},
	     "skills": ["compilers"]}
	  ]
	}`

	var c Company
	if err := json.Unmarshal([]byte(input), &c); err != nil {
		fmt.Println("error:", err)
		return
	}

	for _, e := range c.Employees {
		fmt.Printf("%s (%s, %s) knows %v\n",
			e.Name, e.Address.City, e.Address.Country, e.Skills)
	}
	// Ada (London, UK) knows [go math]
	// Grace (Boston, US) knows [compilers]
}
```

You only need to declare the fields you actually care about — everything else
in the payload is skipped. For a large API response, model the three fields
you use, not all forty.

## Dynamic JSON with maps and `any`

When the shape genuinely is not known ahead of time, decode into
`map[string]any`. Values arrive as a small fixed set of Go types:

| JSON | Go type in `any` |
|------|------------------|
| object | `map[string]any` |
| array | `[]any` |
| string | `string` |
| number | `float64` (**always** — even integers) |
| `true`/`false` | `bool` |
| `null` | `nil` |

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	input := `{"name": "Ada", "age": 36, "active": true, "scores": [1, 2, 3]}`

	var m map[string]any
	if err := json.Unmarshal([]byte(input), &m); err != nil {
		fmt.Println("error:", err)
		return
	}

	// Every value needs a type assertion (see Module 1).
	name := m["name"].(string)
	age := m["age"].(float64) // NOT int -- JSON numbers decode to float64
	fmt.Printf("%s is %d\n", name, int(age)) // Ada is 36

	for k, v := range m {
		fmt.Printf("%s: %v (%T)\n", k, v, v)
	}
}
```

The `float64` rule catches everyone: `m["age"].(int)` panics. Prefer a struct
whenever you know the shape — you get real types and compile-time safety.

## Streaming with Encoder and Decoder

`json.Marshal` builds the entire document in memory. For files, HTTP bodies
and anything large, use the streaming API, which works on any `io.Reader` or
`io.Writer`:

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"strings"
)

type Event struct {
	Type string `json:"type"`
	User string `json:"user"`
}

func main() {
	// Decoder: read a stream of JSON values.
	stream := `{"type":"login","user":"ada"}
{"type":"logout","user":"grace"}`

	dec := json.NewDecoder(strings.NewReader(stream))
	dec.DisallowUnknownFields() // strict mode: error on unexpected keys

	for {
		var e Event
		if err := dec.Decode(&e); err != nil {
			break // io.EOF ends the loop
		}
		fmt.Printf("%s by %s\n", e.Type, e.User)
	}
	// login by ada
	// logout by grace

	// Encoder: write straight to a Writer (stdout, a file, an HTTP response).
	enc := json.NewEncoder(os.Stdout)
	enc.SetIndent("", "  ")
	_ = enc.Encode(Event{Type: "ping", User: "system"})
}
```

`DisallowUnknownFields()` flips the default silence into a hard error — very
useful for config files where a typo'd key should not be quietly ignored.

## Custom marshalling

Implement `json.Marshaler`/`json.Unmarshaler` when a type needs a
non-default representation — a timestamp in a specific layout, for example:

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
	"time"
)

type Date struct{ time.Time }

const dateLayout = "2006-01-02"

func (d Date) MarshalJSON() ([]byte, error) {
	return []byte(`"` + d.Format(dateLayout) + `"`), nil
}

func (d *Date) UnmarshalJSON(b []byte) error {
	s := strings.Trim(string(b), `"`)
	t, err := time.Parse(dateLayout, s)
	if err != nil {
		return fmt.Errorf("parsing date %q: %w", s, err)
	}
	d.Time = t
	return nil
}

type Task struct {
	Title string `json:"title"`
	Due   Date   `json:"due"`
}

func main() {
	var t Task
	_ = json.Unmarshal([]byte(`{"title":"Ship it","due":"2026-03-15"}`), &t)
	fmt.Println(t.Title, t.Due.Year()) // Ship it 2026

	out, _ := json.Marshal(t)
	fmt.Println(string(out)) // {"title":"Ship it","due":"2026-03-15"}
}
```

Note the receivers: `MarshalJSON` on the value, `UnmarshalJSON` on the pointer
(it must modify the receiver) — exactly the method-set rules from
[Module 3](03-methods-receivers.md).

## Cheat sheet

| Task | Syntax |
|------|--------|
| Struct → JSON | `data, err := json.Marshal(v)` |
| Pretty JSON | `json.MarshalIndent(v, "", "  ")` |
| JSON → struct | `json.Unmarshal(data, &v)` |
| Read from a stream | `json.NewDecoder(r).Decode(&v)` |
| Write to a stream | `json.NewEncoder(w).Encode(v)` |
| Reject unknown keys | `dec.DisallowUnknownFields()` |
| Rename a key | `` `json:"user_id"` `` |
| Omit zero values | `` `json:"bio,omitempty"` `` |
| Skip a field | `` `json:"-"` `` |
| Unknown shape | `var m map[string]any` |
| Validate JSON text | `json.Valid(data)` |

## Related lessons

- Structs and tags: [Level 1, Module 6](../level-1/06-structs.md).
- Type assertions for `map[string]any`: [Module 1](01-interfaces.md).
- Reading and writing JSON files: [Module 7](07-file-io.md).
- Decoding API responses over the wire: [Module 8](08-net-http-client.md).

## Exercise

Define a `Book` struct with `Title`, `Author`, `Year`, `ISBN` and
`Tags []string`, tagged so it encodes as `title`, `author`, `year`, `isbn` and
`tags`, with `isbn` and `tags` omitted when empty. Marshal a slice of three
books to indented JSON and print it. Then unmarshal a JSON array string
(including one book with an unknown extra key and one missing `year`) back into
`[]Book` and print each one, confirming the unknown key is ignored and the
missing year becomes `0`. Finally, re-decode the same input with a
`json.Decoder` using `DisallowUnknownFields()` and print the resulting error.
