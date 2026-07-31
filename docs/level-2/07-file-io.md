# 07 · File I/O

## Reading and writing files in Go

Almost every real program touches the filesystem: config files, logs, CSV
exports, caches. Go's file APIs live mainly in the `os` package for opening
and creating files, `bufio` for efficient line-by-line access, and `io` for
the `Reader`/`Writer` interfaces that tie everything together.

The key decision is always the same: **can this file fit comfortably in
memory?** If yes, read it whole in one call. If not — or if you do not know —
stream it.

## Whole-file reads and writes

For small files, `os.ReadFile` and `os.WriteFile` are a single line each.

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	content := []byte("first line\nsecond line\nthird line\n")

	// 0644 = owner read/write, everyone else read-only.
	if err := os.WriteFile("notes.txt", content, 0644); err != nil {
		fmt.Println("write error:", err)
		return
	}

	data, err := os.ReadFile("notes.txt")
	if err != nil {
		fmt.Println("read error:", err)
		return
	}

	fmt.Println(string(data))
	fmt.Println("bytes read:", len(data)) // bytes read: 34
}
```

`os.WriteFile` truncates an existing file. Both functions load everything into
memory at once, so a 2 GB log file becomes a 2 GB allocation — that is when
you switch to the streaming approach below.

## Line-by-line with `bufio.Scanner`

`bufio.Scanner` is the standard way to walk a file one line at a time using
constant memory:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	file, err := os.Open("notes.txt") // read-only
	if err != nil {
		fmt.Println("open error:", err)
		return
	}
	defer file.Close() // ALWAYS defer the close, right after the error check

	scanner := bufio.NewScanner(file)
	lineNum := 0
	for scanner.Scan() {
		lineNum++
		fmt.Printf("%d: %s\n", lineNum, scanner.Text()) // Text() excludes "\n"
	}

	// Scan() returning false means EOF *or* an error -- always check.
	if err := scanner.Err(); err != nil {
		fmt.Println("scan error:", err)
	}
	// 1: first line
	// 2: second line
	// 3: third line
}
```

Two traps here:

1. **Forgetting `scanner.Err()`.** The loop ends identically on EOF and on a
   read error. Skipping the check silently truncates data.
2. **The 64 KB line limit.** A line longer than `bufio.MaxScanTokenSize` makes
   `Scan` return false with `bufio.ErrTooLong`. Raise it explicitly:

```go
scanner := bufio.NewScanner(file)
buf := make([]byte, 0, 64*1024)
scanner.Buffer(buf, 1024*1024) // allow lines up to 1 MB
```

You can also change what counts as a token — `scanner.Split(bufio.ScanWords)`
iterates words, `bufio.ScanRunes` iterates characters.

## Creating, appending and buffered writes

`os.Create` truncates or creates. `os.OpenFile` gives you full control via
flags — appending to a log file is the common case:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	// Append mode: create if missing, write-only, add to the end.
	f, err := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Println("open error:", err)
		return
	}
	defer f.Close()

	// bufio.Writer batches small writes into one syscall.
	w := bufio.NewWriter(f)
	for i := 1; i <= 3; i++ {
		fmt.Fprintf(w, "event %d occurred\n", i)
	}

	// Flush pushes the buffer to the file. WITHOUT THIS, DATA IS LOST.
	if err := w.Flush(); err != nil {
		fmt.Println("flush error:", err)
		return
	}
	fmt.Println("log written")
}
```

Forgetting `Flush()` is the most common `bufio.Writer` bug: the program exits
successfully and the file is empty. `defer w.Flush()` works, but its error is
discarded — flush explicitly when the data matters.

| Flag | Meaning |
|------|---------|
| `os.O_RDONLY` | read only |
| `os.O_WRONLY` | write only |
| `os.O_RDWR` | read and write |
| `os.O_CREATE` | create the file if it does not exist |
| `os.O_APPEND` | writes go to the end |
| `os.O_TRUNC` | truncate to zero length on open |
| `os.O_EXCL` | with `O_CREATE`, fail if the file already exists |

## The `defer f.Close()` discipline

`defer` schedules a call for when the surrounding **function** returns — not
the end of the loop or the block. That distinction causes a real bug:

```go
// BAD: every file stays open until processAll returns.
func processAll(paths []string) error {
	for _, p := range paths {
		f, err := os.Open(p)
		if err != nil {
			return err
		}
		defer f.Close() // 10,000 paths = 10,000 open descriptors
		// ... use f
	}
	return nil
}

// GOOD: give each file its own function scope.
func processAll(paths []string) error {
	for _, p := range paths {
		if err := processOne(p); err != nil {
			return err
		}
	}
	return nil
}

func processOne(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return fmt.Errorf("opening %s: %w", path, err)
	}
	defer f.Close() // runs at the end of THIS function -- correct
	// ... use f
	return nil
}
```

Also worth knowing: deferred calls run **last-in-first-out**, and their
arguments are evaluated at the moment `defer` is written, not when it runs.

```go
func demo() {
	x := 1
	defer fmt.Println("deferred sees:", x) // captures 1 right now
	x = 99
	fmt.Println("current:", x)
	// Output:
	// current: 99
	// deferred sees: 1
}
```

## Checking existence, size and type

Use `os.Stat` for metadata, and `errors.Is(err, os.ErrNotExist)` for the
"missing file" case:

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	info, err := os.Stat("notes.txt")
	switch {
	case errors.Is(err, os.ErrNotExist):
		fmt.Println("file does not exist")
	case err != nil:
		fmt.Println("stat error:", err)
	default:
		fmt.Printf("%s: %d bytes, dir=%v, mode=%v, modified %s\n",
			info.Name(), info.Size(), info.IsDir(), info.Mode(),
			info.ModTime().Format("2006-01-02 15:04"))
	}
}
```

Do not use a "check then open" pattern to avoid errors — the file can vanish
between the two calls. Just open it and handle the error, which is why the
`errors.Is` chain from [Module 4](04-custom-errors-wrapping.md) matters here.

## Directories and paths

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func main() {
	// filepath joins with the correct separator for the OS.
	dir := filepath.Join("data", "2026", "reports")

	if err := os.MkdirAll(dir, 0755); err != nil { // creates parents too
		fmt.Println("mkdir error:", err)
		return
	}

	entries, err := os.ReadDir("data")
	if err != nil {
		fmt.Println("readdir error:", err)
		return
	}
	for _, e := range entries {
		fmt.Println(e.Name(), "dir:", e.IsDir())
	}

	// Walk an entire tree.
	err = filepath.WalkDir("data", func(path string, d os.DirEntry, err error) error {
		if err != nil {
			return err // propagate access errors
		}
		if !d.IsDir() && filepath.Ext(path) == ".txt" {
			fmt.Println("found:", path)
		}
		return nil
	})
	if err != nil {
		fmt.Println("walk error:", err)
	}
}
```

Always build paths with `filepath.Join`, never string concatenation with
`"/"` — it keeps the code working on Windows and cleans up doubled separators.

## Temporary files and atomic writes

Writing a config or data file directly risks corruption if the program dies
mid-write. The safe pattern is write-to-temp-then-rename, because rename
within a filesystem is atomic:

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func atomicWrite(path string, data []byte) error {
	dir := filepath.Dir(path)

	tmp, err := os.CreateTemp(dir, ".tmp-*") // same dir => same filesystem
	if err != nil {
		return fmt.Errorf("creating temp file: %w", err)
	}
	tmpName := tmp.Name()
	defer os.Remove(tmpName) // no-op once the rename succeeds

	if _, err := tmp.Write(data); err != nil {
		tmp.Close()
		return fmt.Errorf("writing temp file: %w", err)
	}
	if err := tmp.Close(); err != nil {
		return fmt.Errorf("closing temp file: %w", err)
	}
	if err := os.Rename(tmpName, path); err != nil {
		return fmt.Errorf("renaming into place: %w", err)
	}
	return nil
}

func main() {
	if err := atomicWrite("config.json", []byte(`{"debug": true}`)); err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println("config written safely")
}
```

Readers of `config.json` see either the complete old version or the complete
new one — never a half-written file.

## Copying with `io.Copy`

Because `*os.File` implements both `io.Reader` and `io.Writer`, copying is one
call with constant memory use regardless of file size:

```go
src, err := os.Open("large.bin")
if err != nil {
	return err
}
defer src.Close()

dst, err := os.Create("copy.bin")
if err != nil {
	return err
}
defer dst.Close()

n, err := io.Copy(dst, src) // dst first, src second
```

The same function copies an HTTP response body to a file
([Module 8](08-net-http-client.md)) or a file to `os.Stdout`. That reuse is
the payoff of small interfaces from [Module 1](01-interfaces.md).

## Cheat sheet

| Task | Syntax |
|------|--------|
| Read a whole file | `data, err := os.ReadFile(path)` |
| Write a whole file | `os.WriteFile(path, data, 0644)` |
| Open for reading | `f, err := os.Open(path)` |
| Create / truncate | `f, err := os.Create(path)` |
| Open for appending | `os.OpenFile(p, os.O_APPEND\|os.O_CREATE\|os.O_WRONLY, 0644)` |
| Close reliably | `defer f.Close()` |
| Read line by line | `sc := bufio.NewScanner(f); for sc.Scan() { sc.Text() }` |
| Check scanner errors | `if err := sc.Err(); err != nil` |
| Buffered writing | `w := bufio.NewWriter(f)` … `w.Flush()` |
| Formatted write | `fmt.Fprintf(w, "x = %d\n", x)` |
| File metadata | `info, err := os.Stat(path)` |
| Missing-file check | `errors.Is(err, os.ErrNotExist)` |
| Delete / rename | `os.Remove(p)` / `os.Rename(old, new)` |
| Make directories | `os.MkdirAll(dir, 0755)` |
| List a directory | `os.ReadDir(dir)` |
| Walk a tree | `filepath.WalkDir(root, fn)` |
| Join paths | `filepath.Join("a", "b")` |
| Stream copy | `io.Copy(dst, src)` |

## Related lessons

- `io.Reader`/`io.Writer` as interfaces: [Module 1](01-interfaces.md).
- Wrapping filesystem errors with context:
  [Module 4](04-custom-errors-wrapping.md).
- Persisting structs as JSON files: [Module 5](05-json-encoding-decoding.md).
- Testing file code with `t.TempDir()`: [Module 6](06-testing-package.md).

## Exercise

Write a `wordcount` program that takes a filename as `os.Args[1]` and prints
the number of lines, words and characters, mimicking `wc`. Read the file with
`bufio.Scanner`, use `scanner.Split(bufio.ScanWords)` for the word pass (or
count with `strings.Fields` per line), and check `scanner.Err()`. Handle a
missing argument and a missing file with clear messages using
`errors.Is(err, os.ErrNotExist)`. Then extend it: if a second argument is
given, write the report to that path using the `atomicWrite` helper above
instead of printing.
