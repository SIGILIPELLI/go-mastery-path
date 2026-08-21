# 02 · Building REST APIs

[Level 2](../level-2/03-http-basics.md) touched `net/http` for simple servers.
This module builds a small but real JSON API — routing by method, decoding
request bodies, encoding responses, and the concurrency-safety concerns that
show up the moment a handler touches shared state.

## A minimal JSON API

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
)

type Task struct {
	ID   int    `json:"id"`
	Text string `json:"text"`
	Done bool   `json:"done"`
}

type store struct {
	mu     sync.Mutex
	nextID int
	tasks  map[int]Task
}

func newStore() *store {
	return &store{nextID: 1, tasks: make(map[int]Task)}
}

func (s *store) create(text string) Task {
	s.mu.Lock()
	defer s.mu.Unlock()
	t := Task{ID: s.nextID, Text: text}
	s.tasks[t.ID] = t
	s.nextID++
	return t
}

func (s *store) list() []Task {
	s.mu.Lock()
	defer s.mu.Unlock()
	out := make([]Task, 0, len(s.tasks))
	for _, t := range s.tasks {
		out = append(out, t)
	}
	return out
}

func tasksHandler(s *store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodGet:
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(s.list())
		case http.MethodPost:
			var in struct{ Text string `json:"text"` }
			if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
				http.Error(w, err.Error(), http.StatusBadRequest)
				return
			}
			t := s.create(in.Text)
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusCreated)
			json.NewEncoder(w).Encode(t)
		default:
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		}
	}
}

func main() {
	s := newStore()
	mux := http.NewServeMux()
	mux.HandleFunc("/tasks", tasksHandler(s))
	fmt.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

One `http.HandlerFunc` dispatches on `r.Method` — GET lists, POST creates.
The store's `sync.Mutex` matters because `net/http` runs each request in its
own goroutine: two POSTs arriving at once must not race on `nextID`.

Running it and hitting it with `curl` from another shell:

```console
$ go run . &
listening on :8080

$ curl -s -X POST localhost:8080/tasks -d '{"text":"buy milk"}'
{"id":1,"text":"buy milk","done":false}

$ curl -s -X POST localhost:8080/tasks -d '{"text":"write report"}'
{"id":2,"text":"write report","done":false}

$ curl -s localhost:8080/tasks
[{"id":2,"text":"write report","done":false},{"id":1,"text":"buy milk","done":false}]

$ curl -s -o /dev/null -w "%{http_code}\n" -X DELETE localhost:8080/tasks
405
```

Two real traps hide here. First, `s.list()` ranges over a Go map, so the
JSON array order is not the insertion order — task 2 came back before task 1
above; never rely on map iteration order for anything user-visible. Second,
forgetting `w.Header().Set("Content-Type", ...)` *before* `w.WriteHeader` (or
before the first `Write`) silently sends `text/plain` — headers are locked in
the instant the first byte of the body goes out.

## Routing with path parameters

Go 1.22 added method- and wildcard-aware patterns to `http.ServeMux`, so you
often don't need a third-party router for simple path params:

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		fmt.Fprintf(w, "task id = %s\n", id)
	})
	http.ListenAndServe(":8081", mux)
}
```

```console
$ curl -s localhost:8081/tasks/42
task id = 42
```

`{id}` is captured by name and read back with `r.PathValue("id")` — no
regexp, no external dependency, and the method prefix (`GET `) means a POST
to the same path falls through to a 404 instead of your handler.

## Middleware as function wrapping

Cross-cutting concerns — logging, auth, recovery — wrap a handler rather than
being called inside it:

```go
func withLogging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
	})
}

// usage: http.ListenAndServe(":8080", withLogging(mux))
```

Because `withLogging` takes and returns `http.Handler`, middlewares compose
by nesting: `withRecover(withLogging(mux))` runs recover-then-log around
every request without either middleware knowing the other exists.

## Go-specific traps

- **`http.ListenAndServe` blocks forever on success** — it only returns on
  error, which is why it's almost always wrapped in `log.Fatal`.
- **The default `http.Client` has no timeout.** A hung server on the other
  end hangs your program forever; always set `http.Client{Timeout: ...}` for
  outbound calls.
- **Reusing `http.ResponseWriter` after the handler returns is a bug** the
  compiler cannot catch — it belongs to that one request only.
- **`json.NewDecoder(r.Body).Decode` does not enforce required fields** — a
  missing `"text"` key just leaves the Go field at its zero value silently.
- **Not closing `r.Body`** on the client side of a `net/http` request leaks
  the underlying connection; `defer resp.Body.Close()` is not optional.

## Cheat sheet

| Task | API |
|---|---|
| Serve JSON | `w.Header().Set("Content-Type","application/json")` then `json.NewEncoder(w).Encode(v)` |
| Parse JSON body | `json.NewDecoder(r.Body).Decode(&v)` |
| Path parameter (Go 1.22+) | `mux.HandleFunc("GET /x/{id}", ...)`, `r.PathValue("id")` |
| Set status code | `w.WriteHeader(http.StatusCreated)` — before any `Write` |
| Method-not-allowed | `http.Error(w, "...", http.StatusMethodNotAllowed)` |
| Wrap handlers | `func(next http.Handler) http.Handler` |
| Concurrency-safe state | Guard shared maps/slices with `sync.Mutex` |

## Related lessons

- HTTP fundamentals: [Level 2, Module 3](../level-2/03-http-basics.md).
- Cancelling slow handlers with request contexts:
  [Module 4](04-context-package.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) extends this exact
  handler shape onto a real SQLite-backed store.

## Exercise

Extend the `Task` API with `PATCH /tasks/{id}` that flips `Done` to `true`
and `DELETE /tasks/{id}` that removes the task, both using the Go 1.22
`{id}` pattern and `r.PathValue`. Return 404 via `http.Error` when the ID
doesn't exist. Add a `withLogging` middleware and confirm with `curl -v`
that every request, including the 404s, gets logged with its method, path,
and duration.
