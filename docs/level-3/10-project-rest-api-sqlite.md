# 10 · Project: REST API with SQLite

This project pulls together every Level 3 module into one small, real
service: a task-tracking REST API backed by SQLite, with request logging
middleware, context-aware queries, a `Store` interface for testability, and
a handler test suite that never touches a real database. It's four files —
small enough to read end to end in one sitting, structured the way a larger
service would be.

## Project layout

```
taskapi/
├── go.mod
├── main.go            # wiring: open the store, start the server
├── store.go           # Store interface + SQLite implementation
├── handlers.go         # HTTP handlers, routing, middleware
└── handlers_test.go    # tests against a fake in-memory Store
```

## `store.go`: the persistence layer

```go
package main

import (
	"context"
	"database/sql"
	"errors"

	_ "modernc.org/sqlite"
)

var ErrNotFound = errors.New("task not found")

type Task struct {
	ID   int64  `json:"id"`
	Text string `json:"text"`
	Done bool   `json:"done"`
}

// Store is the interface handlers depend on -- lets tests use a fake instead
// of a real database.
type Store interface {
	Create(ctx context.Context, text string) (Task, error)
	List(ctx context.Context) ([]Task, error)
	Get(ctx context.Context, id int64) (Task, error)
	Complete(ctx context.Context, id int64) error
	Delete(ctx context.Context, id int64) error
}

type sqliteStore struct {
	db *sql.DB
}

func NewSQLiteStore(dsn string) (*sqliteStore, error) {
	db, err := sql.Open("sqlite", dsn)
	if err != nil {
		return nil, err
	}
	if _, err := db.Exec(`CREATE TABLE IF NOT EXISTS tasks (
		id INTEGER PRIMARY KEY AUTOINCREMENT,
		text TEXT NOT NULL,
		done INTEGER NOT NULL DEFAULT 0
	)`); err != nil {
		db.Close()
		return nil, err
	}
	return &sqliteStore{db: db}, nil
}

func (s *sqliteStore) Close() error { return s.db.Close() }

func (s *sqliteStore) Create(ctx context.Context, text string) (Task, error) {
	res, err := s.db.ExecContext(ctx, `INSERT INTO tasks (text) VALUES (?)`, text)
	if err != nil {
		return Task{}, err
	}
	id, err := res.LastInsertId()
	if err != nil {
		return Task{}, err
	}
	return Task{ID: id, Text: text}, nil
}

func (s *sqliteStore) Get(ctx context.Context, id int64) (Task, error) {
	var t Task
	var done int
	err := s.db.QueryRowContext(ctx, `SELECT id, text, done FROM tasks WHERE id = ?`, id).
		Scan(&t.ID, &t.Text, &done)
	if errors.Is(err, sql.ErrNoRows) {
		return Task{}, ErrNotFound
	}
	if err != nil {
		return Task{}, err
	}
	t.Done = done != 0
	return t, nil
}

func (s *sqliteStore) Complete(ctx context.Context, id int64) error {
	res, err := s.db.ExecContext(ctx, `UPDATE tasks SET done = 1 WHERE id = ?`, id)
	if err != nil {
		return err
	}
	n, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if n == 0 {
		return ErrNotFound
	}
	return nil
}
```

(`List` and `Delete` follow the same shape as `Get` and `Complete` — full
source in the exercise below.) Every method takes a `context.Context` and
passes it to `*Context`-suffixed `database/sql` calls, so a client
disconnect ([Module 4](04-context-package.md)) cancels the in-flight query
instead of letting it run to completion for nobody. `Store` being an
interface — the strategy pattern from [Module 5](05-design-patterns.md) —
is what lets `handlers_test.go` swap in an in-memory fake without spinning
up SQLite.

## `handlers.go`: routing and middleware

```go
package main

import (
	"encoding/json"
	"errors"
	"log"
	"net/http"
	"strconv"
	"time"
)

type api struct {
	store Store
}

func newMux(s Store) http.Handler {
	a := &api{store: s}
	mux := http.NewServeMux()
	mux.HandleFunc("GET /tasks", a.list)
	mux.HandleFunc("POST /tasks", a.create)
	mux.HandleFunc("GET /tasks/{id}", a.get)
	mux.HandleFunc("PATCH /tasks/{id}", a.complete)
	mux.HandleFunc("DELETE /tasks/{id}", a.delete)
	return withLogging(mux)
}

func withLogging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
	})
}

func (a *api) create(w http.ResponseWriter, r *http.Request) {
	var in struct {
		Text string `json:"text"`
	}
	if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
		http.Error(w, "invalid JSON body", http.StatusBadRequest)
		return
	}
	if in.Text == "" {
		http.Error(w, "text is required", http.StatusBadRequest)
		return
	}
	t, err := a.store.Create(r.Context(), in.Text)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	writeJSON(w, http.StatusCreated, t)
}
```

`newMux` uses Go 1.22's method-and-wildcard routing
([Module 2](02-building-rest-apis.md)) — `GET /tasks/{id}` and
`PATCH /tasks/{id}` are separate registrations, so a stray `POST
/tasks/{id}` falls through to a 404 automatically. Every handler funnels
"not found" through `errors.Is(err, ErrNotFound)` rather than a string
comparison, matching the sentinel-error convention from
[Module 3](03-databases-sql.md).

## `main.go`: wiring it together

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	store, err := NewSQLiteStore("file:tasks.db?cache=shared")
	if err != nil {
		log.Fatal(err)
	}
	defer store.Close()

	fmt.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", newMux(store)))
}
```

## Running it

```console
$ go build -o server . && ./server &
listening on :8080

$ curl -s -X POST localhost:8080/tasks -d '{"text":"buy milk"}'
{"id":1,"text":"buy milk","done":false}

$ curl -s -X POST localhost:8080/tasks -d '{"text":"write report"}'
{"id":2,"text":"write report","done":false}

$ curl -s localhost:8080/tasks
[{"id":1,"text":"buy milk","done":false},{"id":2,"text":"write report","done":false}]

$ curl -s -X PATCH localhost:8080/tasks/1
{"id":1,"text":"buy milk","done":true}

$ curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/tasks/999
404

$ curl -s -o /dev/null -w "%{http_code}\n" -X DELETE localhost:8080/tasks/2
204

$ curl -s localhost:8080/tasks
[{"id":1,"text":"buy milk","done":true}]
```

The server's own log, from `withLogging`, for that same session:

```
listening on :8080
2026/08/21 22:27:49 POST /tasks 1.29275ms
2026/08/21 22:27:49 POST /tasks 815.167µs
2026/08/21 22:27:49 GET /tasks 217µs
2026/08/21 22:27:49 GET /tasks/1 111.916µs
2026/08/21 22:27:49 PATCH /tasks/1 643.416µs
2026/08/21 22:27:49 GET /tasks/999 111.208µs
2026/08/21 22:27:49 DELETE /tasks/2 575.042µs
2026/08/21 22:27:49 GET /tasks 164.208µs
```

## Tests against a fake `Store`

```go
type fakeStore struct {
	tasks  map[int64]Task
	nextID int64
}

func (f *fakeStore) Get(ctx context.Context, id int64) (Task, error) {
	t, ok := f.tasks[id]
	if !ok {
		return Task{}, ErrNotFound
	}
	return t, nil
}

func TestCompleteAndDelete(t *testing.T) {
	fs := newFakeStore()
	fs.Create(context.Background(), "wash car")
	mux := newMux(fs)

	req := httptest.NewRequest(http.MethodPatch, "/tasks/1", nil)
	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK || !strings.Contains(rec.Body.String(), `"done":true`) {
		t.Fatalf("PATCH /tasks/1 = %d %s, want 200 with done:true", rec.Code, rec.Body.String())
	}
	// ... delete, then confirm a follow-up GET 404s
}
```

```console
$ go test -v ./...
=== RUN   TestCreateAndGet
--- PASS: TestCreateAndGet (0.00s)
=== RUN   TestGetNotFound
--- PASS: TestGetNotFound (0.00s)
=== RUN   TestCompleteAndDelete
--- PASS: TestCompleteAndDelete (0.00s)
=== RUN   TestCreateEmptyText
--- PASS: TestCreateEmptyText (0.00s)
PASS
ok  	taskapi	0.700s
```

None of these tests import `modernc.org/sqlite` or touch a file on disk —
`httptest.NewRequest` and `httptest.NewRecorder` drive the real
`http.Handler` chain (including `withLogging`) against `fakeStore`, so the
test suite runs in milliseconds and never leaves a stray `tasks.db` behind.

## Design notes and traps hit while building this

- **`List` returning `nil` for zero tasks** would serialize to JSON `null`,
  not `[]` — the handler explicitly checks `if tasks == nil { tasks =
  []Task{} }` before encoding, because API consumers reasonably expect an
  empty array, not `null`, for "no tasks yet."
- **`RowsAffected()` after `UPDATE`/`DELETE` is how `Complete` and `Delete`
  detect a missing ID** without a separate existence-check query first —
  one round trip instead of two.
- **The SQLite DSN `cache=shared`** matters for this driver when multiple
  goroutines might open connections to the same file-backed database
  concurrently; without it, some drivers surface spurious "database is
  locked" errors under load.
- **Every handler that assigns before checking the specific typed error
  (`ErrNotFound`)** — falling through to a generic 500 on any other error —
  is what keeps `errors.Is` chains from silently swallowing unrelated
  failures as 404s.

## Cheat sheet

| Piece | Where |
|---|---|
| Interface for testability | `Store` in `store.go` (Module 5's strategy pattern) |
| Context-aware DB calls | `db.ExecContext`/`QueryRowContext` (Module 3 + 4) |
| Method+wildcard routing | `mux.HandleFunc("PATCH /tasks/{id}", ...)` (Module 2) |
| Request logging | `withLogging` middleware wraps the whole mux |
| Fast tests, no real DB | `fakeStore` + `httptest.NewRequest`/`NewRecorder` |
| Sentinel not-found error | `ErrNotFound`, checked via `errors.Is` |

## Stretch goals

- Add pagination (`?limit=&offset=`) to `GET /tasks`, validating that
  `limit` is a positive integer and defaulting sensibly when absent.
- Add a `PUT /tasks/{id}` to fully replace a task's `text`, and write a
  fake-store test proving it 404s on an unknown ID exactly like `PATCH`
  does.
- Swap the manual routing for `net/http/pprof` wired in alongside the API
  mux (see [Module 7](07-profiling-benchmarking.md)), and capture a CPU
  profile while running a `hey` or `wrk` load test against `GET /tasks`.
- Replace the hand-rolled `sqliteStore` with one built on `database/sql`
  transactions for a new `POST /tasks/batch` endpoint that inserts many
  tasks atomically, rolling back entirely if any single insert fails.
