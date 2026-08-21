# 08 · Security Best Practices

Security bugs in Go services tend to cluster around a handful of concrete
mistakes: unsalted or fast password hashes, timing-leaky comparisons for
secrets, and SQL built by string concatenation. This module demonstrates
the right and wrong version of each, with real output showing exactly what
the wrong version produces.

## Password hashing with bcrypt

```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(pw string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(pw), bcrypt.DefaultCost)
	return string(hash), err
}

func checkPassword(pw, hash string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(pw)) == nil
}
```

```console
$ go run .
hash starts with: $2a$10$
check correct password: true
check wrong password: false
```

`bcrypt.GenerateFromPassword` bakes a random salt into the output hash
automatically — never store passwords with `sha256.Sum256([]byte(pw))`
alone; a fast, unsalted hash lets an attacker with the database precompute
a rainbow table and crack every password at once. `bcrypt.DefaultCost`
(10) picks a work factor slow enough to make brute-forcing expensive per
guess; raising it further slows down both attackers and your own login
endpoint, so it's a real tradeoff, not a free win.

## Constant-time comparison for secrets

```go
import (
	"crypto/hmac"
	"crypto/sha256"
	"crypto/subtle"
)

func sign(secret, msg []byte) []byte {
	mac := hmac.New(sha256.New, secret)
	mac.Write(msg)
	return mac.Sum(nil)
}

func verify(secret, msg, sig []byte) bool {
	expected := sign(secret, msg)
	return subtle.ConstantTimeCompare(expected, sig) == 1
}
```

```console
$ go run .
verify valid sig: true
verify tampered msg: false
```

This is the shape of verifying a webhook signature (Stripe, GitHub, etc.)
or an API token. `subtle.ConstantTimeCompare` matters specifically because
a plain `bytes.Equal` or `==` short-circuits on the *first* mismatched
byte — an attacker who can measure response timing precisely enough can
use that to guess a valid signature one byte at a time. `subtle`'s version
always compares every byte regardless of where a mismatch occurs, closing
that side channel.

## SQL injection: what NOT to do, and the fix

```go
// vulnerable is what NOT to do -- string concatenation into SQL.
func vulnerableQuery(db *sql.DB, username string) (*sql.Rows, error) {
	q := "SELECT id FROM users WHERE username = '" + username + "'"
	return db.Query(q)
}

// safe uses a parameterized query -- the driver escapes the value.
func safeQuery(db *sql.DB, username string) (*sql.Rows, error) {
	return db.Query("SELECT id FROM users WHERE username = ?", username)
}
```

Feeding a classic injection payload through the vulnerable version shows
exactly why it's dangerous:

```console
$ go run .
vulnerable query would be: SELECT id FROM users WHERE username = '' OR '1'='1'
safe query keeps it a literal parameter: ? = ' OR '1'='1
```

The vulnerable query, once assembled, is
`... WHERE username = '' OR '1'='1'` — a condition that's always true,
returning every row in the table instead of the intended single user. The
parameterized version never assembles a string at all: `?` is a
placeholder the driver binds the value to *as data*, so `' OR '1'='1` is
compared literally against the `username` column and (correctly) matches
nothing. This is exactly the placeholder pattern from
[Level 3, Module 3](../level-3/03-databases-sql.md) — the security argument
is the other, equally important reason to always use it.

## Rate limiting

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(2, 1) // 2 events/sec, burst of 1
allowed := 0
for i := 0; i < 5; i++ {
	if limiter.Allow() {
		allowed++
	}
}
fmt.Println("requests allowed immediately out of 5:", allowed)
```

```console
$ go run .
requests allowed immediately out of 5: 1
```

With a burst size of 1, only the very first of five back-to-back calls to
`Allow()` succeeds — the other four are refused instantly because no time
has passed to refill the token bucket at the configured 2-per-second rate.
Wrapped in HTTP middleware (`if !limiter.Allow() { http.Error(w, ...,
http.StatusTooManyRequests); return }`), this is the standard way to cap
abuse of a login or password-reset endpoint without needing an external
rate-limiting proxy.

## Go-specific traps

- **`crypto/md5` and `crypto/sha1` are still in the standard library** and
  still get reached for out of habit — both are cryptographically broken
  for anything security-sensitive (password hashing, signatures); use
  `bcrypt`/`argon2` for passwords and `sha256`/`sha512` for everything else
  that needs a real hash.
- **`math/rand` is not cryptographically secure** — a token, session ID, or
  API key generated with `math/rand` is guessable given enough samples;
  use `crypto/rand` for anything that needs unpredictability an attacker
  can't reconstruct.
- **String concatenation into SQL is not "usually fine for internal
  tools"** — the moment any part of the string touches user input,
  including indirectly (a value from another service, a config file an
  attacker might influence), it's a real vulnerability regardless of how
  "internal" the tool is.
- **`bcrypt.GenerateFromPassword` has a 72-byte input limit** — silently
  truncating longer passwords rather than erroring in older versions of
  the library; hash a pre-image (e.g. `sha256.Sum256([]byte(pw))`) first if
  you need to support arbitrarily long passphrases safely.
- **A `rate.Limiter` is per-process, in-memory** — it does not coordinate
  across multiple replicas of a service behind a load balancer; a
  distributed system needs a shared store (Redis, etc.) for a rate limit
  that holds across instances.

## Cheat sheet

| Need | API |
|---|---|
| Hash a password | `bcrypt.GenerateFromPassword([]byte(pw), bcrypt.DefaultCost)` |
| Verify a password | `bcrypt.CompareHashAndPassword(hash, pw) == nil` |
| Sign a message | `hmac.New(sha256.New, secret)` then `.Write`/`.Sum(nil)` |
| Compare secrets safely | `subtle.ConstantTimeCompare(a, b) == 1` |
| Cryptographically secure random bytes | `crypto/rand.Read(buf)` |
| Parameterized SQL | `db.Query("... WHERE x = ?", val)` — never string-concatenate |
| Per-endpoint rate limiting | `rate.NewLimiter(eventsPerSecond, burst)`, `.Allow()` |

## Related lessons

- Parameterized queries and `database/sql` fundamentals:
  [Level 3, Module 3](../level-3/03-databases-sql.md).
- Middleware structure a rate limiter or auth check plugs into:
  [Level 3, Module 2](../level-3/02-building-rest-apis.md) and
  [Module 4](04-production-grade-apis.md).

## Exercise

Add HTTP middleware wrapping a `/login` handler with a `rate.Limiter` keyed
per-IP (a `map[string]*rate.Limiter` guarded by a mutex, or `sync.Map` from
[Module 1](01-advanced-concurrency.md)) so each client IP gets its own
budget instead of one global limiter starving every user. Then write a
`/webhook` handler that reads an `X-Signature` header, computes the
expected HMAC over the raw request body, and rejects the request with 401
using `subtle.ConstantTimeCompare` — confirm with `curl` that a tampered
body (same signature, different payload) is correctly rejected.
