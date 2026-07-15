# Writing Production-Level Go: A Practical Guide

## 1. Mindset Shift

| Quick Code | Production Code |
|---|---|
| Ignored errors (`_ = err`) | Every error checked and handled deliberately |
| `fmt.Println` debugging | Structured logging |
| Hardcoded config | Env vars / config files |
| No tests | Table-driven tests, benchmarks |
| Global mutable state | Explicit dependency passing |
| "It builds" | "It builds, is tested, race-free, and observable" |

---

## 2. Project Structure

```
my-service/
├── cmd/
│   └── my-service/
│       └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   ├── repository/
│   └── config/
├── pkg/              # code safe for external import, if any
├── api/              # OpenAPI/proto specs
├── migrations/
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── README.md
```

**Tips:**
- `internal/` is enforced by the Go compiler — anything inside cannot be imported from outside the module. Use it for everything not meant to be a public API.
- Keep `main.go` thin — just wiring (config load, dependency construction, server start). Business logic lives in `internal/`.
- One package = one clear purpose; avoid a catch-all `utils` package if you can help it — it becomes a dumping ground.

---

## 3. Code Quality Checklist

- [ ] Run **`gofmt`**/**`goimports`** — Go's formatting is non-negotiable and automatic, so there's no style debate to have
- [ ] Run **`go vet`** to catch suspicious constructs
- [ ] Use **`golangci-lint`** (aggregates staticcheck, errcheck, gosimple, etc.) in CI
- [ ] Keep functions short and single-purpose; if a function needs a scroll to read, it likely needs splitting
- [ ] Prefer clear, exported names with doc comments over cleverness — Go values readability highly

```go
// Bad
func proc(d []float64, t float64) map[string]float64 { ... }

// Good
// CalculateStatistics returns summary statistics for the given data points
// that exceed the threshold.
func CalculateStatistics(data []float64, threshold float64) (map[string]float64, error) { ... }
```

---

## 4. Error Handling

Go's explicit error handling is central to writing correct production code — never ignore an error.

```go
// Bad
result, _ := doSomething()

// Bad — loses context
if err != nil {
    return err
}

// Good — wrap with context using fmt.Errorf + %w
func processOrder(orderID string) error {
    order, err := repo.FindOrder(orderID)
    if err != nil {
        return fmt.Errorf("processOrder: fetching order %s: %w", orderID, err)
    }
    if err := paymentGateway.Charge(order); err != nil {
        return fmt.Errorf("processOrder: charging order %s: %w", orderID, err)
    }
    return nil
}
```

**Best practices:**
- [ ] Check every error; never `_ = err` unless you have a documented reason
- [ ] Wrap errors with `fmt.Errorf("...: %w", err)` to preserve the chain, and unwrap with `errors.Is`/`errors.As`
- [ ] Define sentinel errors (`var ErrNotFound = errors.New("not found")`) or custom error types for cases callers need to branch on
- [ ] Avoid `panic` for expected/recoverable errors — reserve `panic` for truly unrecoverable programmer errors, and only `recover` at well-defined boundaries (e.g., an HTTP middleware)
- [ ] Use `errcheck` (part of golangci-lint) in CI to catch unchecked errors automatically

```go
var ErrOrderNotFound = errors.New("order not found")

func FindOrder(id string) (*Order, error) {
    order, ok := store[id]
    if !ok {
        return nil, fmt.Errorf("FindOrder %s: %w", id, ErrOrderNotFound)
    }
    return order, nil
}

// Caller:
if errors.Is(err, ErrOrderNotFound) {
    // handle specifically
}
```

---

## 5. Logging

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

logger.Info("processing order", "order_id", orderID, "amount", order.Amount)
logger.Error("payment failed", "order_id", orderID, "error", err)
```

**Checklist:**
- [ ] Use **`log/slog`** (stdlib, structured logging since Go 1.21) or a mature library (`zap`, `zerolog`) for high-throughput services
- [ ] Structured (key-value or JSON) logs, not free-form strings — makes them queryable in log aggregators
- [ ] Never log secrets/tokens/PII
- [ ] Attach request/trace IDs via context for correlating logs across a request's lifecycle
- [ ] Log at service boundaries (incoming requests, outgoing calls, errors) — not every line of business logic

---

## 6. Configuration Management

```go
type Config struct {
    Port          int    `env:"PORT" envDefault:"8080"`
    DatabaseURL   string `env:"DATABASE_URL,required"`
    PaymentAPIKey string `env:"PAYMENT_API_KEY,required"`
}

func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("loading config: %w", err)
    }
    return cfg, nil
}
```

- [ ] Externalize config via environment variables (12-factor app principle) — libraries like `caarlos0/env` or `spf13/viper` help
- [ ] Validate config at startup; fail fast with a clear error rather than panicking deep in a request handler
- [ ] Never commit secrets; use a secrets manager or `.env` (gitignored) for local dev
- [ ] Separate config per environment (dev/staging/prod)

---

## 7. Testing

Go's testing is built into the toolchain — use it fully.

```go
func TestCalculateDiscount(t *testing.T) {
    tests := []struct {
        name     string
        price    float64
        rate     float64
        want     float64
        wantErr  bool
    }{
        {"no discount", 100, 0.0, 100, false},
        {"full discount", 100, 1.0, 0, false},
        {"half discount", 50, 0.5, 25, false},
        {"negative price", -10, 0.1, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := CalculateDiscount(tt.price, tt.rate)
            if (err != nil) != tt.wantErr {
                t.Fatalf("unexpected error state: %v", err)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

**Checklist:**
- [ ] Table-driven tests are the Go idiom — use them for anything with multiple input/output cases
- [ ] Use the standard `testing` package; add `testify/assert` or `testify/require` for more expressive assertions if desired
- [ ] Run tests with the **race detector** in CI: `go test -race ./...` — catches data races that are otherwise invisible
- [ ] Use `httptest` for testing HTTP handlers without a real server
- [ ] Write benchmarks (`func BenchmarkX(b *testing.B)`) for performance-critical code paths
- [ ] Use build tags or a separate `_test` suffix to separate unit vs. integration tests; use **Testcontainers-go** for real dependency integration tests
- [ ] Check coverage with `go test -cover ./...`, aim for meaningful coverage on business logic

---

## 8. Documentation

```go
// CalculateDiscount returns the price after applying discountRate.
// discountRate must be between 0 and 1 inclusive; price must be non-negative.
// It returns an error if either constraint is violated.
func CalculateDiscount(price, discountRate float64) (float64, error) {
    if price < 0 {
        return 0, errors.New("price must be non-negative")
    }
    if discountRate < 0 || discountRate > 1 {
        return 0, errors.New("discountRate must be between 0 and 1")
    }
    return price * (1 - discountRate), nil
}
```

- [ ] Every exported identifier gets a doc comment starting with its name (Go convention, enforced by `golint`/`staticcheck`)
- [ ] `godoc`/`pkg.go.dev` renders these automatically — no separate doc-generation step needed
- [ ] Clear `README.md`: what it does, how to build/run/test, architecture diagram for non-trivial services

---

## 9. Dependency Management

- [ ] Go Modules (`go.mod`/`go.sum`) is standard — commit both
- [ ] Run `go mod tidy` regularly to keep dependencies clean
- [ ] Pin versions implicitly via `go.sum` (cryptographic hash pinning — very solid by default)
- [ ] Use `govulncheck` to scan for known vulnerabilities in dependencies
- [ ] Keep the dependency tree small — Go's standard library is unusually capable; reach for it before a third-party package

---

## 10. Security Basics

- [ ] Use `database/sql` parameterized queries — never string-concatenate SQL
- [ ] Validate/sanitize all external input
- [ ] Use `context.Context` with timeouts on all outbound calls (HTTP, DB) — an unbounded call is a production incident waiting to happen
- [ ] Run `govulncheck` and `gosec` in CI
- [ ] Never trust `unsafe` package usage without a very clear reason and review

```go
// Bad
query := "SELECT * FROM users WHERE id = " + userID

// Good
row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", userID)
```

---

## 11. Concurrency (Core Go Strength — and Risk)

```go
func fetchAll(ctx context.Context, ids []string) ([]Result, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Result, len(ids))

    for i, id := range ids {
        i, id := i, id // capture loop vars (pre-Go 1.22)
        g.Go(func() error {
            r, err := fetch(ctx, id)
            if err != nil {
                return err
            }
            results[i] = r
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

- [ ] Always pass and respect `context.Context` for cancellation/timeouts in anything that does I/O
- [ ] Use `sync.WaitGroup` / `errgroup.Group` for coordinating goroutines, not raw channels for simple fan-out/fan-in
- [ ] Protect shared mutable state with `sync.Mutex`/`sync.RWMutex`, or better, avoid shared mutable state via channels ("share memory by communicating")
- [ ] Always run `go test -race` — goroutine bugs are notoriously hard to catch by inspection alone
- [ ] Be deliberate about goroutine lifetimes — a goroutine with no way to be stopped is a leak

---

## 12. Performance & Efficiency

- [ ] Profile with `pprof` before optimizing — Go has excellent built-in profiling (`net/http/pprof` for live services)
- [ ] Preallocate slices with known capacity (`make([]T, 0, n)`) to avoid repeated reallocations
- [ ] Avoid unnecessary allocations in hot paths — use benchmarks (`go test -bench`) to catch regressions
- [ ] Reuse buffers with `sync.Pool` for high-throughput allocation-heavy code
- [ ] Understand escape analysis basics (`go build -gcflags="-m"`) if working on latency-critical code

---

## 13. Version Control & CI/CD

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go build ./...
      - run: go vet ./...
      - run: go test -race -cover ./...
      - uses: golangci/golangci-lint-action@v6
```

- [ ] Small, focused commits; PRs even solo
- [ ] `.gitignore` covers built binaries, `.env`
- [ ] CI runs build, vet, race-enabled tests, and lint on every push
- [ ] Fail the build on lint/vet issues, not just test failures

---

## 14. Design Principles Worth Internalizing

- **Accept interfaces, return structs** — keep function signatures flexible for callers, concrete for implementers
- **Composition over inheritance** — Go has no inheritance; embed structs/interfaces instead
- **Small interfaces** — Go idiom favors single-method interfaces (`io.Reader`, `io.Writer`) over large ones
- **Explicit over implicit** — no hidden magic; dependencies passed explicitly (constructor injection), no global singletons if avoidable
- **Errors are values** — treat them as first-class return values to be handled, not exceptional control flow

```go
type Notifier interface {
    Notify(ctx context.Context, msg string) error
}

// OrderService depends on the interface, not a concrete implementation —
// easy to swap/mock in tests.
type OrderService struct {
    notifier Notifier
}
```

---

## 15. Pre-Deployment Checklist

- [ ] `go build ./...`, `go vet ./...`, and `golangci-lint run` all pass
- [ ] Tests pass with `-race` enabled
- [ ] No secrets in code, config, or logs
- [ ] Every I/O call has a context with a timeout
- [ ] Structured logging with correlation IDs in place
- [ ] Config externalized and validated at startup
- [ ] `govulncheck` shows no known vulnerabilities
- [ ] Health/readiness endpoints exist for orchestration (k8s liveness/readiness probes)
- [ ] Metrics exported (Prometheus via `client_golang` is the common choice)
- [ ] Graceful shutdown implemented (drain in-flight requests on SIGTERM)

```go
srv := &http.Server{Addr: ":8080", Handler: router}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        logger.Error("server error", "error", err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

---

## 16. Tools Summary

| Purpose | Tool |
|---|---|
| Formatting | `gofmt` / `goimports` (built-in) |
| Linting | `golangci-lint` |
| Vulnerability scan | `govulncheck`, `gosec` |
| Testing | stdlib `testing`, `testify` |
| Integration testing | Testcontainers-go |
| Profiling | `pprof` |
| Logging | `log/slog`, `zap`, `zerolog` |
| Config | `caarlos0/env`, `spf13/viper` |
| HTTP framework (optional) | stdlib `net/http` + `chi`, or `gin`/`echo` |
| Concurrency helpers | `golang.org/x/sync/errgroup` |
| Metrics | `prometheus/client_golang` |
| CI/CD | GitHub Actions / GitLab CI |

---

## 17. How to Build the Habit

1. **Never ignore an error** — make it a hard rule, backed by `errcheck` in CI.
2. **Write table-driven tests by default** — it becomes second nature quickly.
3. **Always thread `context.Context`** through I/O-touching functions from day one, even if you don't need cancellation yet — retrofitting it later touches every signature.
4. **Run with `-race` locally** whenever you touch goroutines.
5. **Read the standard library source** — it's some of the best example Go code available, and it's right there in your Go installation.

---

### Quick Reference: The 80/20 That Matters Most

1. Check and wrap every error (`%w`), never silently discard one
2. `context.Context` with timeouts on all I/O
3. Table-driven tests + `go test -race`
4. Structured logging (`slog`) instead of `fmt.Println`
5. `golangci-lint` wired into CI from day one
