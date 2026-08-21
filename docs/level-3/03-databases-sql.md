# 03 · Databases & SQL

Go's standard library ships `database/sql` — a driver-agnostic API for
talking to relational databases. It never talks to a database directly;
you pair it with a driver package registered via a blank import. This
module uses `modernc.org/sqlite` (a pure-Go SQLite driver, no cgo required)
so every example below runs with nothing installed but `go run`.

## Connecting and running statements

```go
package main

import (
	"database/sql"
	"fmt"
	"log"

	_ "modernc.org/sqlite" // registers the "sqlite" driver via its init()
)

func main() {
	db, err := sql.Open("sqlite", "file:demo.db?mode=memory&cache=shared")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	_, err = db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL, age INTEGER)`)
	if err != nil {
		log.Fatal(err)
	}

	stmt, err := db.Prepare(`INSERT INTO users (name, age) VALUES (?, ?)`)
	if err != nil {
		log.Fatal(err)
	}
	defer stmt.Close()

	for _, u := range []struct {
		name string
		age  int
	}{{"Ada", 30}, {"Grace", 45}, {"Linus", 34}} {
		res, err := stmt.Exec(u.name, u.age)
		if err != nil {
			log.Fatal(err)
		}
		id, _ := res.LastInsertId()
		fmt.Printf("inserted %s with id=%d\n", u.name, id)
	}

	rows, err := db.Query(`SELECT id, name, age FROM users WHERE age > ? ORDER BY age`, 31)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	for rows.Next() {
		var id, age int
		var name string
		if err := rows.Scan(&id, &name, &age); err != nil {
			log.Fatal(err)
		}
		fmt.Printf("row: id=%d name=%s age=%d\n", id, name, age)
	}
	if err := rows.Err(); err != nil { // check AFTER the loop -- Next() swallows errors into this
		log.Fatal(err)
	}

	var missingName string
	err = db.QueryRow(`SELECT name FROM users WHERE id = ?`, 999).Scan(&missingName)
	fmt.Println("query for missing id, err:", err)
	fmt.Println("is ErrNoRows:", err == sql.ErrNoRows)
}
```

Real output:

```console
$ go run .
inserted Ada with id=1
inserted Grace with id=2
inserted Linus with id=3
row: id=3 name=Linus age=34
row: id=2 name=Grace age=45
query for missing id, err: sql: no rows in result set
is ErrNoRows: true
```

`sql.Open` does not actually connect — it just validates arguments and sets
up a connection pool lazily used by the first query. `?` placeholders are
always the right way to pass values; string-concatenating user input into
SQL is a SQL-injection bug regardless of language.

`db.QueryRow` returning `sql.ErrNoRows` through `.Scan()`, not through the
call to `QueryRow` itself, is a common source of nil-pointer-shaped bugs —
`QueryRow` never returns an error you can check directly; you must call
`.Scan` and compare against `sql.ErrNoRows`.

## Transactions

Multiple statements that must all succeed or all fail belong in a
transaction, obtained with `db.Begin()`:

```go
tx, err := db.Begin()
if err != nil {
	log.Fatal(err)
}
if _, err := tx.Exec(`UPDATE accounts SET balance = balance - 30 WHERE id = 1`); err != nil {
	tx.Rollback()
	log.Fatal(err)
}
if _, err := tx.Exec(`UPDATE accounts SET balance = balance + 30 WHERE id = 2`); err != nil {
	tx.Rollback()
	log.Fatal(err)
}
if err := tx.Commit(); err != nil {
	log.Fatal(err)
}
```

Starting from `balance=100` and `balance=50`, transferring 30 between the
two accounts and querying afterward:

```console
$ go run .
account 1 balance=70
account 2 balance=80
```

Both updates land together — there is no window where one account is
debited and the other isn't. `defer tx.Rollback()` right after `Begin()` is
a common pattern too: `Rollback` after a successful `Commit` is a documented
no-op, so deferring it unconditionally is safe and guarantees cleanup on any
early return.

## Go-specific traps

- **Not closing `*sql.Rows`.** Every `db.Query` result holds a connection
  from the pool until `rows.Close()` runs (or `Next()` is drained to `false`
  and the driver closes it for you) — `defer rows.Close()` immediately after
  checking the query error.
- **Checking `rows.Err()` after the loop, not just the initial `Query`
  error.** A dropped connection mid-scan surfaces there, not as a panic.
- **`db.Exec`'s `Result.LastInsertId()` and `RowsAffected()` are not
  supported by every driver** — check the driver's docs; on some (e.g. some
  Postgres drivers) calling them returns an error.
- **Forgetting `defer db.Close()`** leaks the whole connection pool, not
  just one connection — `*sql.DB` is meant to be a long-lived, shared handle,
  not something opened per-request.
- **A `*sql.DB` is safe for concurrent use** — it's a pool, not a single
  connection — so there is no need to wrap it in your own mutex the way the
  hand-rolled `store` in [Module 2](02-building-rest-apis.md) needed one.

## Cheat sheet

| Task | API |
|---|---|
| Register a driver | `import _ "modernc.org/sqlite"` (blank import for its `init()`) |
| Open a pooled handle | `db, err := sql.Open(driverName, dsn)` |
| Run DDL/DML, no rows back | `db.Exec(query, args...)` |
| Reusable prepared statement | `db.Prepare(query)` then `stmt.Exec(args...)` |
| Query multiple rows | `rows, err := db.Query(query, args...)`; `defer rows.Close()` |
| Query exactly one row | `db.QueryRow(query, args...).Scan(&dest...)` |
| No matching row | compare scan error to `sql.ErrNoRows` |
| Multi-statement atomicity | `tx, _ := db.Begin()`; `tx.Exec(...)`; `tx.Commit()` / `tx.Rollback()` |

## Related lessons

- Building the HTTP handlers that sit in front of a database:
  [Module 2](02-building-rest-apis.md).
- The [Level 3 project](10-project-rest-api-sqlite.md) wires this exact
  `database/sql` + SQLite pairing into a full REST API.

## Exercise

Extend the `users` table example with a `posts` table
(`id, user_id, title`) and a foreign key to `users(id)`. Write a query using
a `JOIN` that returns each user's name alongside their post titles, ordered
by user name. Then wrap "delete a user" and "delete all their posts" in a
single transaction, and prove atomicity by making the second `Exec`
deliberately fail (e.g. a bad column name) and confirming — via a follow-up
`SELECT` — that the user row is still present after `tx.Rollback()`.
