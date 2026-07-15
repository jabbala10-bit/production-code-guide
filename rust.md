# Writing Production-Level Rust: A Practical Guide

## 1. Mindset Shift

| Quick Code | Production Code |
|---|---|
| `.unwrap()` everywhere | Deliberate `Result`/`Option` handling |
| `clone()` to silence the borrow checker | Understood ownership, minimal cloning |
| No tests | Unit, integration, and property tests |
| `println!` debugging | Structured tracing/logging |
| Panics on bad input | Graceful error propagation |
| "It compiles" (already a high bar in Rust) | "It compiles, is tested, and has no needless `unwrap`s in the hot path" |

Rust's compiler already eliminates a huge class of production bugs (data races, null derefs, use-after-free). Production discipline in Rust is mostly about **error handling, API design, and not fighting the type system with shortcuts**.

---

## 2. Project Structure

```
my-service/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── main.rs           # thin entrypoint
│   ├── lib.rs             # crate root if it's also a library
│   ├── config.rs
│   ├── error.rs
│   ├── handlers/
│   ├── services/
│   ├── models/
│   └── repository/
├── tests/                 # integration tests
│   └── api_test.rs
├── benches/                # benchmarks (criterion)
├── .env.example
├── .gitignore
└── README.md
```

**Tips:**
- For multi-binary or multi-crate projects, use a **Cargo workspace** (`[workspace]` in root `Cargo.toml`) to share dependencies and build once.
- Keep `main.rs` minimal — argument parsing, config load, and calling into `lib.rs`/modules. Business logic belongs in testable library code, not the binary crate.
- `tests/` files are compiled as separate crates and only see your public API — good for true black-box integration tests.

---

## 3. Code Quality Checklist

- [ ] Run **`rustfmt`** — like Go, Rust formatting is standardized and automatic, no style debates
- [ ] Run **`clippy`** (the Rust linter) — genuinely excellent at catching non-idiomatic or inefficient patterns
- [ ] Treat `clippy::pedantic` warnings as worth reading even if you don't fix every one
- [ ] Prefer `&str` over `String` in function parameters unless ownership is actually needed
- [ ] Prefer borrowing over cloning; if you're calling `.clone()` to make the compiler happy, pause and check if a reference or lifetime annotation solves it instead

```bash
cargo fmt --check
cargo clippy -- -D warnings
```

---

## 4. Error Handling

This is where "production Rust" is most distinct from "learning Rust."

```rust
// Bad — panics propagate as crashes
fn get_user(id: &str) -> User {
    db.query(id).unwrap()
}

// Good — explicit, propagated Result
use thiserror::Error;

#[derive(Error, Debug)]
pub enum UserError {
    #[error("user not found: {0}")]
    NotFound(String),
    #[error("database error")]
    Database(#[from] sqlx::Error),
}

fn get_user(id: &str) -> Result<User, UserError> {
    db.query(id)?.ok_or_else(|| UserError::NotFound(id.to_string()))
}
```

**Best practices:**
- [ ] Use `Result<T, E>` for anything that can fail — never `unwrap()`/`expect()` in library or production request-handling code paths
- [ ] `unwrap()`/`expect()` are acceptable in tests, quick prototypes, or cases that are provably infallible (and `expect("reason")` should explain why)
- [ ] Use **`thiserror`** for defining library error types (derives `std::error::Error` cleanly)
- [ ] Use **`anyhow`** at the application/binary boundary where you just need to propagate errors with context, not match on variants
- [ ] Use the `?` operator to propagate errors instead of manual `match`/`if let Err`
- [ ] Add context to errors as they propagate (`anyhow::Context::context("failed to load config")`)

```rust
use anyhow::{Context, Result};

fn load_config() -> Result<Config> {
    let raw = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;
    toml::from_str(&raw).context("failed to parse config.toml")
}
```

---

## 5. Logging & Observability

```rust
use tracing::{info, error, instrument};

#[instrument(skip(db))]
async fn process_order(db: &Database, order_id: &str) -> Result<()> {
    info!(order_id, "processing order");
    match charge_payment(db, order_id).await {
        Ok(_) => Ok(()),
        Err(e) => {
            error!(order_id, error = %e, "payment failed");
            Err(e)
        }
    }
}
```

**Checklist:**
- [ ] Use **`tracing`** (the modern standard) over `log` for anything async or spanning multiple operations — it supports structured, contextual, span-based logging natively
- [ ] `#[instrument]` macro auto-adds function args and timing to spans — very low-effort observability
- [ ] Structured fields (`order_id, error = %e`) over string interpolation
- [ ] Never log secrets/tokens
- [ ] Integrate with `tracing-subscriber` + OpenTelemetry exporters for distributed tracing in real services

---

## 6. Configuration Management

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Config {
    port: u16,
    database_url: String,
    #[serde(default)]
    debug: bool,
}

fn load_config() -> anyhow::Result<Config> {
    envy::from_env::<Config>().context("failed to load config from environment")
}
```

- [ ] Externalize config via environment variables (`envy`, `figment`, or `config` crate) or config files (with `serde` + `toml`/`yaml`)
- [ ] Validate config at startup; fail fast with a readable error, not a panic three layers deep
- [ ] Never commit secrets; `.env` for local dev only, gitignored
- [ ] Separate configuration per environment

---

## 7. Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn applies_percentage_discount() {
        assert_eq!(calculate_discount(100.0, 0.1).unwrap(), 90.0);
    }

    #[test]
    fn rejects_negative_price() {
        assert!(calculate_discount(-10.0, 0.1).is_err());
    }

    #[test]
    #[should_panic(expected = "discount out of range")]
    fn panics_on_invalid_input_where_appropriate() {
        // only for genuinely unrecoverable invariant violations
    }
}
```

**Checklist:**
- [ ] Unit tests live in a `#[cfg(test)] mod tests` block right next to the code they test — Rust convention, zero extra setup
- [ ] Integration tests go in `tests/` and exercise only the public API
- [ ] Use **`proptest`** or **`quickcheck`** for property-based testing on pure logic (great fit for parsers, math, serialization code)
- [ ] Use **`criterion`** for benchmarks (`benches/` directory) — much more statistically rigorous than manual timing
- [ ] For async code, use `#[tokio::test]` and consider `wiremock`/`mockito` for mocking HTTP dependencies
- [ ] Check coverage with `cargo tarpaulin` or `cargo llvm-cov`
- [ ] Run `cargo test` in CI on every push, plus `cargo clippy -- -D warnings` and `cargo fmt --check`

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn discount_never_produces_negative_price(price in 0.0..1_000_000.0, rate in 0.0..1.0) {
        let result = calculate_discount(price, rate).unwrap();
        prop_assert!(result >= 0.0);
    }
}
```

---

## 8. Documentation

```rust
/// Calculates the discounted price.
///
/// # Arguments
/// * `price` - original price, must be non-negative
/// * `discount_rate` - fraction between 0.0 and 1.0 inclusive
///
/// # Errors
/// Returns `DiscountError::InvalidPrice` or `DiscountError::InvalidRate`
/// if constraints are violated.
///
/// # Examples
/// ```
/// let result = calculate_discount(100.0, 0.1)?;
/// assert_eq!(result, 90.0);
/// ```
pub fn calculate_discount(price: f64, discount_rate: f64) -> Result<f64, DiscountError> {
    // ...
}
```

- [ ] `///` doc comments on all public items — `cargo doc --open` generates browsable HTML automatically
- [ ] Doc examples (` ```rust ... ``` `) are compiled and run by `cargo test` — free extra tests that also serve as usage documentation
- [ ] `README.md` with build/run/test instructions and crate purpose
- [ ] Run `cargo doc --no-deps` in CI to catch broken doc links/examples

---

## 9. Dependency Management

- [ ] `Cargo.toml`/`Cargo.lock` — commit the lockfile for binaries (services); libraries typically don't commit it
- [ ] Run `cargo update` deliberately, review the diff, don't blindly auto-update in CI
- [ ] Use **`cargo audit`** to check for known vulnerabilities (RUSTSEC advisories)
- [ ] Use **`cargo deny`** to enforce license and dependency policy in larger orgs
- [ ] Keep the dependency tree lean — check `cargo tree` occasionally to see what's being pulled in transitively

---

## 10. Security Basics

- [ ] Use `sqlx`/`diesel` with parameterized queries — never format raw SQL strings
- [ ] Validate all external input at the boundary (consider the `validator` crate)
- [ ] Avoid `unsafe` unless absolutely necessary; when used, isolate it, document the invariants it relies on, and test it heavily
- [ ] `cargo audit` in CI to catch known-vulnerable dependencies
- [ ] Use `zeroize` crate for sensitive data (keys, passwords) that needs to be scrubbed from memory after use

```rust
// Bad
let query = format!("SELECT * FROM users WHERE id = {}", user_id);

// Good (sqlx)
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", user_id)
    .fetch_one(&pool)
    .await?;
```

---

## 11. Concurrency & Async

- [ ] Use **`tokio`** as the async runtime for most production services (the de facto standard); avoid mixing multiple async runtimes in one binary
- [ ] Understand `Send`/`Sync` — the compiler enforces safe concurrency at compile time, but you still need to design around it (e.g., wrapping shared state in `Arc<Mutex<T>>` or, preferably, using channels)
- [ ] Prefer message-passing (`tokio::sync::mpsc`) over shared mutable state where it simplifies reasoning
- [ ] Use `Arc<RwLock<T>>` when reads vastly outnumber writes; plain `Mutex` otherwise
- [ ] Set timeouts on all async I/O (`tokio::time::timeout`) — an unbounded await is a production incident

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState {
    cache: Arc<RwLock<HashMap<String, Product>>>,
}
```

---

## 12. Performance & Efficiency

- [ ] Profile before optimizing (`cargo flamegraph`, `perf`, `criterion` for microbenchmarks)
- [ ] Avoid unnecessary `.clone()`/`.to_string()` — these are often the biggest easy win in a Rust codebase's performance
- [ ] Prefer iterators over manual index loops — they're typically as fast (often faster, due to bounds-check elision) and more expressive
- [ ] Use `#[inline]` sparingly and only where profiling justifies it — the compiler is usually better at this decision than you are
- [ ] Build with `--release` for any performance testing or deployment (debug builds are dramatically slower)
- [ ] Consider `cargo-bloat` to inspect what's contributing to binary size for embedded/CLI distribution scenarios

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
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test --all-features
      - run: cargo audit
```

- [ ] `.gitignore` covers `/target`, `.env`
- [ ] CI runs fmt check, clippy (deny warnings), tests, and `cargo audit` on every push
- [ ] Cache `~/.cargo` and `target/` in CI to keep build times reasonable (Rust compile times are the classic pain point — caching mitigates a lot of it)

---

## 14. Design Principles Worth Internalizing

- **Make illegal states unrepresentable** — use the type system (enums, newtypes) so invalid states can't even be constructed, rather than validating at runtime
- **Newtype pattern** for domain concepts instead of raw primitives (`struct UserId(String)` instead of passing bare `String` everywhere)
- **Prefer composition via traits** over inheritance-like hierarchies
- **Zero-cost abstractions** — don't be afraid of using iterators, generics, and traits; the compiler typically optimizes them away to the same performance as hand-written loops
- **Ownership signals intent** — taking `&T` vs `&mut T` vs `T` in a function signature documents exactly what happens to the data; use this deliberately

```rust
// Illegal states unrepresentable: can't construct an OrderStatus
// with an invalid combination of fields
enum OrderStatus {
    Pending,
    Shipped { tracking_number: String },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String },
}
```

---

## 15. Pre-Deployment Checklist

- [ ] `cargo fmt --check` and `cargo clippy -- -D warnings` pass cleanly
- [ ] `cargo test --all-features` passes
- [ ] No `unwrap()`/`expect()` in production request-handling paths (grep for it as a CI check if needed)
- [ ] `cargo audit` shows no known vulnerabilities
- [ ] No secrets in code, config, or logs
- [ ] Timeouts set on all outbound async I/O
- [ ] Structured tracing with span context in place
- [ ] Config externalized and validated at startup
- [ ] Built with `--release` for deployment
- [ ] Graceful shutdown handled (drain in-flight requests on SIGTERM, common with `tokio::signal`)

---

## 16. Tools Summary

| Purpose | Tool |
|---|---|
| Formatting | `rustfmt` |
| Linting | `clippy` |
| Error types | `thiserror` (libraries), `anyhow` (applications) |
| Async runtime | `tokio` |
| Logging/tracing | `tracing` + `tracing-subscriber` |
| Config | `envy`, `figment`, `config` |
| Testing | built-in `#[test]`, `proptest`, `criterion` |
| HTTP mocking | `wiremock`, `mockito` |
| Vulnerability scan | `cargo audit`, `cargo deny` |
| Web framework (optional) | `axum`, `actix-web` |
| DB access | `sqlx` (async, compile-time checked queries), `diesel` (sync ORM) |
| Coverage | `cargo tarpaulin`, `cargo llvm-cov` |
| Profiling | `cargo flamegraph`, `perf` |

---

## 17. How to Build the Habit

1. **Ban `unwrap()`/`expect()` from any code path that handles real input** — treat every one you write as a decision, not a default.
2. **Write the `#[cfg(test)] mod tests` block in the same file as you write the function** — Rust makes this frictionless, so there's no excuse to defer it.
3. **Run `cargo clippy -- -D warnings` before every commit** (or wire it into a pre-commit hook / CI gate).
4. **Model your domain with enums and newtypes** before reaching for primitive types and runtime checks.
5. **Read the standard library and popular crates' source** (`tokio`, `serde`) — idiomatic Rust is best learned by example, and the ecosystem's top crates are unusually well-written.

---

### Quick Reference: The 80/20 That Matters Most

1. No `unwrap()`/`expect()` in production code paths — use `Result` + `?` + `thiserror`/`anyhow`
2. `cargo clippy -- -D warnings` and `cargo fmt --check` in CI, always
3. Tests alongside code (`#[cfg(test)]`) + `cargo test` in CI
4. `tracing` for structured, contextual logs instead of `println!`
5. `cargo audit` for dependency vulnerabilities before every release
