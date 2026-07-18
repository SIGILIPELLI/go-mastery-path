# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: structs,
slices, pointers, error handling, and packages/modules, with persistence to
a JSON file on disk.

## What you'll build

A command-line to-do app that:

- Adds tasks
- Lists all tasks (done and pending)
- Marks a task done by its number
- Deletes a task by its number
- Persists everything to `tasks.json` between runs

## Project layout

```text
todocli/
    go.mod
    main.go
    todo/
        task.go
        store.go
```

## Setting up the module

```bash
mkdir todocli && cd todocli
go mod init todocli
mkdir todo
```

## todo/task.go — the data type

```go
// todo/task.go
package todo

// Task is a single to-do item.
type Task struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}
```

## todo/store.go — JSON persistence

```go
// todo/store.go
package todo

import (
	"encoding/json"
	"fmt"
	"os"
)

// Store manages loading and saving Tasks to a JSON file.
type Store struct {
	path string
}

// NewStore returns a Store backed by the given file path.
func NewStore(path string) *Store {
	return &Store{path: path}
}

// Load reads all tasks from disk. A missing file is not an error --
// it just means there are no tasks yet.
func (s *Store) Load() ([]Task, error) {
	data, err := os.ReadFile(s.path)
	if os.IsNotExist(err) {
		return []Task{}, nil
	}
	if err != nil {
		return nil, fmt.Errorf("reading tasks file: %w", err)
	}

	var tasks []Task
	if err := json.Unmarshal(data, &tasks); err != nil {
		return nil, fmt.Errorf("parsing tasks file: %w", err)
	}
	return tasks, nil
}

// Save writes all tasks back to disk as indented JSON.
func (s *Store) Save(tasks []Task) error {
	data, err := json.MarshalIndent(tasks, "", "  ")
	if err != nil {
		return fmt.Errorf("encoding tasks: %w", err)
	}
	if err := os.WriteFile(s.path, data, 0644); err != nil {
		return fmt.Errorf("writing tasks file: %w", err)
	}
	return nil
}

// NextID returns the smallest unused, positive ID for a new task.
func NextID(tasks []Task) int {
	max := 0
	for _, t := range tasks {
		if t.ID > max {
			max = t.ID
		}
	}
	return max + 1
}
```

`%w` in `fmt.Errorf` wraps the underlying error so callers can inspect the
original cause with `errors.Is`/`errors.Unwrap` — you'll use this properly
in [Level 2, Module 4](../level-2/04-custom-errors-wrapping.md).

## main.go — the CLI

```go
// main.go
package main

import (
	"fmt"
	"os"
	"strconv"

	"todocli/todo"
)

const dataFile = "tasks.json"

func main() {
	store := todo.NewStore(dataFile)
	tasks, err := store.Load()
	if err != nil {
		fmt.Println("error loading tasks:", err)
		os.Exit(1)
	}

	if len(os.Args) < 2 {
		printUsage()
		return
	}

	command := os.Args[1]
	args := os.Args[2:]

	switch command {
	case "add":
		tasks, err = addTask(tasks, args)
	case "list":
		listTasks(tasks)
		return
	case "done":
		tasks, err = markDone(tasks, args)
	case "delete":
		tasks, err = deleteTask(tasks, args)
	default:
		printUsage()
		return
	}

	if err != nil {
		fmt.Println("error:", err)
		os.Exit(1)
	}

	if err := store.Save(tasks); err != nil {
		fmt.Println("error saving tasks:", err)
		os.Exit(1)
	}
}

func printUsage() {
	fmt.Println("Usage: todocli [add <title> | list | done <id> | delete <id>]")
}

func addTask(tasks []todo.Task, args []string) ([]todo.Task, error) {
	if len(args) != 1 {
		return tasks, fmt.Errorf("add requires exactly one quoted title")
	}
	newTask := todo.Task{
		ID:    todo.NextID(tasks),
		Title: args[0],
		Done:  false,
	}
	tasks = append(tasks, newTask)
	fmt.Printf("Added #%d: %s\n", newTask.ID, newTask.Title)
	return tasks, nil
}

func listTasks(tasks []todo.Task) {
	if len(tasks) == 0 {
		fmt.Println("No tasks yet.")
		return
	}
	for _, t := range tasks {
		status := " "
		if t.Done {
			status = "x"
		}
		fmt.Printf("[%s] #%d %s\n", status, t.ID, t.Title)
	}
}

func markDone(tasks []todo.Task, args []string) ([]todo.Task, error) {
	id, err := parseID(args)
	if err != nil {
		return tasks, err
	}
	for i := range tasks {
		if tasks[i].ID == id {
			tasks[i].Done = true
			fmt.Printf("Marked #%d done\n", id)
			return tasks, nil
		}
	}
	return tasks, fmt.Errorf("no task with id %d", id)
}

func deleteTask(tasks []todo.Task, args []string) ([]todo.Task, error) {
	id, err := parseID(args)
	if err != nil {
		return tasks, err
	}
	remaining := make([]todo.Task, 0, len(tasks))
	found := false
	for _, t := range tasks {
		if t.ID == id {
			found = true
			continue
		}
		remaining = append(remaining, t)
	}
	if !found {
		return tasks, fmt.Errorf("no task with id %d", id)
	}
	fmt.Printf("Deleted #%d\n", id)
	return remaining, nil
}

func parseID(args []string) (int, error) {
	if len(args) != 1 {
		return 0, fmt.Errorf("expected exactly one id argument")
	}
	id, err := strconv.Atoi(args[0])
	if err != nil {
		return 0, fmt.Errorf("invalid id %q: %w", args[0], err)
	}
	return id, nil
}
```

## Running it

```bash
go run . add "Buy milk"
# Added #1: Buy milk

go run . add "Write report"
# Added #2: Write report

go run . list
# [ ] #1 Buy milk
# [ ] #2 Write report

go run . done 1
# Marked #1 done

go run . list
# [x] #1 Buy milk
# [ ] #2 Write report

go run . delete 2
# Deleted #2

go run . list
# [x] #1 Buy milk
```

Each invocation reloads `tasks.json` from disk, so tasks persist across
separate runs of the program. Build a standalone binary with `go build -o
todocli .` and `./todocli list` works the same way, no `go run` needed.

## Stretch goals

- Add a `priority` field (`low`/`medium`/`high`) and sort `list` output by
  it.
- Add a `--filter=done` / `--filter=pending` flag to `list` (you'll build
  proper flag parsing with the `flag` package in
  [Level 3, Module 8](../level-3/08-building-clis.md)).
- Write unit tests for `NextID` and the add/done/delete logic (formalized
  with the `testing` package in
  [Level 2, Module 6](../level-2/06-testing-package.md)).
- Switch from a full-file rewrite on every save to an append-only log file.

Completing this project means you're ready for **Level 2 · Intermediate**.
