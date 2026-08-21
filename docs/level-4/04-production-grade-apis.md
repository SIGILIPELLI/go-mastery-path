# 04 · Production-Grade APIs

[Level 3, Module 2](../level-3/02-building-rest-apis.md) built a working
REST API. This module hardens it for production: server timeouts that
protect against slow clients, panic recovery so one bad request doesn't
kill the process, structured error responses, and graceful shutdown that
finishes in-flight requests instead of dropping them on deploy.

## Timeouts on the server, not just the client

`http.ListenAndServe` with a bare `http.Server{}` inherits no timeouts at
all — a slow or malicious client can hold a connection open indefinitely.
Configure the `http.Server` explicitly instead:

```go
srv := &http.Server{
	Addr:              ":8090",
	Handler:           recoverMiddleware(mux),
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

`ReadHeaderTimeout` alone closes off the "Slowloris" class of attack
(a client trickling headers one byte at a time to hold a connection open) —
it's the one timeout that has essentially no downside and should be set on
every production `http.Server`.

## Panic recovery middleware

```go
func recoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic recovered: %v", rec)
				writeError(w, &apiError{Status: 500, Code: "internal_error", Msg: "something went wrong"})
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

```go
type apiError struct {
	Status int    `json:"-"`
	Code   string `json:"code"`
	Msg    string `json:"message"`
}

func writeError(w http.ResponseWriter, err *apiError) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(err.Status)
	json.NewEncoder(w).Encode(err)
}

func flakyHandler(w http.ResponseWriter, r *http.Request) {
	panic("simulated bug")
}
```

```console
$ curl -s localhost:8090/healthz
{"status":"ok"}

$ curl -s -o /dev/null -w "%{http_code}\n" localhost:8090/boom
500

$ curl -s localhost:8090/boom
{"code":"internal_error","message":"something went wrong"}
```

Server log for that same request:

```
2026/08/21 22:33:44 panic recovered: simulated bug
```

Without `recoverMiddleware`, `net/http` already recovers panics per-request
at a lower level (it won't crash the whole process), but it just closes the
connection with no response body and logs a full stack trace — a client
sees a bare connection reset, not a usable JSON error. Wrapping every
handler in your *own* recover gives you a controlled, structured response
and a place to log with your own format/fields (request ID, route, etc.)
instead of the default trace dump.

**The trap**: `defer` + `recover()` only catches a panic on the *same*
goroutine. A panic inside a goroutine the handler spawns (`go
doSomething()`) is invisible to `recoverMiddleware` and crashes the whole
process — any goroutine launched from inside a handler needs its own
recover.

## Graceful shutdown

```go
go func() {
	fmt.Println("listening on", srv.Addr)
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
}()

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
<-stop

fmt.Println("shutting down...")
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
	log.Fatal("graceful shutdown failed:", err)
}
fmt.Println("shutdown complete")
```

```console
$ ./server &
listening on :8090

$ curl -s localhost:8090/healthz > /dev/null

$ kill -TERM <pid>
```

Server output after the `SIGTERM`:

```
shutting down...
shutdown complete
```

`srv.ListenAndServe()` runs in its own goroutine specifically so `main` is
free to block on `<-stop` — the OS signal — instead of blocking on the
server itself, which never returns during normal operation.
`srv.Shutdown(ctx)` stops accepting new connections immediately but lets
in-flight requests finish (up to the 5-second context deadline), which is
what separates a graceful deploy from one that cuts active users off
mid-request. `errors.Is(err, http.ErrServerClosed)` distinguishes "we shut
it down on purpose" from a genuine startup/runtime failure that deserves
`log.Fatal`.

## Go-specific traps

- **`http.ListenAndServe`'s return value being non-nil is *expected* on a
  clean shutdown** — `srv.Shutdown` makes `ListenAndServe` return
  `http.ErrServerClosed`; treating that as a fatal error crashes the
  process on every intentional shutdown.
- **A handler spawning a goroutine that outlives the request** needs its
  own context derived from something other than `r.Context()` — the
  request's context is cancelled the instant the handler returns, so
  `go func() { doSlow(r.Context()) }()` silently gets cancelled almost
  immediately.
- **Setting `WriteTimeout` too low for legitimately slow endpoints**
  (large file downloads, long-polling) cuts them off mid-response — apply
  a longer or per-route timeout for those specific handlers rather than
  raising the global default for everything.
- **`signal.Notify` with an unbuffered channel** can miss a signal sent
  before the receiving goroutine is scheduled — always give the channel
  capacity 1, as shown, per the `os/signal` package's own documented
  requirement.
- **Forgetting `defer cancel()` on the shutdown context** leaks the timer
  the same way any other `context.WithTimeout` does
  ([Level 3, Module 4](../level-3/04-context-package.md)) — small in a
  process about to exit, but still worth the habit.

## Cheat sheet

| Concern | Setting / pattern |
|---|---|
| Protect against slow header writers | `http.Server{ReadHeaderTimeout: d}` |
| Bound full request/response time | `ReadTimeout`, `WriteTimeout` |
| Idle keep-alive connections | `IdleTimeout` |
| Structured JSON error response | Custom `apiError` type + a `writeError` helper |
| Survive a handler panic | `recoverMiddleware` wrapping the whole mux |
| Stop accepting new work, finish in-flight | `srv.Shutdown(ctx)` on `SIGINT`/`SIGTERM` |
| Distinguish clean shutdown from real failure | `errors.Is(err, http.ErrServerClosed)` |

## Related lessons

- Basic handler and middleware shape this module hardens:
  [Level 3, Module 2](../level-3/02-building-rest-apis.md).
- `context` deadlines used identically for the shutdown timeout:
  [Level 3, Module 4](../level-3/04-context-package.md).
- Observability (structured logging, metrics) to pair with this hardening:
  [Module 7](07-observability.md).

## Exercise

Add a request ID middleware that generates a random ID per request (a
simple counter or `crypto/rand`-backed string is fine), stores it in the
request context via `context.WithValue`
([Level 3, Module 4](../level-3/04-context-package.md)), and includes it in
both the structured error JSON and the recovered-panic log line. Then add a
second, deliberately slow handler (`time.Sleep(3 * time.Second)`) and
confirm that sending `SIGTERM` while a request to it is in flight lets that
request complete before "shutdown complete" prints — but a *second* request
sent immediately after the signal gets connection-refused, since
`Shutdown` stops accepting new connections right away.
