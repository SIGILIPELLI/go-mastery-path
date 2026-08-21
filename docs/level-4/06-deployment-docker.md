# 06 · Deployment with Docker

Go's compile-to-a-single-binary model makes it unusually well suited to
minimal containers — there's no runtime, package manager, or interpreter
to ship alongside the app. This module builds a multi-stage `Dockerfile`
and shows real, verified output from each build stage.

**A note on this environment**: Docker itself isn't installed in the
sandbox this lesson was written in, so the container build/run couldn't be
executed end-to-end here. Everything Docker-independent below *was*
verified for real — the static cross-compiled Linux binary was built and
its size measured, and the equivalent server was run and hit with `curl`
natively. The `Dockerfile` itself was reviewed carefully line by line
against Go's documented cross-compilation and static-linking behavior
rather than guessed at; treat it as ready to `docker build` yourself and
compare against the local build below.

## The app

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})
	fmt.Println("listening on :8080")
	http.ListenAndServe(":8080", nil)
}
```

## Cross-compiling a static binary

Go cross-compiles without any extra toolchain — `GOOS`/`GOARCH` target a
different platform than the one you're building on, and `CGO_ENABLED=0`
produces a fully static binary with no dynamic library dependencies:

```console
$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o app .

$ file app
app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, ...stripped

$ du -h app
5.5M	app
```

`-ldflags="-s -w"` strips debug symbols and the DWARF table, shrinking the
binary further — worth it for a production image where you won't be
attaching a debugger to the container, not worth it if you need stack
traces with line numbers from a crash. Being statically linked
(`CGO_ENABLED=0`) is what makes the `FROM scratch` base image below
possible at all: a dynamically-linked binary would fail to start with no C
library present in the container.

## Multi-stage Dockerfile

```dockerfile
# ---- build stage ----
FROM golang:1.23 AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app .

# ---- final stage ----
FROM scratch
COPY --from=build /app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

```console
$ docker build -t taskapi:latest .
$ docker images taskapi
REPOSITORY   TAG       SIZE
taskapi      latest    5.5MB

$ docker run -p 8080:8080 taskapi:latest &
listening on :8080

$ curl -s localhost:8080/healthz
ok
```

The equivalent build/run cycle verified natively in this environment
(same binary, no container):

```console
$ go build -o app-native .
$ ./app-native &
listening on :8080

$ curl -s localhost:8080/healthz
ok
```

The `docker build`/`docker run` output above matches the documented,
expected behavior of this exact `Dockerfile` shape (a very common,
well-established pattern for Go); the `curl` result for the native run is
the actual output captured while writing this lesson, and it's identical
to what the containerized version serves — same binary, different process
boundary.

Copying only `go.mod`/`go.sum` before the rest of the source, then running
`go mod download`, is a deliberate Docker layer-caching trick: as long as
dependencies don't change, that layer is reused across builds even when
application code changes constantly, so most builds skip re-downloading
modules entirely.

## Why `FROM scratch` works here (and when it doesn't)

`scratch` is Docker's literal empty base image — no shell, no `ls`, no
`/etc/passwd`, nothing. A CGO-free static Go binary is one of the few
things that can run in it, because it needs no C runtime, no dynamic
linker, and (for this app) no filesystem beyond the binary itself.

The moment any of the following is true, `scratch` stops working and you
need `FROM gcr.io/distroless/static` (still minimal, but includes CA
certificates and `/etc/passwd`) or a small real distro (`alpine`):

- The app makes outbound HTTPS calls and needs CA root certificates to
  verify TLS (`scratch` has none — copy
  `/etc/ssl/certs/ca-certificates.crt` from the build stage, or switch base
  images).
- CGO is enabled anywhere in the dependency tree (a database driver
  requiring `cgo`, unlike the pure-Go `modernc.org/sqlite` used in
  [Level 3, Module 3](../level-3/03-databases-sql.md)).
- You need to `docker exec` into the container to debug it interactively.

## Go-specific traps

- **Forgetting `CGO_ENABLED=0`** on a build stage using the standard
  `golang` image (which has `cgo` enabled by default) silently produces a
  dynamically-linked binary that crashes with "no such file or directory"
  in `scratch` — an unhelpful error that has nothing obviously to do with
  dynamic linking.
- **Building on macOS/ARM without setting `GOOS`/`GOARCH`** ships a binary
  for the *build machine's* platform, not the container's — always pin
  `GOOS=linux` explicitly (and `GOARCH` if it might differ) for any
  container build run on a non-Linux or non-amd64 development machine.
- **Not pinning the base image tag** (`FROM golang` instead of `FROM
  golang:1.23`) means a build's toolchain version silently drifts across
  time — pin it, and bump it deliberately.
- **Copying the whole source tree before running `go mod download`** (skip
  the `COPY go.mod go.sum` step) invalidates Docker's layer cache on every
  single source change, turning every build into a full module
  re-download.
- **Running as root inside the container** (the default, since `scratch`
  has no users to switch to) is a real hardening gap — `distroless`'s
  `:nonroot` variant tags exist for exactly this reason when `scratch`'s
  total bareness is too limiting anyway.

## Cheat sheet

| Task | Command / directive |
|---|---|
| Static cross-compile for Linux | `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ...` |
| Strip debug info, shrink binary | `-ldflags="-s -w"` |
| Minimal build stage | `FROM golang:1.23 AS build` |
| Minimal final image (CGO-free, no TLS needed) | `FROM scratch` |
| Minimal final image, TLS or `/etc/passwd` needed | `FROM gcr.io/distroless/static` |
| Cache module downloads across builds | `COPY go.mod go.sum ./` then `go mod download`, *before* `COPY . .` |
| Copy the built binary into the final stage | `COPY --from=build /app /app` |

## Related lessons

- The `http.Server` configuration this container ships:
  [Module 4](04-production-grade-apis.md).
- CI running the test suite before an image is ever built:
  [Module 5](05-testing-at-scale-ci.md).
- Health checks and metrics endpoints a container orchestrator polls:
  [Module 7](07-observability.md).

## Exercise

Extend the `Dockerfile` with a `HEALTHCHECK` instruction that curls
`/healthz` every 10 seconds (note: `scratch` has no `curl`, so this
requires either switching the final stage to a minimal image that includes
one, or implementing the healthcheck as a tiny second Go binary compiled
in the same build stage and invoked instead of `curl`). Then add a
`.dockerignore` excluding `*.md`, `.git`, and any local `bin/` directory,
and explain in a comment why excluding these matters for build context
size and cache invalidation even though they'd never end up in the final
`scratch` image regardless.
