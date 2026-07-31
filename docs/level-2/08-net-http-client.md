# 08 · Working with net/http

## Talking to the web from Go

`net/http` is one of the standard library's crown jewels: a production-quality
HTTP client and server with no dependencies. This module covers the **client**
side — fetching data from APIs, posting JSON, setting headers and handling
failures. (Building servers comes in
[Level 3, Module 2](../level-3/02-building-rest-apis.md).)

## The simplest possible request

```go
package main

import (
	"fmt"
	"io"
	"net/http"
)

func main() {
	resp, err := http.Get("https://api.github.com/zen")
	if err != nil {
		fmt.Println("request failed:", err)
		return
	}
	defer resp.Body.Close() // ALWAYS -- otherwise the connection leaks

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("reading body:", err)
		return
	}

	fmt.Println("status:", resp.Status)     // status: 200 OK
	fmt.Println("body:", string(body))      // a random GitHub design proverb
	fmt.Println("code:", resp.StatusCode)   // code: 200
}
```

Three non-negotiables in that snippet:

1. **`defer resp.Body.Close()`** — the body is an `io.ReadCloser` backed by a
   live TCP connection. Not closing it leaks connections until the process
   runs out of file descriptors.
2. **`err != nil` only covers transport failures** — DNS failure, refused
   connection, timeout. A `404` or `500` is a perfectly successful *request*.
   You must check `resp.StatusCode` yourself.
3. **Read the body even if you discard it.** Draining it lets the connection
   be reused; abandoning it forces a new one.

## Checking status codes properly

```go
package main

import (
	"fmt"
	"io"
	"net/http"
)

func fetch(url string) ([]byte, error) {
	resp, err := http.Get(url)
	if err != nil {
		return nil, fmt.Errorf("get %s: %w", url, err)
	}
	defer resp.Body.Close()

	// 2xx is success; anything else is an application-level failure.
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		// Read a little of the body -- APIs put the reason there.
		snippet, _ := io.ReadAll(io.LimitReader(resp.Body, 512))
		return nil, fmt.Errorf("get %s: unexpected status %s: %s",
			url, resp.Status, snippet)
	}

	return io.ReadAll(resp.Body)
}

func main() {
	if _, err := fetch("https://httpbin.org/status/404"); err != nil {
		fmt.Println(err)
		// get https://httpbin.org/status/404: unexpected status 404 NOT FOUND:
	}
}
```

`io.LimitReader` guards against a hostile or broken server streaming gigabytes
into your error message.

## Decoding a JSON response

Combine this with [Module 5](05-json-encoding-decoding.md). Stream the body
straight into a decoder rather than reading it into a `[]byte` first:

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

type Repo struct {
	Name        string `json:"name"`
	FullName    string `json:"full_name"`
	Stars       int    `json:"stargazers_count"`
	Description string `json:"description"`
	Language    string `json:"language"`
}

func getRepo(owner, name string) (*Repo, error) {
	url := fmt.Sprintf("https://api.github.com/repos/%s/%s", owner, name)

	client := &http.Client{Timeout: 10 * time.Second}
	resp, err := client.Get(url)
	if err != nil {
		return nil, fmt.Errorf("fetching repo: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("github returned %s", resp.Status)
	}

	var repo Repo
	// Decode reads directly from the stream -- no intermediate buffer.
	if err := json.NewDecoder(resp.Body).Decode(&repo); err != nil {
		return nil, fmt.Errorf("decoding response: %w", err)
	}
	return &repo, nil
}

func main() {
	repo, err := getRepo("golang", "go")
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Printf("%s (%s) — %d stars\n", repo.FullName, repo.Language, repo.Stars)
	// golang/go (Go) — 120000+ stars
}
```

This endpoint is public and needs no API key, so the program runs as written.

## Never use `http.DefaultClient` in production

`http.Get`, `http.Post` and friends use `http.DefaultClient`, which has **no
timeout**. A server that accepts your connection and then never responds will
hang your program forever. Always construct your own client:

```go
client := &http.Client{
	Timeout: 10 * time.Second, // covers dial + redirects + reading the body
}
```

For finer control, configure the transport:

```go
client := &http.Client{
	Timeout: 30 * time.Second,
	Transport: &http.Transport{
		MaxIdleConns:        100,
		MaxIdleConnsPerHost: 10,
		IdleConnTimeout:     90 * time.Second,
	},
}
```

Create **one** client and reuse it for the life of the program. A new client
per request throws away the connection pool and its keep-alives. Clients are
safe for concurrent use by multiple goroutines.

## Custom requests: headers, auth, POST bodies

`http.NewRequest` gives you full control over method, headers and body:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type CreatePayload struct {
	Title string   `json:"title"`
	Tags  []string `json:"tags"`
}

func main() {
	payload := CreatePayload{Title: "Hello", Tags: []string{"go", "http"}}

	body, err := json.Marshal(payload)
	if err != nil {
		fmt.Println("encoding payload:", err)
		return
	}

	req, err := http.NewRequest(http.MethodPost,
		"https://httpbin.org/post", bytes.NewReader(body))
	if err != nil {
		fmt.Println("building request:", err)
		return
	}

	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("User-Agent", "go-mastery-path/1.0")
	// Read secrets from the environment -- never hard-code them.
	if token := os.Getenv("API_TOKEN"); token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("request failed:", err)
		return
	}
	defer resp.Body.Close()

	out, _ := io.ReadAll(io.LimitReader(resp.Body, 2048))
	fmt.Println(resp.Status)
	fmt.Println(string(out))
}
```

Use `req.Header.Set` to replace a header and `req.Header.Add` to append
another value to the same key.

## Query parameters

Build query strings with `url.Values` so values are escaped correctly —
manual string concatenation breaks on spaces, `&` and non-ASCII characters:

```go
package main

import (
	"fmt"
	"net/url"
)

func main() {
	base, _ := url.Parse("https://api.example.com/search")

	q := url.Values{}
	q.Set("q", "go programming & concurrency")
	q.Set("limit", "20")
	q.Set("lang", "en")
	base.RawQuery = q.Encode()

	fmt.Println(base.String())
	// https://api.example.com/search?lang=en&limit=20&q=go+programming+%26+concurrency
}
```

## Timeouts, cancellation and retries

`http.NewRequestWithContext` attaches a context so a request can be cancelled
or given its own deadline, independent of the client-wide timeout:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"time"
)

func fetchWithRetry(ctx context.Context, url string, attempts int) (*http.Response, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	var lastErr error

	for i := 0; i < attempts; i++ {
		if i > 0 {
			// Exponential backoff: 200ms, 400ms, 800ms...
			backoff := time.Duration(200*(1<<uint(i-1))) * time.Millisecond
			select {
			case <-time.After(backoff):
			case <-ctx.Done():
				return nil, ctx.Err() // caller cancelled while we waited
			}
		}

		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return nil, err // a malformed URL will never succeed -- don't retry
		}

		resp, err := client.Do(req)
		if err != nil {
			lastErr = err
			continue
		}
		// Retry only on server errors; 4xx means WE are wrong.
		if resp.StatusCode >= 500 {
			resp.Body.Close()
			lastErr = fmt.Errorf("server error %s", resp.Status)
			continue
		}
		return resp, nil
	}
	return nil, fmt.Errorf("after %d attempts: %w", attempts, lastErr)
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	resp, err := fetchWithRetry(ctx, "https://api.github.com/zen", 3)
	if err != nil {
		if errors.Is(err, context.DeadlineExceeded) {
			fmt.Println("gave up: overall deadline exceeded")
			return
		}
		fmt.Println("error:", err)
		return
	}
	defer resp.Body.Close()
	fmt.Println("succeeded:", resp.Status)
}
```

Retrying `4xx` responses is pointless and can get you rate-limited — only
`5xx` and transport errors deserve a second attempt. `context` is covered
properly in [Level 3, Module 4](../level-3/04-context-package.md).

## Downloading a file without loading it into memory

```go
resp, err := client.Get(fileURL)
if err != nil {
	return err
}
defer resp.Body.Close()

out, err := os.Create("download.bin")
if err != nil {
	return err
}
defer out.Close()

_, err = io.Copy(out, resp.Body) // streams in fixed-size chunks
```

Because `resp.Body` is an `io.Reader` and `*os.File` is an `io.Writer`, the
same `io.Copy` from [Module 7](07-file-io.md) works unchanged.

## Cheat sheet

| Task | Syntax |
|------|--------|
| Quick GET | `resp, err := http.Get(url)` |
| Always close the body | `defer resp.Body.Close()` |
| Read the whole body | `io.ReadAll(resp.Body)` |
| Bounded read | `io.ReadAll(io.LimitReader(resp.Body, 1<<20))` |
| Decode JSON | `json.NewDecoder(resp.Body).Decode(&v)` |
| Client with a timeout | `&http.Client{Timeout: 10 * time.Second}` |
| Build a request | `http.NewRequest(method, url, body)` |
| With cancellation | `http.NewRequestWithContext(ctx, ...)` |
| Send it | `resp, err := client.Do(req)` |
| Set a header | `req.Header.Set("Authorization", "Bearer "+t)` |
| JSON POST body | `bytes.NewReader(jsonBytes)` |
| Query parameters | `u.RawQuery = url.Values{...}.Encode()` |
| Status constants | `http.StatusOK`, `http.StatusNotFound` |
| Stream to disk | `io.Copy(file, resp.Body)` |

## Related lessons

- Decoding responses into structs: [Module 5](05-json-encoding-decoding.md).
- Saving downloads and streaming with `io.Copy`: [Module 7](07-file-io.md).
- Wrapping network errors with context:
  [Module 4](04-custom-errors-wrapping.md).
- Put it all together in the [Weather CLI project](10-project-weather-cli.md).
- Writing servers: [Level 3, Module 2](../level-3/02-building-rest-apis.md).

## Exercise

Write a program that fetches `https://api.github.com/users/<username>` for a
username given on the command line and prints the login, public repo count and
account creation date. Define a struct with only the three fields you need,
decode with `json.NewDecoder`, use a client with a 10-second timeout, and
handle the `404` case with a friendly "no such user" message rather than a raw
status dump. Then extend it to accept multiple usernames and fetch them
concurrently with goroutines and a `sync.WaitGroup`
([Module 2](02-goroutines-channels.md)), collecting results on a channel.
