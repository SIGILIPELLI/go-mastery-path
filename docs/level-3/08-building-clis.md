# 08 · Building CLIs

Go compiles to a single static binary, which makes it a natural fit for
command-line tools — no runtime to install on the target machine. This
module covers the standard library's `flag` package, reading `stdin` line
by line, and the exit-code and error-reporting conventions Unix tools are
expected to follow.

## Flags, stdin, and exit codes

```go
package main

import (
	"bufio"
	"flag"
	"fmt"
	"os"
)

func main() {
	upper := flag.Bool("upper", false, "uppercase each line")
	prefix := flag.String("prefix", "", "prefix to add to each line")
	flag.Parse()

	if flag.NArg() > 0 && flag.Arg(0) == "help" {
		flag.Usage()
		return
	}

	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		line := scanner.Text()
		if *upper {
			line = toUpper(line)
		}
		fmt.Println(*prefix + line)
	}
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, "error reading input:", err)
		os.Exit(1)
	}
}

func toUpper(s string) string {
	b := []byte(s)
	for i, c := range b {
		if c >= 'a' && c <= 'z' {
			b[i] = c - 32
		}
	}
	return string(b)
}
```

Building and running it:

```console
$ go build -o textcli .
$ printf "hello\nworld\n" | ./textcli -upper -prefix="> "
> HELLO
> WORLD

$ ./textcli -h
Usage of ./textcli:
  -prefix string
    	prefix to add to each line
  -upper
    	uppercase each line

$ ./textcli -bogus
flag provided but not defined: -bogus
Usage of ./textcli:
  -prefix string
    	prefix to add to each line
  -upper
    	uppercase each line
$ echo "exit=$?"
exit=2
```

`flag.Parse()` returning an unrecognized flag prints usage automatically and
calls `os.Exit(2)` for you — that exit code (2 for usage errors, distinct
from 1 for runtime errors) is a real Unix convention scripts rely on to
tell "you used me wrong" apart from "something failed while running."
`bufio.Scanner` handles line splitting and buffer growth so you don't
hand-roll a `ReadString('\n')` loop that mishandles the last line missing a
trailing newline.

**The trap**: `flag.Bool`/`flag.String` return `*bool`/`*string` — you must
dereference (`*upper`, `*prefix`) after `flag.Parse()` runs, and reading
them *before* `Parse()` is called always sees the zero value regardless of
what the user actually passed.

## Reporting errors like a Unix tool

Two rules make a CLI compose well with pipes and scripts: errors go to
`stderr`, not `stdout` (so `mytool | grep x` doesn't see error noise mixed
into real output), and a non-zero exit code signals failure to the shell
and to `&&`/`||` chains:

```go
if err := scanner.Err(); err != nil {
	fmt.Fprintln(os.Stderr, "error reading input:", err)
	os.Exit(1)
}
```

Anything printed with plain `fmt.Println` goes to `stdout` and is what
downstream tools in a pipeline actually consume — mixing a log line into
that stream (instead of `stderr`) is a common bug that only shows up once
someone pipes your tool's output somewhere else.

## Subcommands without a framework

For tools with multiple verbs (`mytool add`, `mytool list`), a `flag.NewFlagSet`
per subcommand keeps each verb's flags independent, no third-party CLI
library required:

```go
func main() {
	addCmd := flag.NewFlagSet("add", flag.ExitOnError)
	addName := addCmd.String("name", "", "item name")

	listCmd := flag.NewFlagSet("list", flag.ExitOnError)
	listAll := listCmd.Bool("all", false, "include archived items")

	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "expected 'add' or 'list' subcommand")
		os.Exit(2)
	}

	switch os.Args[1] {
	case "add":
		addCmd.Parse(os.Args[2:])
		fmt.Println("adding:", *addName)
	case "list":
		listCmd.Parse(os.Args[2:])
		fmt.Println("listing, all =", *listAll)
	default:
		fmt.Fprintf(os.Stderr, "unknown subcommand %q\n", os.Args[1])
		os.Exit(2)
	}
}
```

`addCmd` and `listCmd` each own their own flag namespace, so `-all` means
nothing to `add` and `-name` means nothing to `list` — no cross-talk, and
`flag.ExitOnError` on each set gives per-subcommand usage text for free.

## Go-specific traps

- **`os.Exit` skips deferred functions.** Any `defer file.Close()` or
  `defer cleanup()` earlier in `main` will *not* run if a later code path
  calls `os.Exit` — structure `main` so exit-worthy errors are detected
  before resources that need cleanup are opened, or factor the real logic
  into a function that returns an `int` exit code and call `os.Exit` only
  once, at the very end of `main`.
- **`flag.Parse()` must run before `flag.Args()`/`flag.NArg()`** are
  meaningful — calling them first sees an empty set.
- **Global `flag.CommandLine` state** means calling `flag.String(...)` twice
  with the same name across a codebase panics at init time — a good reason
  to prefer `flag.NewFlagSet` per subcommand once a CLI grows past one verb.
- **`bufio.Scanner`'s default buffer caps at 64KB per line** — a line
  longer than that (a huge JSON blob on one line, say) returns
  `bufio.ErrTooLong` from `scanner.Err()`; call `scanner.Buffer(buf,
  maxSize)` to raise the limit if that's expected input.

## Cheat sheet

| Task | API |
|---|---|
| Boolean flag | `f := flag.Bool("name", false, "usage")` |
| String flag | `f := flag.String("name", "", "usage")` |
| Parse `os.Args` | `flag.Parse()` (call before reading any flag value) |
| Non-flag positional args | `flag.Args()`, `flag.Arg(i)`, `flag.NArg()` |
| Per-subcommand flags | `fs := flag.NewFlagSet("name", flag.ExitOnError)` |
| Read stdin line by line | `bufio.NewScanner(os.Stdin)`, `for scanner.Scan() { scanner.Text() }` |
| Report an error | `fmt.Fprintln(os.Stderr, "...", err)` |
| Signal failure to the shell | `os.Exit(1)` (or `2` for usage errors) |

## Related lessons

- Structuring larger programs into packages the way a multi-command CLI
  needs: [Level 2, Module 6](../level-2/09-dependency-management.md).
- Interfaces for swapping a real `os.Stdin` for a test `strings.Reader`:
  [Module 9](09-interfaces-generics.md).

## Exercise

Add a `-count` int flag to `textcli` that only prints the first N lines
(default: all lines, i.e. no limit when unset or `<= 0`). Then convert the
tool into two subcommands using `flag.NewFlagSet`: `textcli upper` (the
existing uppercase behavior) and `textcli count -n 3` (print only the first
`n` lines, unmodified). Verify both with piped input via `printf ... |
./textcli <subcommand> ...` and confirm `textcli bogus` exits with code 2
and an error on `stderr`.
