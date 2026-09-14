# 01 · Setup & First Program

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/ubSWGDnc3Fc" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Install Go

Go ships as a single toolchain: compiler, formatter, test runner, and package
manager all live behind one `go` command. Install a recent stable release:

```bash
# macOS (Homebrew)
brew install go

# Ubuntu/Debian
sudo apt install golang-go

# Windows / any OS: download the installer from https://go.dev/dl/
```

Verify the install:

```bash
go version
# go version go1.22.x darwin/arm64
```

`GOPATH` and workspace conventions used to matter a lot; modern Go (1.16+)
works fine anywhere on disk once you're inside a module — no special folder
required.

## Your first program

Create a directory and a file called `main.go`:

```go
// main.go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world!")
}
```

Every runnable Go program has a `package main` and a `func main()` — that's
the entry point the Go runtime looks for. `import "fmt"` pulls in the
standard library's formatted-I/O package so you can use `fmt.Println`.

## Running and building

```bash
go run main.go
# Hello, world!
```

`go run` compiles to a temporary binary, executes it, and cleans up — perfect
for quick iteration. For a real, distributable binary:

```bash
go build main.go
# produces an executable named "main" (or "main.exe" on Windows)

./main
# Hello, world!
```

`go build` compiles to a static binary with no external runtime dependency —
you can copy that file to another machine of the same OS/architecture and it
will just work.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `package main` | Declares this file belongs to the `main` package — the one that produces an executable. |
| `import "fmt"` | Brings in a standard-library package for formatted I/O. |
| `func main()` | The program's entry point — no parameters, no return value. |
| `fmt.Println(...)` | Prints its arguments followed by a newline to standard output. |
| No semicolons | Go statements end at the newline; the compiler inserts semicolons for you. |

## gofmt: one style, no debates

Go ships an opinionated formatter. Run it on save (most editors do this
automatically) and every Go codebase looks the same:

```bash
gofmt -l .        # list files that aren't formatted
gofmt -w main.go  # rewrite a file in place
go fmt ./...      # gofmt across an entire module
```

## Choosing an editor

**VS Code** with the official "Go" extension (free, includes `gopls` — the Go
language server — for autocomplete, jump-to-definition, and inline errors) is
the most common choice. **GoLand** (JetBrains) is a strong paid alternative
with deeper refactoring tools. Either works well; pick one and move on.

## How It Actually Works

`go build` doesn't shell out to a linker toolchain the way C does by default — the Go
toolchain ships its own assembler, compiler, and linker (`cmd/compile`, `cmd/link`),
so a fresh install can compile and statically link a binary with zero external
dependencies. `go run` is really `go build` into a temp directory under
`$GOCACHE`/`os.TempDir()`, followed by exec of that binary — nothing magic, just a
convenience wrapper. `GOPATH` used to be mandatory (every import path was a directory
under `$GOPATH/src`); Go modules (`go.mod`) replaced that by recording a module path
and dependency versions directly in the repo, and the module cache under
`$GOPATH/pkg/mod` is now just a shared, content-addressed download cache, not the
place your code has to live. `gofmt` isn't a linter opinion — it's an AST
pretty-printer: it parses your file into a syntax tree with `go/parser` and
re-emits it in one canonical layout, which is why two different Go files formatted
by two different people always diff cleanly against each other.


## Exercise

Write a program `greeter.go` with a `main` function that prints a greeting
for three different names, one per line, using `fmt.Println`. Run it with
`go run greeter.go`, then compile it to a binary with `go build` and run the
resulting executable directly.
