# 10 · Project — Weather CLI

A complete command-line weather client that pulls live data from a public API.
It exercises everything from Level 2: [interfaces](01-interfaces.md),
[methods](03-methods-receivers.md),
[wrapped errors](04-custom-errors-wrapping.md),
[JSON decoding](05-json-encoding-decoding.md),
[table-driven tests](06-testing-package.md),
[file I/O](07-file-io.md) for caching, and the
[net/http client](08-net-http-client.md).

## What you'll build

A tool called `weather` that:

- Takes a city name and resolves it to coordinates
- Fetches current conditions and a multi-day forecast
- Prints a readable report with temperature, humidity, wind and conditions
- Caches responses to disk so repeated runs are instant and stay under rate limits
- Fails with clear, actionable messages instead of stack traces

**No API key required.** This project uses [Open-Meteo](https://open-meteo.com),
whose forecast and geocoding endpoints are free and keyless for reasonable
non-commercial use. If you later swap in a service like OpenWeatherMap, you
would read its key from an environment variable (`os.Getenv("OWM_API_KEY")`) —
never hard-code a key into source you plan to commit.

## Project layout

```text
weathercli/
    go.mod
    main.go
    weather/
        types.go
        geocode.go
        client.go
        codes.go
        cache.go
        format.go
```

## Setting up the module

```bash
mkdir weathercli && cd weathercli
go mod init weathercli
mkdir weather
```

There are zero external dependencies — the standard library covers HTTP, JSON,
files and flags.

## weather/types.go — the data model

```go
// weather/types.go
package weather

import "time"

// Location is a geocoded place.
type Location struct {
	Name      string  `json:"name"`
	Country   string  `json:"country"`
	Region    string  `json:"admin1"`
	Latitude  float64 `json:"latitude"`
	Longitude float64 `json:"longitude"`
}

// geocodeResponse mirrors the geocoding API's envelope.
type geocodeResponse struct {
	Results []Location `json:"results"`
}

// forecastResponse mirrors only the fields we actually use.
// The API returns far more; unknown keys are ignored by encoding/json.
type forecastResponse struct {
	Timezone string `json:"timezone"`
	Current  struct {
		Time        string  `json:"time"`
		Temperature float64 `json:"temperature_2m"`
		Humidity    int     `json:"relative_humidity_2m"`
		WindSpeed   float64 `json:"wind_speed_10m"`
		WeatherCode int     `json:"weather_code"`
	} `json:"current"`
	Daily struct {
		Time    []string  `json:"time"`
		Codes   []int     `json:"weather_code"`
		MaxTemp []float64 `json:"temperature_2m_max"`
		MinTemp []float64 `json:"temperature_2m_min"`
	} `json:"daily"`
}

// Report is the tidy, presentation-ready result our CLI works with.
type Report struct {
	Location    Location  `json:"location"`
	Timezone    string    `json:"timezone"`
	Temperature float64   `json:"temperature"`
	Humidity    int       `json:"humidity"`
	WindSpeed   float64   `json:"wind_speed"`
	Conditions  string    `json:"conditions"`
	Days        []DayForecast `json:"days"`
	FetchedAt   time.Time `json:"fetched_at"`
}

// DayForecast is one day of the outlook.
type DayForecast struct {
	Date       string  `json:"date"`
	High       float64 `json:"high"`
	Low        float64 `json:"low"`
	Conditions string  `json:"conditions"`
}
```

Separating the wire types (`forecastResponse`) from the domain type (`Report`)
is deliberate: the API's shape is not your program's shape, and decoupling them
means an upstream change touches one file.

## weather/codes.go — WMO condition codes

```go
// weather/codes.go
package weather

// wmoCodes maps WMO weather interpretation codes to plain English.
var wmoCodes = map[int]string{
	0:  "Clear sky",
	1:  "Mainly clear",
	2:  "Partly cloudy",
	3:  "Overcast",
	45: "Fog",
	48: "Depositing rime fog",
	51: "Light drizzle",
	53: "Moderate drizzle",
	55: "Dense drizzle",
	61: "Slight rain",
	63: "Moderate rain",
	65: "Heavy rain",
	71: "Slight snow",
	73: "Moderate snow",
	75: "Heavy snow",
	77: "Snow grains",
	80: "Slight rain showers",
	81: "Moderate rain showers",
	82: "Violent rain showers",
	85: "Slight snow showers",
	86: "Heavy snow showers",
	95: "Thunderstorm",
	96: "Thunderstorm with slight hail",
	99: "Thunderstorm with heavy hail",
}

// Describe converts a WMO code into text, with a safe fallback.
func Describe(code int) string {
	if desc, ok := wmoCodes[code]; ok { // comma-ok: never assume the key exists
		return desc
	}
	return "Unknown conditions"
}
```

## weather/geocode.go — city name to coordinates

```go
// weather/geocode.go
package weather

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"net/url"
)

// ErrCityNotFound is a sentinel so callers can react without parsing strings.
var ErrCityNotFound = errors.New("city not found")

const geocodeURL = "https://geocoding-api.open-meteo.com/v1/search"

// Geocode resolves a city name to a Location.
func (c *Client) Geocode(ctx context.Context, city string) (Location, error) {
	q := url.Values{}
	q.Set("name", city)
	q.Set("count", "1")
	q.Set("language", "en")
	q.Set("format", "json")

	endpoint := geocodeURL + "?" + q.Encode()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
	if err != nil {
		return Location{}, fmt.Errorf("building geocode request: %w", err)
	}

	resp, err := c.http.Do(req)
	if err != nil {
		return Location{}, fmt.Errorf("geocoding %q: %w", city, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return Location{}, fmt.Errorf("geocoding %q: unexpected status %s", city, resp.Status)
	}

	var gr geocodeResponse
	if err := json.NewDecoder(resp.Body).Decode(&gr); err != nil {
		return Location{}, fmt.Errorf("decoding geocode response: %w", err)
	}

	if len(gr.Results) == 0 {
		// Wrap the sentinel so errors.Is still matches, but keep the context.
		return Location{}, fmt.Errorf("%q: %w", city, ErrCityNotFound)
	}
	return gr.Results[0], nil
}
```

## weather/client.go — the API client

```go
// weather/client.go
package weather

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"strconv"
	"time"
)

const forecastURL = "https://api.open-meteo.com/v1/forecast"

// Client talks to the weather service. One instance is reused for all calls
// so the underlying connection pool is shared.
type Client struct {
	http *http.Client
}

// NewClient returns a Client with a sane timeout.
func NewClient(timeout time.Duration) *Client {
	return &Client{
		http: &http.Client{Timeout: timeout}, // never use http.DefaultClient
	}
}

// Fetch retrieves current conditions plus a `days`-day outlook.
func (c *Client) Fetch(ctx context.Context, loc Location, days int) (*Report, error) {
	q := url.Values{}
	q.Set("latitude", strconv.FormatFloat(loc.Latitude, 'f', 4, 64))
	q.Set("longitude", strconv.FormatFloat(loc.Longitude, 'f', 4, 64))
	q.Set("current", "temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code")
	q.Set("daily", "weather_code,temperature_2m_max,temperature_2m_min")
	q.Set("timezone", "auto")
	q.Set("forecast_days", strconv.Itoa(days))

	endpoint := forecastURL + "?" + q.Encode()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
	if err != nil {
		return nil, fmt.Errorf("building forecast request: %w", err)
	}
	req.Header.Set("User-Agent", "weathercli/1.0 (go-mastery-path)")

	resp, err := c.http.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fetching forecast: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("forecast API returned %s", resp.Status)
	}

	var fr forecastResponse
	if err := json.NewDecoder(resp.Body).Decode(&fr); err != nil {
		return nil, fmt.Errorf("decoding forecast: %w", err)
	}

	return buildReport(loc, fr), nil
}

// buildReport converts the wire format into our domain type.
func buildReport(loc Location, fr forecastResponse) *Report {
	r := &Report{
		Location:    loc,
		Timezone:    fr.Timezone,
		Temperature: fr.Current.Temperature,
		Humidity:    fr.Current.Humidity,
		WindSpeed:   fr.Current.WindSpeed,
		Conditions:  Describe(fr.Current.WeatherCode),
		FetchedAt:   time.Now(),
	}

	// The daily arrays are parallel; guard against ragged responses.
	for i := range fr.Daily.Time {
		if i >= len(fr.Daily.MaxTemp) || i >= len(fr.Daily.MinTemp) || i >= len(fr.Daily.Codes) {
			break
		}
		r.Days = append(r.Days, DayForecast{
			Date:       fr.Daily.Time[i],
			High:       fr.Daily.MaxTemp[i],
			Low:        fr.Daily.MinTemp[i],
			Conditions: Describe(fr.Daily.Codes[i]),
		})
	}
	return r
}
```

Indexing four parallel arrays without a bounds check is a classic way to earn
an `index out of range` panic from a slightly-off API response. The `break`
guard costs one line.

## weather/cache.go — a disk cache with TTL

```go
// weather/cache.go
package weather

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
	"strings"
	"time"
)

// Cache stores Reports as JSON files under a directory.
type Cache struct {
	dir string
	ttl time.Duration
}

// NewCache puts its files in the user's cache directory, falling back to
// the temp directory if that is unavailable.
func NewCache(ttl time.Duration) *Cache {
	base, err := os.UserCacheDir()
	if err != nil {
		base = os.TempDir()
	}
	return &Cache{dir: filepath.Join(base, "weathercli"), ttl: ttl}
}

// key turns a city name into a safe filename.
func (c *Cache) key(city string) string {
	safe := strings.Map(func(r rune) rune {
		switch {
		case r >= 'a' && r <= 'z', r >= '0' && r <= '9':
			return r
		case r >= 'A' && r <= 'Z':
			return r + 32 // lowercase
		default:
			return '_'
		}
	}, city)
	return filepath.Join(c.dir, safe+".json")
}

// Get returns a cached Report if one exists and is still fresh.
// A miss is not an error: (nil, false) simply means "go fetch it".
func (c *Cache) Get(city string) (*Report, bool) {
	data, err := os.ReadFile(c.key(city))
	if err != nil {
		return nil, false
	}

	var r Report
	if err := json.Unmarshal(data, &r); err != nil {
		return nil, false // corrupt cache entry: ignore and refetch
	}
	if time.Since(r.FetchedAt) > c.ttl {
		return nil, false // stale
	}
	return &r, true
}

// Put writes a Report to the cache atomically.
func (c *Cache) Put(city string, r *Report) error {
	if err := os.MkdirAll(c.dir, 0o755); err != nil {
		return fmt.Errorf("creating cache dir: %w", err)
	}

	data, err := json.MarshalIndent(r, "", "  ")
	if err != nil {
		return fmt.Errorf("encoding cache entry: %w", err)
	}

	// Write to a temp file then rename, so a crash never leaves half a file.
	tmp, err := os.CreateTemp(c.dir, ".tmp-*")
	if err != nil {
		return fmt.Errorf("creating temp cache file: %w", err)
	}
	tmpName := tmp.Name()
	defer os.Remove(tmpName) // no-op after a successful rename

	if _, err := tmp.Write(data); err != nil {
		tmp.Close()
		return fmt.Errorf("writing cache file: %w", err)
	}
	if err := tmp.Close(); err != nil {
		return fmt.Errorf("closing cache file: %w", err)
	}
	return os.Rename(tmpName, c.key(city))
}
```

## weather/format.go — human-readable output

```go
// weather/format.go
package weather

import (
	"fmt"
	"strings"
)

// String satisfies fmt.Stringer, so a *Report can be passed to fmt.Println.
func (r *Report) String() string {
	var b strings.Builder

	place := r.Location.Name
	if r.Location.Region != "" {
		place += ", " + r.Location.Region
	}
	if r.Location.Country != "" {
		place += ", " + r.Location.Country
	}

	fmt.Fprintf(&b, "\n  %s\n", place)
	fmt.Fprintf(&b, "  %s\n\n", strings.Repeat("─", len(place)))
	fmt.Fprintf(&b, "  Now:       %.1f°C, %s\n", r.Temperature, r.Conditions)
	fmt.Fprintf(&b, "  Humidity:  %d%%\n", r.Humidity)
	fmt.Fprintf(&b, "  Wind:      %.1f km/h\n", r.WindSpeed)

	if len(r.Days) > 0 {
		fmt.Fprintf(&b, "\n  Forecast\n")
		for _, d := range r.Days {
			fmt.Fprintf(&b, "  %-12s %5.1f° / %5.1f°  %s\n",
				d.Date, d.High, d.Low, d.Conditions)
		}
	}
	fmt.Fprintf(&b, "\n  Timezone: %s · fetched %s\n",
		r.Timezone, r.FetchedAt.Format("15:04:05"))

	return b.String()
}
```

`strings.Builder` is the efficient way to assemble a string piece by piece —
repeated `s += ...` allocates a new string every time.

## main.go — flags, wiring and exit codes

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"os"
	"time"

	"weathercli/weather"
)

func main() {
	// Exit codes belong in one place, so defer-based cleanup still runs.
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "weather:", err)
		os.Exit(1)
	}
}

func run() error {
	var (
		city    = flag.String("city", "", "city name to look up (required)")
		days    = flag.Int("days", 3, "number of forecast days (1-7)")
		asJSON  = flag.Bool("json", false, "print raw JSON instead of a report")
		noCache = flag.Bool("no-cache", false, "bypass the local cache")
		ttl     = flag.Duration("ttl", 15*time.Minute, "cache lifetime")
	)
	flag.Parse()

	if *city == "" {
		flag.Usage()
		return errors.New("the -city flag is required")
	}
	if *days < 1 || *days > 7 {
		return fmt.Errorf("-days must be between 1 and 7, got %d", *days)
	}

	cache := weather.NewCache(*ttl)

	if !*noCache {
		if report, ok := cache.Get(*city); ok {
			return output(report, *asJSON)
		}
	}

	// One overall deadline for geocoding + forecast together.
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	client := weather.NewClient(10 * time.Second)

	loc, err := client.Geocode(ctx, *city)
	if err != nil {
		if errors.Is(err, weather.ErrCityNotFound) {
			return fmt.Errorf("no city named %q — check the spelling", *city)
		}
		return err
	}

	report, err := client.Fetch(ctx, loc, *days)
	if err != nil {
		if errors.Is(err, context.DeadlineExceeded) {
			return errors.New("the weather service took too long to respond")
		}
		return err
	}

	// A cache write failure should not fail the command.
	if err := cache.Put(*city, report); err != nil {
		fmt.Fprintln(os.Stderr, "warning: could not write cache:", err)
	}

	return output(report, *asJSON)
}

func output(r *weather.Report, asJSON bool) error {
	if asJSON {
		enc := json.NewEncoder(os.Stdout)
		enc.SetIndent("", "  ")
		return enc.Encode(r)
	}
	fmt.Print(r) // uses the Stringer from format.go
	return nil
}
```

## weather/codes_test.go — a table-driven test

```go
// weather/codes_test.go
package weather

import "testing"

func TestDescribe(t *testing.T) {
	tests := []struct {
		name string
		code int
		want string
	}{
		{"clear", 0, "Clear sky"},
		{"overcast", 3, "Overcast"},
		{"heavy rain", 65, "Heavy rain"},
		{"thunderstorm", 95, "Thunderstorm"},
		{"unmapped code", 12345, "Unknown conditions"},
		{"negative code", -1, "Unknown conditions"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Describe(tt.code); got != tt.want {
				t.Errorf("Describe(%d) = %q; want %q", tt.code, got, tt.want)
			}
		})
	}
}

func TestCacheRoundTrip(t *testing.T) {
	c := &Cache{dir: t.TempDir(), ttl: time.Minute} // t.TempDir auto-cleans

	want := &Report{
		Location:    Location{Name: "Testville"},
		Temperature: 18.5,
		Conditions:  "Overcast",
		FetchedAt:   time.Now(),
	}
	if err := c.Put("Testville", want); err != nil {
		t.Fatalf("Put() error = %v", err)
	}

	got, ok := c.Get("Testville")
	if !ok {
		t.Fatal("Get() reported a miss right after Put()")
	}
	if got.Temperature != want.Temperature {
		t.Errorf("Temperature = %v; want %v", got.Temperature, want.Temperature)
	}
}
```

(Add `"time"` to that file's imports.)

## Running it

```bash
go build -o weather .

./weather -city "Tokyo"
```

```text
  Tokyo, Tokyo, Japan
  ───────────────────

  Now:       28.4°C, Partly cloudy
  Humidity:  71%
  Wind:      9.2 km/h

  Forecast
  2026-07-31    31.0° /  24.6°  Partly cloudy
  2026-08-01    32.4° /  25.1°  Slight rain showers
  2026-08-02    29.8° /  24.0°  Moderate rain

  Timezone: Asia/Tokyo · fetched 14:22:09
```

Other invocations:

```bash
./weather -city "Reykjavik" -days 7      # a week ahead
./weather -city "Nairobi" -json          # machine-readable output
./weather -city "Tokyo" -no-cache        # force a fresh fetch
./weather -city "Xyzzyville"             # weather: no city named "Xyzzyville" — check the spelling
./weather                                # usage, then exit code 1
```

Run the second `./weather -city "Tokyo"` and it returns instantly from the
cache in `~/Library/Caches/weathercli` (macOS) or `~/.cache/weathercli`
(Linux). Delete that directory to reset.

Run the tests with:

```bash
go test ./...
go test -v -run TestDescribe ./weather
```

## What to notice in this design

- **`run() error` instead of `os.Exit` everywhere.** `os.Exit` skips deferred
  calls, so all error paths funnel back to `main`, which exits once.
- **Sentinels for expected conditions.** `ErrCityNotFound` lets `main` turn one
  specific failure into friendly advice while everything else prints raw.
- **Wire types separate from domain types.** `forecastResponse` is unexported
  and never leaves the package.
- **Degraded, not failed.** A cache write error prints a warning to stderr; the
  weather report still prints to stdout.
- **stdout vs stderr.** Data goes to stdout so `./weather -city X -json | jq`
  works; diagnostics go to stderr.

## Stretch goals

- Add `-units=imperial` and convert to °F and mph (use a method on `Report`).
- Add a `Provider` interface with a `Fetch` method, then write a
  `mockProvider` for tests that returns a fixed `Report` with no network at
  all — see [Module 1](01-interfaces.md) and [Module 6](06-testing-package.md).
- Accept several cities (`./weather -city Tokyo -city Oslo`) and fetch them
  concurrently with goroutines and a `sync.WaitGroup`
  ([Module 2](02-goroutines-channels.md)).
- Replace `flag` with a subcommand-based CLI in
  [Level 3, Module 8](../level-3/08-building-clis.md).
- Add a `-history` flag that appends every lookup to a log file with
  `os.O_APPEND` ([Module 7](07-file-io.md)).
- Use `httptest.NewServer` to test `Fetch` against a canned JSON response —
  covered in [Level 3, Module 6](../level-3/06-testing-advanced.md).

Finishing this project means you can consume any JSON HTTP API in Go, handle
its failures properly, and ship the result as a real binary. You're ready for
**Level 3 · Advanced**.
