# 10 · Capstone: A Production-Shaped URL Shortener

This capstone combines every Level 4 module into one small real service: a
URL shortener with a SQLite-backed store, per-IP rate limiting, structured
logging, panic recovery, graceful shutdown, and a test suite that never
touches the database. It's deliberately the same shape as a service you'd
actually deploy — small enough to read end to end, structured the way a
larger one would be.

## Project layout

```
shortener/
├── go.mod
├── main.go            # wiring: store, server, signal handling, shutdown
├── store.go           # Store interface + SQLite implementation
├── handlers.go         # routes, rate limiting, logging, panic recovery
└── handlers_test.go    # tests against a fake in-memory Store
```

## `store.go`: persistence and code generation

```go
package main

import (
	"context"
	"crypto/rand"
	"database/sql"
	"encoding/base64"
	"errors"

	_ "modernc.org/sqlite"
)

var ErrNotFound = errors.New("short code not found")

type Link struct {
	Code string `json:"code"`
	URL  string `json:"url"`
	Hits int    `json:"hits"`
}

type Store interface {
	Create(ctx context.Context, url string) (Link, error)
	Resolve(ctx context.Context, code string) (Link, error)
	RecordHit(ctx context.Context, code string) error
}

func randomCode() (string, error) {
	buf := make([]byte, 6)
	if _, err := rand.Read(buf); err != nil { // crypto/rand -- Module 8
		return "", err
	}
	return base64.RawURLEncoding.EncodeToString(buf), nil
}

func (s *sqliteStore) Create(ctx context.Context, url string) (Link, error) {
	code, err := randomCode()
	if err != nil {
		return Link{}, err
	}
	if _, err := s.db.ExecContext(ctx, `INSERT INTO links (code, url) VALUES (?, ?)`, code, url); err != nil {
		return Link{}, err
	}
	return Link{Code: code, URL: url}, nil
}

func (s *sqliteStore) RecordHit(ctx context.Context, code string) error {
	res, err := s.db.ExecContext(ctx, `UPDATE links SET hits = hits + 1 WHERE code = ?`, code)
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

`randomCode` uses `crypto/rand`, not `math/rand` — [Module 8](08-security-best-practices.md)'s
trap about predictable short codes applies directly here: a guessable
short-link generator lets an attacker enumerate other users' links.

## `handlers.go`: rate limiting, logging, and recovery in one chain

```go
type api struct {
	store    Store
	logger   *slog.Logger
	limiters sync.Map // ip string -> *rate.Limiter -- Module 1's sync.Map
}

func (a *api) limiterFor(ip string) *rate.Limiter {
	if v, ok := a.limiters.Load(ip); ok {
		return v.(*rate.Limiter)
	}
	l := rate.NewLimiter(5, 5) // 5 req/sec, burst 5, per IP
	actual, _ := a.limiters.LoadOrStore(ip, l)
	return actual.(*rate.Limiter)
}

func (a *api) withRateLimit(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if !a.limiterFor(r.RemoteAddr).Allow() {
			http.Error(w, `{"error":"rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func (a *api) withLogging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: 200}
		defer func() {
			if rv := recover(); rv != nil {
				a.logger.Error("panic recovered", slog.Any("panic", rv))
				http.Error(w, `{"error":"internal error"}`, http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(rec, r)
		a.logger.Info("request",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", rec.status),
			slog.Float64("duration_ms", float64(time.Since(start).Microseconds())/1000),
		)
	})
}

func newMux(a *api) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", a.health)
	mux.HandleFunc("POST /links", a.create)
	mux.HandleFunc("GET /links/{code}/stats", a.stats)
	mux.HandleFunc("GET /r/{code}", a.redirect)
	return a.withLogging(a.withRateLimit(mux))
}
```

`a.limiters` is a `sync.Map` keyed by IP ([Module 1](01-advanced-concurrency.md))
because each client needs its own independent rate budget — a global
limiter would let one abusive client starve everyone else. Middleware
order matters: `withLogging` wraps `withRateLimit`, so even a
429-rate-limited request still gets logged (useful for spotting abuse
patterns), while the panic recovery inside `withLogging` protects the
*whole* chain including the rate limiter itself, following
[Module 4](04-production-grade-apis.md)'s recovery-middleware pattern with
the JSON error shape from the same module. Note also
`slog.Float64("duration_ms", ...)` here instead of `slog.Duration` —
directly applying the fix flagged as a trap in
[Module 7](07-observability.md).

## `main.go`: graceful shutdown

```go
srv := &http.Server{
	Addr:              ":8099",
	Handler:           newMux(a),
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}

go func() {
	fmt.Println("listening on", srv.Addr)
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		logger.Error("server failed", slog.Any("err", err))
		os.Exit(1)
	}
}()

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
<-stop

logger.Info("shutting down")
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
srv.Shutdown(ctx)
logger.Info("shutdown complete")
```

Exactly [Module 4](04-production-grade-apis.md)'s pattern: timeouts
configured explicitly, `ListenAndServe` in its own goroutine, `SIGTERM`
triggering a bounded graceful `Shutdown`.

## Running it

```console
$ go build -o srv . && ./srv &
listening on :8099

$ curl -s localhost:8099/healthz
{"status":"ok"}

$ curl -s -X POST localhost:8099/links -d '{"url":"https://go.dev"}'
{"code":"ItiYuAGV","url":"https://go.dev","hits":0}

$ curl -s -o /dev/null -w "%{http_code} -> %{redirect_url}\n" localhost:8099/r/ItiYuAGV
302 -> https://go.dev/

$ curl -s localhost:8099/links/ItiYuAGV/stats
{"code":"ItiYuAGV","url":"https://go.dev","hits":2}

$ curl -s -o /dev/null -w "%{http_code}\n" localhost:8099/r/zzzzzz
404
```

The server's structured logs for that exact session:

```json
{"time":"2026-08-21T22:42:06.427689+05:30","level":"INFO","msg":"request","method":"GET","path":"/healthz","status":200,"duration_ms":0.263}
{"time":"2026-08-21T22:42:06.443453+05:30","level":"INFO","msg":"request","method":"POST","path":"/links","status":201,"duration_ms":1.248}
{"time":"2026-08-21T22:42:06.474024+05:30","level":"INFO","msg":"request","method":"GET","path":"/r/ItiYuAGV","status":302,"duration_ms":0.753}
{"time":"2026-08-21T22:42:06.483071+05:30","level":"INFO","msg":"request","method":"GET","path":"/r/ItiYuAGV","status":302,"duration_ms":0.626}
{"time":"2026-08-21T22:42:06.490988+05:30","level":"INFO","msg":"request","method":"GET","path":"/links/ItiYuAGV/stats","status":200,"duration_ms":0.12}
{"time":"2026-08-21T22:42:06.498517+05:30","level":"INFO","msg":"request","method":"GET","path":"/r/zzzzzz","status":404,"duration_ms":0.091}
```

And a `SIGTERM` sent to the running process:

```
{"time":"2026-08-21T22:42:11.851423+05:30","level":"INFO","msg":"shutting down"}
{"time":"2026-08-21T22:42:11.851599+05:30","level":"INFO","msg":"shutdown complete"}
```

## Tests: fake store, real rate limiter

```go
func TestStatsAfterHits(t *testing.T) {
	_, mux := testMux()
	mux.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest(http.MethodPost, "/links", strings.NewReader(`{"url":"https://example.com"}`)))
	mux.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest(http.MethodGet, "/r/abc123", nil))
	mux.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest(http.MethodGet, "/r/abc123", nil))

	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, "/links/abc123/stats", nil))
	if !strings.Contains(rec.Body.String(), `"hits":2`) {
		t.Fatalf("stats body = %s, want hits:2", rec.Body.String())
	}
}

func TestRateLimit(t *testing.T) {
	_, mux := testMux()
	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	req.RemoteAddr = "10.0.0.1:1234"

	var lastCode int
	for i := 0; i < 10; i++ {
		rec := httptest.NewRecorder()
		mux.ServeHTTP(rec, req)
		lastCode = rec.Code
	}
	if lastCode != http.StatusTooManyRequests {
		t.Fatalf("after 10 rapid requests, last status = %d, want 429", lastCode)
	}
}
```

```console
$ go test -v ./...
=== RUN   TestCreateAndRedirect
--- PASS: TestCreateAndRedirect (0.00s)
=== RUN   TestRedirectNotFound
--- PASS: TestRedirectNotFound (0.00s)
=== RUN   TestStatsAfterHits
--- PASS: TestStatsAfterHits (0.00s)
=== RUN   TestRateLimit
--- PASS: TestRateLimit (0.00s)
PASS
ok  	shortener	0.759s
```

`TestRateLimit` exercises the *real* `rate.Limiter`, not a fake — with
burst 5 and 10 requests fired back to back with no elapsed time between
them, the 6th onward genuinely get refused, so the test proves the limiter
is wired into the middleware chain correctly rather than just asserting
mocked behavior.

## Design notes tying the levels together

- **Interface-based `Store`** (Level 3's strategy pattern) is what makes
  `handlers_test.go` possible without SQLite in the loop at all.
- **`context.Context` threaded through every store call** means a client
  disconnect cancels in-flight database work, not just in theory but
  because every `*Context`-suffixed `database/sql` method is used
  consistently.
- **`crypto/rand` over `math/rand`** for short codes is a security decision
  ([Module 8](08-security-best-practices.md)), not a style preference —
  predictable codes are enumerable.
- **`sync.Map` for per-IP limiters** ([Module 1](01-advanced-concurrency.md))
  avoids a global mutex becoming a bottleneck as distinct IPs accumulate.

## Stretch goals

- Add a `DELETE /links/{code}` endpoint restricted to the link's creator —
  this requires adding a simple bearer-token auth middleware and an
  `owner` column, and writing a test proving another "user"'s token gets
  403'd.
- Add Prometheus metrics ([Module 7](07-observability.md)) counting
  redirects per code and exposing them at `/metrics`, then containerize
  the whole service with the multi-stage `Dockerfile` from
  [Module 6](06-deployment-docker.md) and confirm the static binary still
  builds cleanly with `CGO_ENABLED=0` despite using `modernc.org/sqlite`
  (a pure-Go driver, so this should just work — confirm it does).
- Replace the manual REST API with a parallel gRPC service
  ([Module 3](03-grpc-basics.md)) exposing the same `Create`/`Resolve`
  operations, sharing the same `Store` implementation underneath both
  transports.
- Wire the whole thing into the CI pipeline from
  [Module 5](05-testing-at-scale-ci.md): `go vet`, `go test -race
  -coverprofile`, a coverage floor, and `golangci-lint`, all running on
  every push.
