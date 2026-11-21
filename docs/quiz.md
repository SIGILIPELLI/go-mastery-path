---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Go Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\n\nfunc main() {\nx := 7\nfmt.Println(x/2, float64(x)/2)\n}\n```\n",
    "options": [
      "3 3.5",
      "3.5 3.5",
      "3 3",
      "4 3.5"
    ],
    "correctIndex": 0,
    "explanation": "Integer division truncates; convert explicitly. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\n\nfunc main() {\na := []int{1, 2, 3}\nb := a[:2]\nb = append(b, 99)\nfmt.Println(a, b)\n}\n```\n",
    "options": [
      "[1 2 99] [1 2 99]",
      "[1 2 99] [1 2]",
      "[1 2 3] [1 2 99]",
      "Error"
    ],
    "correctIndex": 0,
    "explanation": "Appending within capacity overwrites the shared array. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\n\nfunc main() {\nm := map[string]int{}\nv, ok := m[\"x\"]\nm[\"x\"]++\nfmt.Println(v, ok, m[\"x\"])\n}\n```\n",
    "options": [
      "Error",
      "nil false 1",
      "0 false 1",
      "0 true 1"
    ],
    "correctIndex": 2,
    "explanation": "Missing keys give zero values. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\ntype Point struct{ X, Y int }\nfunc main() {\np := Point{1, 2}\nq := p\nq.X = 9\nfmt.Println(p, q)\n}\n```\n",
    "options": [
      "Error",
      "{1 2} {9 2}",
      "{9 2} {9 2}",
      "{1 2} {1 2}"
    ],
    "correctIndex": 1,
    "explanation": "Structs copy on assignment. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\nfunc inc(n int) { n++ }\nfunc incP(n *int) { *n++ }\nfunc main() {\na := 1\ninc(a)\nb := 1\nincP(&b)\nfmt.Println(a, b)\n}\n```\n",
    "options": [
      "1 2",
      "1 1",
      "2 2",
      "2 1"
    ],
    "correctIndex": 0,
    "explanation": "Pass a pointer to modify. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"errors\"; \"fmt\")\nvar ErrNotFound = errors.New(\"not found\")\nfunc main() {\ne := fmt.Errorf(\"load: %w\", ErrNotFound)\nfmt.Println(errors.Is(e, ErrNotFound), e)\n}\n```\n",
    "options": [
      "true load: not found",
      "Error",
      "false load: not found",
      "true not found"
    ],
    "correctIndex": 0,
    "explanation": "%w keeps the chain for errors.Is. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\n\nfunc main() {\ndefer fmt.Println(\"one\")\ndefer fmt.Println(\"two\")\nfmt.Println(\"body\")\n}\n```\n",
    "options": [
      "body\ntwo\none",
      "body\none\ntwo",
      "Error",
      "one\ntwo\nbody"
    ],
    "correctIndex": 0,
    "explanation": "Deferred calls run last-in first-out. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\ntype Shape interface{ Area() int }\ntype Square struct{ S int }\nfunc (q Square) Area() int { return q.S * q.S }\nfunc main() {\nvar s Shape = Square{3}\nfmt.Println(s.Area())\n}\n```\n",
    "options": [
      "Error",
      "3",
      "6",
      "9"
    ],
    "correctIndex": 3,
    "explanation": "Implicit interface satisfaction. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\"; \"sync\")\n\nfunc main() {\nvar wg sync.WaitGroup\nvar mu sync.Mutex\nn := 0\nfor i := 0; i < 50; i++ {\n wg.Add(1)\n go func() { defer wg.Done(); mu.Lock(); n++; mu.Unlock() }()\n}\nwg.Wait()\nfmt.Println(n)\n}\n```\n",
    "options": [
      "50",
      "0",
      "Race",
      "Less than 50"
    ],
    "correctIndex": 0,
    "explanation": "A mutex protects shared state. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\n\nfunc main() {\nch := make(chan int, 2)\nch <- 1\nch <- 2\nclose(ch)\nfor v := range ch { fmt.Print(v, \" \") }\nv, ok := <-ch\nfmt.Println(v, ok)\n}\n```\n",
    "options": [
      "1 2 0 false",
      "1 2",
      "Deadlock",
      "1 2 0 true"
    ],
    "correctIndex": 0,
    "explanation": "Receiving from a closed channel yields the zero value with ok=false. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"context\"; \"fmt\"; \"time\")\n\nfunc main() {\nctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)\ndefer cancel()\n<-ctx.Done()\nfmt.Println(ctx.Err())\n}\n```\n",
    "options": [
      "nil",
      "context deadline exceeded",
      "Error",
      "context canceled"
    ],
    "correctIndex": 1,
    "explanation": "Contexts carry deadlines and cancellation. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```go\npackage main\nimport (\"fmt\")\ntype Ordered interface{ ~int | ~string }\nfunc Max[T Ordered](a, b T) T { if a > b { return a }; return b }\nfunc main() {\nfmt.Println(Max(3, 9), Max(\"pear\", \"apple\"))\n}\n```\n",
    "options": [
      "9 apple",
      "3 apple",
      "Error",
      "9 pear"
    ],
    "correctIndex": 3,
    "explanation": "Generic functions with a constraint. Verified by compiling and running it."
  },
  {
    "question": "A `for range` loop starts goroutines that all print the final loop value in older Go. What's the standard fix?",
    "options": [
      "Pass the variable as an argument or copy it per iteration (Go 1.22+ scopes it per iteration)",
      "Mutex",
      "Channel",
      "Sleep"
    ],
    "correctIndex": 0,
    "explanation": "Loop variable capture."
  },
  {
    "question": "How do you make an HTTP client call safe against hangs?",
    "options": [
      "Set a timeout (client Timeout or request context) and always close the response body",
      "Nothing",
      "Retry",
      "Log"
    ],
    "correctIndex": 0,
    "explanation": "The default client has no timeout."
  },
  {
    "question": "Which tool finds data races?",
    "options": [
      "pprof",
      "go vet",
      "go fmt",
      "go test -race"
    ],
    "correctIndex": 3,
    "explanation": "Run it in CI."
  },
  {
    "question": "Where should the DB handle live in a handler-based service?",
    "options": [
      "In a struct injected into handlers, created once at startup",
      "Context",
      "Global",
      "Request"
    ],
    "correctIndex": 0,
    "explanation": "sql.DB is a pool."
  },
  {
    "question": "How do you shut an HTTP server down gracefully?",
    "options": [
      "Kill",
      "os.Exit",
      "Close",
      "Call server.Shutdown(ctx) when SIGTERM arrives so in-flight requests finish"
    ],
    "correctIndex": 3,
    "explanation": "Production readiness."
  },
  {
    "question": "A container image for a Go service is 900 MB. What's the fix?",
    "options": [
      "Compress",
      "Bigger base",
      "Multi-stage build copying a static binary into a minimal base image",
      "More layers"
    ],
    "correctIndex": 2,
    "explanation": "CGO_ENABLED=0."
  },
  {
    "question": "How do you find a performance bottleneck?",
    "options": [
      "Rewrite",
      "Guess",
      "Benchmark and profile (pprof) before optimizing",
      "Cache"
    ],
    "correctIndex": 2,
    "explanation": "Measure first."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A worker pool with errors and cancellation

Complete squareAll: results are collected by original index, negative inputs produce per-item errors ("item N: negative input") combined with errors.Join in sorted order, and a cancelled context returns ctx.Err(). Three runs show success, partial failure and cancellation.

**Starter code**

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"sort"
	"sync"
)

type result struct {
	id  int
	val int
	err error
}

// squareAll squares each number using `workers` goroutines. It stops early when ctx is cancelled.
func squareAll(ctx context.Context, nums []int, workers int) ([]int, error) {
	jobs := make(chan int)
	results := make(chan result)
	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := range jobs {
				if nums[i] < 0 {
					results <- result{id: i, err: errors.New("negative input")}
					continue
				}
				results <- result{id: i, val: nums[i] * nums[i]}
			}
		}()
	}
	go func() {
		defer close(jobs)
		for i := range nums {
			select {
			case jobs <- i:
			case <-ctx.Done():
				return
			}
		}
	}()
	go func() { wg.Wait(); close(results) }()

	// collect results into out (index = original position); gather per-item errors as
	// fmt.Errorf("item %d: %w", id, err), sort them, and return errors.Join(...);
	// when there are no item errors, return out and ctx.Err()
	return nil, nil
}

func main() {
	out, err := squareAll(context.Background(), []int{1, 2, 3, 4, 5}, 3)
	fmt.Println(out, err)
	out, err = squareAll(context.Background(), []int{2, -3, 4, -1}, 2)
	fmt.Println(out, err)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()
	_, err = squareAll(ctx, []int{1, 2, 3}, 2)
	fmt.Println(err)
}
```

**Expected output**

```text
[1 4 9 16 25] <nil>
[4 0 16 0] item 1: negative input
item 3: negative input
context canceled
```

<details><summary>Show solution and explanation</summary>

<pre><code>package main

import (
	&quot;context&quot;
	&quot;errors&quot;
	&quot;fmt&quot;
	&quot;sort&quot;
	&quot;sync&quot;
)

type result struct {
	id  int
	val int
	err error
}

// squareAll squares each number using `workers` goroutines. It stops early when ctx is cancelled.
func squareAll(ctx context.Context, nums []int, workers int) ([]int, error) {
	jobs := make(chan int)
	results := make(chan result)
	var wg sync.WaitGroup
	for w := 0; w &lt; workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := range jobs {
				if nums[i] &lt; 0 {
					results &lt;- result{id: i, err: errors.New(&quot;negative input&quot;)}
					continue
				}
				results &lt;- result{id: i, val: nums[i] * nums[i]}
			}
		}()
	}
	go func() {
		defer close(jobs)
		for i := range nums {
			select {
			case jobs &lt;- i:
			case &lt;-ctx.Done():
				return
			}
		}
	}()
	go func() { wg.Wait(); close(results) }()

	out := make([]int, len(nums))
	var errs []error
	for r := range results {
		if r.err != nil {
			errs = append(errs, fmt.Errorf(&quot;item %d: %w&quot;, r.id, r.err))
			continue
		}
		out[r.id] = r.val
	}
	if len(errs) &gt; 0 {
		sort.Slice(errs, func(i, j int) bool { return errs[i].Error() &lt; errs[j].Error() })
		return out, errors.Join(errs...)
	}
	return out, ctx.Err()
}

func main() {
	out, err := squareAll(context.Background(), []int{1, 2, 3, 4, 5}, 3)
	fmt.Println(out, err)
	out, err = squareAll(context.Background(), []int{2, -3, 4, -1}, 2)
	fmt.Println(out, err)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()
	_, err = squareAll(ctx, []int{1, 2, 3}, 2)
	fmt.Println(err)
}
</code></pre>

<p>Workers read indexes from a jobs channel, results carry their original index so order is preserved, a goroutine closes results after the WaitGroup, and the producer stops on ctx.Done(). Verified by running the solution with go run.</p>
</details>

### Test an HTTP handler with httptest

Implement the handler: POST /items (body size limit, 400 on invalid or empty name, 201 with the stored item) and GET /items/{id} (200 or 404 for unknown/non-numeric ids), guarded by a mutex. The do() helper drives it with httptest.

**Starter code**

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strconv"
	"strings"
	"sync"
)

type Item struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

type Store struct {
	mu    sync.Mutex
	items map[int]Item
	next  int
}

func NewStore() *Store { return &Store{items: map[int]Item{}, next: 1} }

func (s *Store) Handler() http.Handler {
	mux := http.NewServeMux()
	// POST /items: decode {"name": ...} with a body size limit; 400 {"error":"name is required"} when
	//   decoding fails or the trimmed name is empty; otherwise store under the next id and reply 201 + JSON
	// GET /items/{id}: 200 + JSON, or 404 {"error":"not found"} for unknown or non-numeric ids
	// (use r.PathValue("id") and guard the map with the mutex)
	return mux
}

func do(h http.Handler, method, path, body string) string {
	req := httptest.NewRequest(method, path, strings.NewReader(body))
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	return fmt.Sprintf("%d %s", rec.Code, strings.TrimSpace(rec.Body.String()))
}

func main() {
	h := NewStore().Handler()
	fmt.Println(do(h, "POST", "/items", `{"name":"pen"}`))
	fmt.Println(do(h, "POST", "/items", `{"name":"  "}`))
	fmt.Println(do(h, "GET", "/items/1", ""))
	fmt.Println(do(h, "GET", "/items/9", ""))
	fmt.Println(do(h, "GET", "/items/abc", ""))
	fmt.Println(do(h, "DELETE", "/items/1", ""))
}
```

**Expected output**

```text
201 {"id":1,"name":"pen"}
400 {"error":"name is required"}
200 {"id":1,"name":"pen"}
404 {"error":"not found"}
404 {"error":"not found"}
405 Method Not Allowed
```

<details><summary>Show solution and explanation</summary>

<pre><code>package main

import (
	&quot;encoding/json&quot;
	&quot;fmt&quot;
	&quot;net/http&quot;
	&quot;net/http/httptest&quot;
	&quot;strconv&quot;
	&quot;strings&quot;
	&quot;sync&quot;
)

type Item struct {
	ID   int    `json:&quot;id&quot;`
	Name string `json:&quot;name&quot;`
}

type Store struct {
	mu    sync.Mutex
	items map[int]Item
	next  int
}

func NewStore() *Store { return &amp;Store{items: map[int]Item{}, next: 1} }

func (s *Store) Handler() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc(&quot;POST /items&quot;, func(w http.ResponseWriter, r *http.Request) {
		var in struct{ Name string `json:&quot;name&quot;` }
		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1&lt;&lt;10)).Decode(&amp;in); err != nil || strings.TrimSpace(in.Name) == &quot;&quot; {
			http.Error(w, `{&quot;error&quot;:&quot;name is required&quot;}`, http.StatusBadRequest)
			return
		}
		s.mu.Lock()
		it := Item{ID: s.next, Name: in.Name}
		s.items[it.ID] = it
		s.next++
		s.mu.Unlock()
		w.Header().Set(&quot;Content-Type&quot;, &quot;application/json&quot;)
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(it)
	})
	mux.HandleFunc(&quot;GET /items/{id}&quot;, func(w http.ResponseWriter, r *http.Request) {
		id, err := strconv.Atoi(r.PathValue(&quot;id&quot;))
		s.mu.Lock()
		it, ok := s.items[id]
		s.mu.Unlock()
		if err != nil || !ok {
			http.Error(w, `{&quot;error&quot;:&quot;not found&quot;}`, http.StatusNotFound)
			return
		}
		w.Header().Set(&quot;Content-Type&quot;, &quot;application/json&quot;)
		json.NewEncoder(w).Encode(it)
	})
	return mux
}

func do(h http.Handler, method, path, body string) string {
	req := httptest.NewRequest(method, path, strings.NewReader(body))
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	return fmt.Sprintf(&quot;%d %s&quot;, rec.Code, strings.TrimSpace(rec.Body.String()))
}

func main() {
	h := NewStore().Handler()
	fmt.Println(do(h, &quot;POST&quot;, &quot;/items&quot;, `{&quot;name&quot;:&quot;pen&quot;}`))
	fmt.Println(do(h, &quot;POST&quot;, &quot;/items&quot;, `{&quot;name&quot;:&quot;  &quot;}`))
	fmt.Println(do(h, &quot;GET&quot;, &quot;/items/1&quot;, &quot;&quot;))
	fmt.Println(do(h, &quot;GET&quot;, &quot;/items/9&quot;, &quot;&quot;))
	fmt.Println(do(h, &quot;GET&quot;, &quot;/items/abc&quot;, &quot;&quot;))
	fmt.Println(do(h, &quot;DELETE&quot;, &quot;/items/1&quot;, &quot;&quot;))
}
</code></pre>

<p>httptest lets you exercise handlers in-process with no network. The Go 1.22 ServeMux patterns (&quot;POST /items&quot;, &quot;GET /items/{id}&quot;) route by method and path, and unmatched methods get 405 automatically. Verified by running the solution with go run.</p>
</details>
