# Production-Grade Code: A Multi-Language Reference Guide

A practical, checklist-driven reference for writing production-quality code in **Python, Java, Go, Rust, and TypeScript**. Each language has its own detailed guide; this README ties them together, highlights what's actually different between them, and distills the takeaways that matter most regardless of which language you're writing.

## 📚 Guides in This Collection

| Language | Guide | Core Tension It Solves |
|---|---|---|
| 🐍 Python | [`production_python_guide.md`](./production_python_guide.md) | Dynamic typing → discipline via types, tests, and tooling |
| ☕ Java | [`production_java_guide.md`](./production_java_guide.md) | Verbosity/legacy patterns → modern, lean, well-tested design |
| 🐹 Go | [`production_go_guide.md`](./production_go_guide.md) | Simplicity by design → not cutting corners on errors/concurrency |
| 🦀 Rust | [`production_rust_guide.md`](./production_rust_guide.md) | Compiler already prevents bugs → don't fight it with shortcuts |
| 🟦 TypeScript | [`production_typescript_guide.md`](./production_typescript_guide.md) | Types vanish at runtime → validate the boundary |

Each guide follows the same structure: project layout, code quality, error handling, logging, config, testing, docs, dependencies, security, performance, CI/CD, design principles, and a pre-deployment checklist — so you can jump to the same section across languages and compare directly.

---

## 🎯 The One-Line Summary Per Language

| Language | If you only remember one thing |
|---|---|
| Python | Type hints + `mypy` + `pytest` turn "duck typing chaos" into a maintainable codebase |
| Java | Don't catch `Exception` broadly, and inject dependencies instead of `new`-ing them everywhere |
| Go | Never ignore an error — check and wrap every single one |
| Rust | If you're calling `.unwrap()` in production code, stop and ask why `Result` isn't being propagated |
| TypeScript | A type is not a guarantee — validate every external input with a schema (Zod) |

---

## 🔑 Key Differentiators Across Languages

### 1. How Each Language Forces (or Doesn't Force) Correctness

| Concern | Python | Java | Go | Rust | TypeScript |
|---|---|---|---|---|---|
| Type safety | Optional (gradual, via hints + mypy) | Enforced at compile time | Enforced at compile time | Enforced at compile time, very strict | Enforced at compile time, **erased at runtime** |
| Null safety | No built-in protection (`None` anywhere) | `Optional<T>` opt-in; `null` still pervasive | No `null`; zero values (`""`, `0`, `nil` for pointers) instead | No `null`; `Option<T>` enforced by compiler | `strictNullChecks` opt-in but recommended |
| Memory safety | Managed (GC) | Managed (GC) | Managed (GC) | Enforced at compile time, no GC | Managed (GC, via JS runtime) |
| Concurrency safety | GIL limits true parallelism; async via `asyncio` | Manual (`synchronized`, `java.util.concurrent`) | Built-in (goroutines + channels), but races still possible | Compiler-enforced (`Send`/`Sync`) — data races caught at compile time | Single-threaded event loop; no shared-memory races by design |
| Error handling | Exceptions | Exceptions (checked + unchecked) | Explicit `error` return values | `Result<T, E>` / `Option<T>`, compiler-enforced handling | Exceptions + `Promise` rejections |

**Takeaway:** the further right on the "compile-time enforcement" spectrum a language sits, the more its production checklist shifts from *"catch bugs before they ship"* toward *"don't defeat the safety the compiler already gives you"* (Rust's `.unwrap()`, TypeScript's `any`/unvalidated `unknown`). Python and Java sit at the other end, where **tooling and discipline substitute for what the compiler won't check.**

### 2. Error Handling Philosophy

This is the single biggest day-to-day difference between these languages:

- **Python/Java/TypeScript** — exceptions. Errors are thrown and can be caught anywhere up the call stack, including nowhere (silent crash or unhandled rejection).
- **Go** — errors are ordinary return values. There's no "throw" — you check `if err != nil` at every call site. Verbose, but nothing is hidden.
- **Rust** — `Result<T, E>` and `Option<T>` are types the compiler forces you to handle (or explicitly ignore via `.unwrap()`, which is a visible, greppable decision).

```
Python/Java/TS:  try { risky() } catch (e) { handle(e) }   // easy to forget the catch
Go:              val, err := risky(); if err != nil { ... } // impossible to silently forget err exists
Rust:             risky()?;                                  // compiler won't let you ignore Result
```

**Takeaway:** in exception-based languages, the *discipline* is remembering to catch and never swallow. In Go and Rust, the *language* removes the ability to silently forget — the discipline shifts to not bypassing that safety net (`_ = err` in Go, `.unwrap()` in Rust).

### 3. The Runtime-vs-Compile-Time Trust Gap

Every language has a boundary where the type system's guarantees stop being true:

| Language | Where types can lie |
|---|---|
| Python | Everywhere, until validated (no enforcement without mypy + runtime checks like Pydantic) |
| Java | Deserialization, reflection, unchecked casts (`@SuppressWarnings("unchecked")`) |
| Go | `interface{}`/`any` values, JSON unmarshaling into loosely-defined structs |
| Rust | `unsafe` blocks, and raw `serde` deserialization without validation |
| TypeScript | **Everywhere data crosses a runtime boundary** — API responses, `JSON.parse`, `process.env` — types are compile-time only and erased at runtime |

**Takeaway:** TypeScript has this problem the worst, because its type system is 100% erased at runtime with zero enforcement — hence why schema validation (Zod) is the single most emphasized practice in the TS guide. Every language needs this discipline at its I/O boundaries; TypeScript just needs it *everywhere*, not just at a few sharp edges.

### 4. Concurrency Model

| Language | Model | Production Concern |
|---|---|---|
| Python | Threads (GIL-limited) + `asyncio` | Use multiprocessing for CPU-bound work; `asyncio` for I/O-bound |
| Java | OS threads + `java.util.concurrent`; virtual threads (21+) | Thread-safety of shared mutable state; prefer immutability |
| Go | Goroutines + channels ("share memory by communicating") | Goroutine leaks, races (mitigated but not prevented by design) |
| Rust | `tokio`/async + compiler-enforced `Send`/`Sync` | Data races are a **compile error**, not a runtime bug |
| TypeScript | Single-threaded event loop + `Promise`/`async`/`await` | No data races, but unhandled rejections and event-loop blocking are the equivalent risk |

**Takeaway:** Go and Rust both make concurrency a first-class citizen, but only Rust makes data races impossible to compile. Go gives you the tools and the race detector, but the compiler won't stop you from writing one.

### 5. Tooling Convergence (What's Actually the Same Everywhere)

Despite the differences above, the *shape* of a good production checklist converges hard across all five languages:

| Category | Common thread |
|---|---|
| Formatting | Every language now has a de-facto standard, opinionated auto-formatter (Black, google-java-format, `gofmt`, `rustfmt`, Prettier) — the debate is settled, just turn it on |
| Linting | Every ecosystem has a mature static analyzer that should gate CI (Ruff, Checkstyle/SpotBugs, `golangci-lint`, Clippy, ESLint) |
| Testing | Table/parameterized tests, mocking external dependencies, and containerized integration tests (Testcontainers exists for all five) are universal patterns |
| Structured logging | Every guide says the same thing: no `print`/`console.log`/`System.out` in production — use a structured logger with correlation IDs |
| Config | Externalize via env vars, validate at startup, fail fast — identical advice in every language |
| Dependency hygiene | Lockfiles + vulnerability scanning (`pip-audit`, OWASP Dependency-Check, `govulncheck`, `cargo audit`, `npm audit`) are equivalent tools solving the same problem |
| CI gate shape | Format check → lint → typecheck (where applicable) → test → build, in that order, in every single guide |

---

## 🗺️ Cross-Language Concept Map

Use this when switching contexts between languages — it maps the same underlying concern to its idiomatic tool/pattern in each:

| Concern | Python | Java | Go | Rust | TypeScript |
|---|---|---|---|---|---|
| Formatter | Black | Spotless / google-java-format | `gofmt` | `rustfmt` | Prettier |
| Linter | Ruff | Checkstyle / SpotBugs | `golangci-lint` | Clippy | ESLint |
| Type checker | mypy / pyright | (built-in, compiled) | (built-in, compiled) | (built-in, compiled) | `tsc` |
| Test framework | pytest | JUnit 5 | `testing` (stdlib) | built-in `#[test]` | Vitest / Jest |
| Mocking | `unittest.mock` | Mockito | manual / interfaces | `mockall` | `vi.mock` |
| Integration testing | Testcontainers | Testcontainers | Testcontainers-go | (containers via `testcontainers-rs`) | Testcontainers (Node) |
| Structured logging | `logging` / `structlog` | SLF4J + Logback | `log/slog`, `zap` | `tracing` | Pino |
| Config validation | Pydantic Settings | `@ConfigurationProperties` | `envy` / `viper` | `envy` / `figment` | Zod on `process.env` |
| Dependency lock | `poetry.lock` / `uv.lock` | `pom.xml` + BOM | `go.sum` | `Cargo.lock` | `package-lock.json` / `pnpm-lock.yaml` |
| Vulnerability scan | `pip-audit` | OWASP Dependency-Check | `govulncheck` | `cargo audit` | `npm audit` |
| Custom errors | Exception subclasses | Exception subclasses | Sentinel errors / wrapped `error` | `thiserror` enums | Error class hierarchy |
| Runtime validation | Pydantic | Bean Validation (`@Valid`) | manual / `validator` pkg | `serde` + manual checks | Zod |
| Async/concurrency | `asyncio` | `CompletableFuture` / virtual threads | goroutines + channels | `tokio` | `Promise` / `async`-`await` |

---

## ✅ The Universal Pre-Deployment Checklist

Regardless of language, these are non-negotiable before shipping:

- [ ] Formatter and linter pass cleanly, wired into CI as a hard gate
- [ ] All tests pass (unit + integration), including against real dependencies via containers
- [ ] No secrets in code, config files, or logs
- [ ] Every external input is validated at the boundary (don't trust it just because it typechecks or "should" be fine)
- [ ] Every failure mode (timeout, downstream failure, bad input) is handled explicitly — nothing fails silently
- [ ] Structured logging with correlation/trace IDs, not print-statement debugging
- [ ] Config is externalized and validated at startup — fail fast on misconfiguration, not mid-request
- [ ] Dependencies are locked and scanned for known vulnerabilities
- [ ] Health/readiness checks exist for orchestration
- [ ] Graceful shutdown is implemented (drain in-flight work before exiting)
- [ ] Monitoring/metrics are wired up, not bolted on after the first incident

---

## 🧭 How to Use This Repo

1. **New to a language?** Start with its individual guide — each is self-contained with copy-pasteable examples.
2. **Switching between languages regularly?** Keep the [Cross-Language Concept Map](#-cross-language-concept-map) open as a translation table.
3. **Reviewing a PR or auditing a codebase?** Use the [Universal Pre-Deployment Checklist](#-the-universal-pre-deployment-checklist) as a baseline, then check the language-specific guide for anything deeper.
4. **Onboarding a team?** Each guide's "Quick Reference: The 80/20" section at the bottom is the fastest way to align a team on non-negotiables in under 10 minutes.

---

## 📌 Final Takeaway

Production-quality code isn't about mastering a language's syntax — it's about closing the gap between **"this compiles/runs"** and **"this behaves correctly under conditions I didn't anticipate."** Every language gives you different tools for that (a stricter compiler, a runtime validator, an explicit error type), but the underlying discipline is identical:

> **Validate inputs. Handle failures explicitly. Make illegal states hard to represent. Test the logic that matters. Log enough to debug production without guessing. Never let "it works on my machine" be the bar.**

The five guides in this repo are the language-specific implementations of that one idea.
