# 09 · Dependency Management

## Modules are Go's package manager

Since Go 1.11 (and mandatory since 1.16), every Go project is a **module**: a
directory tree with a `go.mod` file at its root declaring the module's own
import path, the Go version it targets, and every external package it depends
on. There is no separate package manager to install — `go` is the tool.

[Level 1, Module 9](../level-1/09-packages-modules.md) introduced `go mod init`
and internal packages. This module covers the other half: consuming third-party
code safely, understanding versions, and keeping builds reproducible.

## Anatomy of `go.mod`

```text
module github.com/yourname/weathercli

go 1.22

require (
    github.com/spf13/cobra v1.8.0
    github.com/stretchr/testify v1.9.0
)

require (
    github.com/inconshreveable/mousetrap v1.1.0 // indirect
    github.com/spf13/pflag v1.0.5 // indirect
    gopkg.in/yaml.v3 v3.0.1 // indirect
)
```

| Directive | Meaning |
|-----------|---------|
| `module` | this module's import path — how others import your packages |
| `go` | the language version the module targets, gating language features |
| `require` | a dependency and its minimum acceptable version |
| `// indirect` | pulled in by a dependency, not imported by your code directly |
| `replace` | substitute a different source or version for a module |
| `exclude` | forbid a specific version from being selected |
| `toolchain` | the minimum Go toolchain version to build with |

The `module` line matters more than it looks: if you publish to
`github.com/yourname/weathercli`, that path must match exactly, because it is
both the import path *and* the address the tooling fetches from.

## Adding dependencies

Two workflows, both fine:

```bash
# 1. Explicit: fetch and record a dependency
go get github.com/spf13/cobra@latest
go get github.com/spf13/cobra@v1.8.0     # a specific version
go get github.com/spf13/cobra@v1          # latest v1.x.y

# 2. Implicit: write the import, then let the toolchain resolve it
go mod tidy
```

`go mod tidy` is the workhorse. It scans your source (including tests), adds
anything imported but missing, removes anything recorded but unused, and
updates `go.sum`. Run it before every commit — a `go.mod` with stale entries is
a common source of confusing CI failures.

Note that since Go 1.16, `go build` and `go run` will **not** silently modify
`go.mod`; they error and tell you to run `go mod tidy`. That is deliberate:
builds should be reproducible, not self-mutating.

## Semantic versioning and how Go picks versions

Go modules require semantic versioning: `vMAJOR.MINOR.PATCH`.

| Change | Bump | Meaning |
|--------|------|---------|
| bug fix, no API change | `v1.2.3` → `v1.2.4` | patch: always safe |
| new backwards-compatible feature | `v1.2.3` → `v1.3.0` | minor: safe to upgrade |
| breaking change | `v1.2.3` → `v2.0.0` | major: requires code changes |

Go uses **Minimal Version Selection (MVS)**, and it is deliberately different
from npm or pip. Go builds with the *highest version explicitly required by
anyone in the graph* — and nothing newer. If your module needs `v1.2.0` and a
dependency needs `v1.4.0`, you get `v1.4.0`. Nobody's build silently drifts to
a version released this morning; upgrades happen only when someone edits
`go.mod`. This is why Go builds are so reproducible.

### The major-version rule

For `v2` and above, the major version becomes part of the import path:

```go
import "github.com/user/lib"      // v0 or v1
import "github.com/user/lib/v2"   // v2.x.y
import "github.com/user/lib/v3"   // v3.x.y
```

This looks strange at first, but it means `v1` and `v2` of the same library
can coexist in one build — a genuine problem in other ecosystems ("diamond
dependency hell") that Go sidesteps entirely.

## `go.sum` and integrity

`go.sum` records a cryptographic hash for every module version your build has
ever used:

```text
github.com/spf13/cobra v1.8.0 h1:7aJaZx1B85qltLMc546zn58BxxfZdR/W22ej9CFoEf0=
github.com/spf13/cobra v1.8.0/go.mod h1:WXLWApfZ71AjXPya3WOlMsY9yMs7YeiHhFVlvLyhcho=
```

Every download is verified against these hashes. If a published version is
ever altered — a tag force-pushed, a proxy compromised — the build fails
loudly instead of silently compiling different code.

**Commit both `go.mod` and `go.sum`.** They are not lockfile clutter; they are
your build's integrity guarantee. Never edit `go.sum` by hand; regenerate it
with `go mod tidy`.

## The module proxy and checksum database

By default the toolchain fetches through Google's public infrastructure:

- **`proxy.golang.org`** — an immutable cache of module versions. Once
  published, a version is stored forever, so a deleted or force-pushed GitHub
  tag cannot break your build.
- **`sum.golang.org`** — a transparency log of module hashes, cross-checking
  what the proxy serves.

Configure via environment variables:

```bash
go env GOPROXY          # https://proxy.golang.org,direct
go env GOSUMDB          # sum.golang.org
go env GOPRIVATE        # (empty by default)

# Private repos: skip the proxy and checksum DB for your company's modules
go env -w GOPRIVATE=github.com/mycompany/*

# Corporate proxy, or fully offline
go env -w GOPROXY=https://proxy.internal.example.com,direct
go env -w GOFLAGS=-mod=mod
```

`GOPRIVATE` is the one you will actually need: without it, the toolchain tries
to fetch your private repo through the public proxy, fails, and gives a
confusing 404.

## Inspecting and upgrading

```bash
go list -m all                  # every module in the build, with versions
go list -m -u all               # ... and mark which have updates available
go list -m -versions github.com/spf13/cobra   # all published versions

go get -u ./...                 # upgrade all deps to latest minor/patch
go get -u=patch ./...           # patch releases only -- the conservative choice
go get github.com/x/y@v1.5.0    # pin one dependency exactly
go get github.com/x/y@none      # remove a dependency

go mod why github.com/spf13/pflag   # explain why this module is in the build
go mod graph                        # full dependency graph, one edge per line
go mod verify                       # confirm downloads match go.sum
```

`go mod why` is invaluable when an unfamiliar `// indirect` entry appears and
you want to know who dragged it in.

Upgrade deliberately: `go get -u ./...` followed immediately by
`go test ./...`. Minor versions are *supposed* to be compatible, but "supposed
to" is not a guarantee.

## `replace` for local development

When you need to test an unreleased fix in a dependency, point the module at a
local checkout:

```text
// go.mod
require github.com/user/lib v1.2.0

replace github.com/user/lib => ../lib
```

Or point at a fork:

```text
replace github.com/user/lib => github.com/yourfork/lib v1.2.1-fix
```

`replace` directives are honoured only in the **main** module — they are
ignored when your module is consumed as a dependency, so a stray local
`replace` breaks nobody but you. Still, remove local-path replaces before
publishing, or others cannot build your module at all.

## Workspaces for multi-module development

Go 1.18 added `go.work` for editing several modules at once without polluting
any `go.mod` with `replace` lines:

```bash
mkdir myproject && cd myproject
go work init ./api ./worker ./shared
go work use ./newservice   # add another later
```

```text
// go.work -- add this to .gitignore; it is a local development file
go 1.22

use (
    ./api
    ./worker
    ./shared
)
```

Now `./api` builds against your local `./shared` automatically. Delete
`go.work` (or set `GOWORK=off`) to build the way CI does.

## Vendoring

`go mod vendor` copies every dependency's source into a `vendor/` directory in
your repo:

```bash
go mod vendor
go build -mod=vendor ./...   # implied automatically when vendor/ exists
```

This makes builds fully offline and immune to upstream deletion, at the cost
of a much larger repository and noisy diffs. With the module proxy providing
immutability, most projects no longer need it — reach for it only under an
air-gapped or strict-audit requirement.

## Reproducible-build checklist

- Commit `go.mod` **and** `go.sum`.
- Run `go mod tidy` before committing; make CI fail if it produces a diff.
- Use `go build -mod=readonly` (the default) in CI so a build never silently
  edits `go.mod`.
- Do not commit `go.work`.
- Set `GOPRIVATE` for internal modules.
- Audit dependencies with `go list -m all` and check known issues with
  `govulncheck ./...`.

## Cheat sheet

| Task | Command |
|------|---------|
| Start a module | `go mod init github.com/you/proj` |
| Add / update a dependency | `go get github.com/x/y@v1.2.3` |
| Sync `go.mod` with imports | `go mod tidy` |
| Upgrade everything | `go get -u ./...` |
| Patch-only upgrades | `go get -u=patch ./...` |
| Remove a dependency | `go get github.com/x/y@none` |
| List the build's modules | `go list -m all` |
| Show available updates | `go list -m -u all` |
| Explain a dependency | `go mod why github.com/x/y` |
| Verify checksums | `go mod verify` |
| Vendor dependencies | `go mod vendor` |
| Local override | `replace github.com/x/y => ../y` |
| Multi-module workspace | `go work init ./a ./b` |
| Download without building | `go mod download` |
| Scan for vulnerabilities | `govulncheck ./...` |

## Related lessons

- Package structure, `internal/`, and exported identifiers:
  [Level 1, Module 9](../level-1/09-packages-modules.md).
- The project in [Module 10](10-project-weather-cli.md) keeps zero external
  dependencies on purpose — the standard library covers it.
- Supply-chain hardening in production:
  [Level 4, Module 8](../level-4/08-security-best-practices.md).

## Exercise

Create a module `example.com/depdemo` and add `github.com/google/uuid` with
`go get`. Write a `main.go` that prints three generated UUIDs. Then: run
`go list -m all` and note the indirect dependencies; run `go mod why
github.com/google/uuid`; delete the import from `main.go`, run `go mod tidy`,
and confirm the `require` line disappears from `go.mod`. Finally, open
`go.sum`, count how many lines exist per module version, and explain what the
`/go.mod` suffixed hashes are for.
