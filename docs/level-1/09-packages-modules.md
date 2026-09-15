---
description: "Packages & Modules — Every Go file belongs to a package, declared at the top with package name. Files in the same directory must share the same package…"
---

# 09 · Packages & Modules

## Packages: Go's unit of code organization

Every Go file belongs to a package, declared at the top with `package
name`. Files in the same directory must share the same package name. Code
in one package accesses another package's exported names via
`import`.

```text
todoapp/
    go.mod
    main.go          // package main
    task/
        task.go       // package task
        storage.go    // package task
```

```go
// task/task.go
package task

// Exported (capitalized) -- visible to other packages
type Task struct {
	Title string
	Done  bool
}

// unexported (lowercase) -- visible only inside package task
func normalize(title string) string {
	return title
}
```

```go
// main.go
package main

import (
	"fmt"

	"todoapp/task" // import path derived from the module name + folder
)

func main() {
	t := task.Task{Title: "Buy milk"}
	fmt.Println(t)
}
```

## Exported vs unexported identifiers

Go has no `public`/`private` keywords. **Capitalization is the visibility
rule**: any function, type, variable, or struct field starting with an
uppercase letter is exported (importable elsewhere); lowercase means
package-private.

```go
package task

var MaxTasks = 100      // exported -- other packages see task.MaxTasks
var defaultPriority = 1 // unexported -- only visible inside this package

func NewTask(title string) Task { // exported constructor function
	return Task{Title: title}
}
```

## go.mod: the module file

A **module** is a collection of packages versioned and distributed
together, described by a `go.mod` file at its root:

```bash
go mod init todoapp
# creates go.mod with: module todoapp
#                       go 1.22
```

```text
// go.mod
module todoapp

go 1.22

require (
	github.com/google/uuid v1.6.0
)
```

- `module todoapp` — the module's import path root; internal packages are
  imported as `todoapp/task`, `todoapp/storage`, etc.
- `go 1.22` — minimum Go language version this module requires.
- `require (...)` — external dependencies and their versions, added
  automatically when you `go get` a package or run `go mod tidy`.

## Adding and tidying dependencies

```bash
go get github.com/google/uuid@v1.6.0   # add a specific version
go get github.com/google/uuid          # add the latest version

go mod tidy   # adds missing requires, removes unused ones,
              # and updates go.sum (checksums for reproducible builds)
```

`go.sum` records cryptographic checksums of every dependency version used,
so builds are verifiable and reproducible — always commit both `go.mod` and
`go.sum` to version control.

## The standard library is huge — use it first

Before reaching for a third-party package, check whether the standard
library already covers it: `fmt` (formatting), `strings`/`strconv`
(text/parsing), `os` (files, environment, args), `net/http` (HTTP client
and server), `encoding/json` (JSON), `time` (dates/durations), `sort`,
`errors`. Idiomatic Go code leans on the standard library heavily and adds
external dependencies sparingly.

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println("args:", os.Args[1:]) // command-line arguments (excluding program name)
	home, _ := os.UserHomeDir()
	fmt.Println("home:", home)
}
```

## How It Actually Works

`go.mod` and `go.sum` aren't just metadata — they drive Minimal Version Selection
(MVS), Go's dependency resolution algorithm: for every module in your build's
dependency graph, Go picks the *minimum* version that satisfies every requirement
anywhere in the graph (not the newest available), which makes builds reproducible
without a separate lockfile-vs-manifest distinction like npm's package.json/
package-lock.json split. `go.sum` records cryptographic hashes (SHA-256, via the
module checksum database `sum.golang.org` by default) of every module version's
source tree and go.mod, and `go build`/`go get` verify a fresh download against that
hash before compiling it — a mismatch fails the build outright, which is what
protects you from a compromised or tampered module registry. The compiler processes
packages in dependency order, compiling each into an object file cached in
`$GOCACHE` keyed by a hash of the source plus build flags — this is why an unchanged
package rebuilds instantly (cache hit) while touching a leaf package invalidates
every package that (transitively) imports it.


## Cheat sheet

| Task | Command / syntax |
|------|-------------------|
| Create a module | `go mod init <module-name>` |
| Add a dependency | `go get <import-path>@<version>` |
| Clean up dependencies | `go mod tidy` |
| Import a package | `import "todoapp/task"` |
| Import stdlib package | `import "fmt"` |
| Import with alias | `import f "fmt"` |
| Exported identifier | Starts with an uppercase letter |
| Unexported identifier | Starts with a lowercase letter |

## 🔀 See this in another language

- [Rust — Modules & Cargo Project Structure](https://sigilipelli.github.io/rust-mastery-path/level-1/09-modules-cargo/)
- [Dart — Packages (pub, pubspec.yaml)](https://sigilipelli.github.io/dart-mastery-path/level-1/09-packages/)
- [Python — Modules, Packages & pip](https://sigilipelli.github.io/python-mastery-path/level-1/09-modules-packages/)

## Exercise

Create a new module `mathutils` with `go mod init mathutils`. Add a
subpackage `calc` (folder `calc/`) with an exported function `Square(n int)
int` and an unexported helper it calls internally. Import `mathutils/calc`
from a `main.go` at the module root and print `calc.Square(7)`.
