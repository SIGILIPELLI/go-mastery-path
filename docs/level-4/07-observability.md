# 07 · Observability

A service that's hard to debug in production is a service that costs
sleep. This module wires structured logging (`log/slog`, standard library
since Go 1.21) and Prometheus metrics into the middleware chain built up
over [Module 4](04-production-grade-apis.md), and shows the actual JSON
logs and metrics output a real request produces.

## Structured logging with `log/slog`

```go
var logger = slog.New(slog.NewJSONHandler(os.Stdout, nil))

logger.Info("request handled",
	slog.String("method", r.Method),
	slog.String("path", r.URL.Path),
	slog.Int("status", rec.status),
	slog.Duration("duration", dur),
)
```

`slog.NewJSONHandler` emits one JSON object per line — a log aggregator
(anything ingesting from stdout in a container environment) can index
`path` or `status` as real fields instead of regex-parsing a free-text log
line. Compare this to `log.Printf("%s %s %d %v", ...)`: readable to a
human tailing a terminal, but "find every request over 500ms" requires a
parser for the printed one and a simple query for the structured one.

## Prometheus metrics

```go
var (
	requestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "http_requests_total",
		Help: "Total HTTP requests",
	}, []string{"path", "status"})

	requestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "http_request_duration_seconds",
		Help:    "Request duration in seconds",
		Buckets: prometheus.DefBuckets,
	}, []string{"path"})
)
```

`promauto.NewCounterVec`/`NewHistogramVec` register the metric with
Prometheus's default registry the moment they're created — no separate
registration call needed. The `[]string{"path", "status"}` label set means
a single counter definition tracks per-route, per-status-code counts
simultaneously; querying `http_requests_total{path="/fail"}` in Prometheus
later slices exactly that dimension out.

## Wiring both into one middleware

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func withObservability(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: 200}
		next.ServeHTTP(rec, r)
		dur := time.Since(start)

		requestsTotal.WithLabelValues(r.URL.Path, http.StatusText(rec.status)).Inc()
		requestDuration.WithLabelValues(r.URL.Path).Observe(dur.Seconds())

		logger.Info("request handled",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", rec.status),
			slog.Duration("duration", dur),
		)
	})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("hi"))
	})
	mux.HandleFunc("GET /fail", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "boom", http.StatusInternalServerError)
	})
	mux.Handle("GET /metrics", promhttp.Handler())

	http.ListenAndServe(":8095", withObservability(mux))
}
```

`statusRecorder` exists because a plain `http.ResponseWriter` never
exposes the status code your own handler wrote — wrapping it and
intercepting `WriteHeader` is the standard trick for capturing it in
middleware that runs *after* the handler.

Running it and hitting all three endpoints:

```console
$ curl -s localhost:8095/hello
hi
$ curl -s -o /dev/null localhost:8095/fail
$ curl -s localhost:8095/hello
hi
```

The structured JSON logs produced, one line per request:

```json
{"time":"2026-08-21T22:37:42.691167+05:30","level":"INFO","msg":"starting server","addr":":8095"}
{"time":"2026-08-21T22:37:42.996255+05:30","level":"INFO","msg":"request handled","method":"GET","path":"/hello","status":200,"duration":4583}
{"time":"2026-08-21T22:37:43.012343+05:30","level":"INFO","msg":"request handled","method":"GET","path":"/fail","status":500,"duration":14042}
{"time":"2026-08-21T22:37:43.024788+05:30","level":"INFO","msg":"request handled","method":"GET","path":"/hello","status":200,"duration":3750}
```

And the resulting Prometheus metrics, scraped from `/metrics`:

```
http_request_duration_seconds_count{path="/fail"} 1
http_request_duration_seconds_count{path="/hello"} 2
http_requests_total{path="/fail",status="Internal Server Error"} 1
http_requests_total{path="/hello",status="OK"} 2
```

Notice `duration` in the JSON logs is a bare integer (`4583`,
`14042`) — that's nanoseconds, `slog.Duration`'s default `time.Duration`
encoding, not milliseconds. A log consumer expecting milliseconds will
silently misinterpret this by a factor of a million unless the field is
documented or converted explicitly before logging (e.g.
`slog.Float64("duration_ms", float64(dur.Milliseconds()))`).

## Go-specific traps

- **`slog.Duration`'s JSON output is nanoseconds as a plain number**, not a
  human string like `"4.5ms"` — as shown above, this catches people
  expecting `time.Duration`'s `String()` format (`4.583ms`), which only
  appears with `slog.NewTextHandler`, not the JSON handler.
- **Forgetting the `statusRecorder` wrapper** means middleware logging
  "status: 200" for every request regardless of what the handler actually
  sent — `http.ResponseWriter` has no built-in way to read back a status
  code once written.
- **Registering the same metric name twice** (e.g. a package-level
  `promauto.NewCounterVec` called inside a function instead of at
  package scope) panics at runtime with "duplicate metrics collector
  registration attempted" — metrics must be created exactly once, typically
  as package-level `var`s.
- **High-cardinality labels** (putting a user ID or a raw request path
  with path parameters, e.g. `/users/12345`, directly into a Prometheus
  label) can blow up the metrics storage — normalize dynamic path segments
  to their route pattern (`/users/{id}`) before using them as a label
  value.
- **`http.StatusText(rec.status)` returns `""` for a status code it
  doesn't recognize**, which then appears as an empty label value in
  Prometheus — using the raw integer status code as the label avoids the
  ambiguity, at the cost of a slightly less readable metric line.

## Cheat sheet

| Need | API |
|---|---|
| Structured JSON logs | `slog.New(slog.NewJSONHandler(os.Stdout, nil))` |
| Log a field | `slog.String(k, v)`, `slog.Int(k, v)`, `slog.Duration(k, d)` |
| Counter metric | `promauto.NewCounterVec(prometheus.CounterOpts{...}, labels)` |
| Histogram (latency) metric | `promauto.NewHistogramVec(prometheus.HistogramOpts{...}, labels)` |
| Increment/observe | `.WithLabelValues(...).Inc()` / `.Observe(seconds)` |
| Expose a scrape endpoint | `mux.Handle("GET /metrics", promhttp.Handler())` |
| Capture the response status in middleware | Wrap `http.ResponseWriter`, override `WriteHeader` |

## Related lessons

- The middleware chain and `http.Server` this module extends:
  [Module 4](04-production-grade-apis.md).
- Circuit breakers and client timeouts whose failures these metrics would
  surface: [Module 2](02-microservices-patterns.md).
- Deploying this instrumented service in a container:
  [Module 6](06-deployment-docker.md).

## Exercise

Add a `route` label derived from the matched pattern rather than the raw
`r.URL.Path` (hint: `net/http`'s Go 1.22+ mux exposes the matched pattern
via `r.Pattern` inside the handler once it's been dispatched — capture it
in `statusRecorder` or read it after `next.ServeHTTP` returns) so that
`GET /tasks/{id}` for ID `1` and ID `2` both aggregate under one metric
series instead of two. Then fix the nanosecond-duration logging trap
called out above by adding a `slog.Float64("duration_ms", ...)` field
alongside (or instead of) the raw `slog.Duration`, and confirm in the
output that it reads as a normal millisecond float.
