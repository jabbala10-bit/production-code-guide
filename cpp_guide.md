# Writing Production-Level C++: A Practical Guide

## 1. Mindset Shift

| Quick Code | Production Code |
|---|---|
| Raw `new`/`delete` | RAII, smart pointers, no manual lifetime tracking |
| Raw pointers everywhere | Ownership expressed in the type (`unique_ptr`, references, spans) |
| `using namespace std;` in headers | Scoped, deliberate namespace usage |
| No tests | Unit tests, sanitizers, fuzzing |
| `printf`/`cout` debugging | Structured logging |
| "It compiles, no crashes yet" | "It compiles clean under sanitizers, warnings-as-errors, and static analysis" |

C++ gives you the most rope of any language on this list — manual memory, undefined behavior, and no compiler safety net for lifetimes by default. Production discipline here is largely about **using modern C++ (17/20/23) idioms that make entire bug classes structurally impossible**, and leaning hard on tooling to catch what the language itself won't.

---

## 2. Project Structure

```
my-project/
├── CMakeLists.txt
├── cmake/
│   └── CompilerWarnings.cmake
├── include/
│   └── myproject/
│       ├── core.hpp
│       └── config.hpp
├── src/
│   ├── core.cpp
│   └── main.cpp
├── tests/
│   ├── CMakeLists.txt
│   └── core_test.cpp
├── third_party/           # or managed via package manager
├── .clang-format
├── .clang-tidy
├── conanfile.txt (or vcpkg.json)
├── .gitignore
└── README.md
```

**Tips:**
- Separate `include/` (public API, what consumers `#include`) from `src/` (implementation) for anything meant to be used as a library.
- Use **CMake** as the build system — it's the de facto standard for portability across compilers/platforms.
- Prefix public headers with the project/namespace directory (`myproject/core.hpp`) to avoid include collisions when installed system-wide.

---

## 3. Code Quality Checklist

- [ ] Target **C++17 minimum**, prefer **C++20/23** where the toolchain allows — modern C++ eliminates most of the reasons old C++ was dangerous
- [ ] Use **clang-format** for consistent formatting — no style debates
- [ ] Use **clang-tidy** for static analysis (catches modernization opportunities, common bugs, and style issues in one tool)
- [ ] Compile with warnings as errors: `-Wall -Wextra -Wpedantic -Werror` (GCC/Clang) or `/W4 /WX` (MSVC)
- [ ] Follow the **C++ Core Guidelines** (Stroustrup/Sutter) — the closest thing this language has to an authoritative style bible
- [ ] Never `using namespace std;` in a header file (pollutes every translation unit that includes it)

```cmake
# cmake/CompilerWarnings.cmake
add_compile_options(-Wall -Wextra -Wpedantic -Werror -Wshadow -Wconversion)
```

---

## 4. Memory Management & RAII (The Core C++-Specific Concern)

This is where "production C++" is most distinct from every other language in this collection — there's no garbage collector, so ownership must be explicit and automatic.

```cpp
// Bad — manual lifetime management, leaks on early return/exception
Connection* openConnection(const std::string& host) {
    Connection* conn = new Connection(host);
    if (!conn->connect()) {
        delete conn; // easy to forget this line
        return nullptr;
    }
    return conn; // caller must remember to delete — easy to forget
}

// Good — RAII, ownership is automatic and exception-safe
std::unique_ptr<Connection> openConnection(const std::string& host) {
    auto conn = std::make_unique<Connection>(host);
    if (!conn->connect()) {
        return nullptr; // destructor runs automatically
    }
    return conn; // ownership transferred, no leak possible
}
```

**Best practices:**
- [ ] **Never use raw `new`/`delete`** in application code — use `std::make_unique`/`std::make_shared` instead
- [ ] Default to `std::unique_ptr` for single ownership; reach for `std::shared_ptr` only when shared ownership is genuinely needed (it has real overhead — atomic refcounting)
- [ ] Use `std::weak_ptr` to break reference cycles between `shared_ptr`s
- [ ] Pass by `const&` for read-only access to avoid copies; pass by value + `std::move` when taking ownership
- [ ] Use `std::span` (C++20) or `(pointer, size)` pairs instead of raw arrays decaying to pointers
- [ ] Rule of Zero: prefer types that need no custom destructor/copy/move (let the compiler generate them from RAII members) over the Rule of Five
- [ ] Never return a reference or pointer to a local/temporary — this is one of the most common sources of undefined behavior

```cpp
// Rule of Zero in action — no custom destructor/copy/move needed at all,
// because every member manages its own lifetime.
class OrderService {
    std::unique_ptr<Database> db_;
    std::vector<std::string> pendingOrders_;
    std::shared_ptr<Logger> logger_;
    // compiler-generated destructor/move are correct and sufficient
};
```

---

## 5. Error Handling

C++ has multiple competing error-handling idioms — pick one deliberately per project/team, and be consistent.

```cpp
// Exceptions — idiomatic for genuinely exceptional, rare failures
class OrderNotFoundException : public std::runtime_error {
public:
    explicit OrderNotFoundException(const std::string& orderId)
        : std::runtime_error("Order not found: " + orderId) {}
};

Order findOrder(const std::string& orderId) {
    auto it = orders_.find(orderId);
    if (it == orders_.end()) {
        throw OrderNotFoundException(orderId);
    }
    return it->second;
}

// std::expected (C++23) / tl::expected (pre-23) — for expected, common failure paths
std::expected<Order, OrderError> findOrder(const std::string& orderId) {
    auto it = orders_.find(orderId);
    if (it == orders_.end()) {
        return std::unexpected(OrderError::NotFound);
    }
    return it->second;
}
```

**Best practices:**
- [ ] Use **exceptions** for genuinely exceptional conditions (corrupted state, unrecoverable I/O failure), not routine control flow
- [ ] Use **`std::expected<T, E>`** (C++23) or **`std::optional<T>`** for expected, common failure paths (e.g., "value not found") — avoids exception overhead and makes the failure visible in the signature
- [ ] Mark functions that cannot throw with `noexcept` — enables compiler optimizations and documents intent
- [ ] Never let an exception escape a destructor (undefined behavior if it happens during stack unwinding)
- [ ] Always catch exceptions by `const&`, never by value (avoids slicing) or by pointer
- [ ] Avoid error codes via output parameters (`bool doThing(int* errorCode)`) — `std::expected`/exceptions are clearer and harder to ignore accidentally

```cpp
try {
    auto order = findOrder(orderId);
} catch (const OrderNotFoundException& e) {
    logger.error("{}", e.what());
    throw; // re-throw if this layer can't handle it — never swallow silently
}
```

---

## 6. Logging

```cpp
#include <spdlog/spdlog.h>

spdlog::info("Processing order order_id={} amount={}", orderId, amount);
spdlog::error("Payment failed order_id={} error={}", orderId, e.what());
```

**Checklist:**
- [ ] Use a real logging library (**spdlog** is the most common modern choice) — never bare `std::cout`/`printf` in production code paths
- [ ] Structured, leveled logging: `trace`, `debug`, `info`, `warn`, `error`, `critical`
- [ ] Never log secrets/credentials/PII
- [ ] Use async logging (spdlog supports this) in latency-sensitive hot paths so logging I/O doesn't block application logic
- [ ] Flush logs appropriately on crash/abnormal termination for post-mortem debugging

---

## 7. Configuration Management

```cpp
struct Config {
    int port;
    std::string databaseUrl;
    bool debug;

    static Config fromEnvironment() {
        Config cfg;
        cfg.port = std::stoi(getEnvOrDefault("PORT", "8080"));
        cfg.databaseUrl = getEnvOrThrow("DATABASE_URL");
        cfg.debug = getEnvOrDefault("DEBUG", "false") == "true";
        return cfg;
    }
};
```

- [ ] Externalize config via environment variables or config files (JSON/YAML/TOML via a library like `nlohmann/json` or `toml++`)
- [ ] Validate configuration at startup — fail fast with a clear error rather than crashing deep in application logic
- [ ] Never hardcode secrets/URLs/paths in source
- [ ] Separate configuration per build type (Debug/Release) and environment

---

## 8. Testing

```cpp
#include <gtest/gtest.h>

TEST(DiscountCalculator, AppliesPercentageDiscount) {
    EXPECT_DOUBLE_EQ(calculateDiscount(100.0, 0.1), 90.0);
}

TEST(DiscountCalculator, ThrowsOnNegativePrice) {
    EXPECT_THROW(calculateDiscount(-10.0, 0.1), std::invalid_argument);
}

class DiscountParamTest : public ::testing::TestWithParam<std::tuple<double, double, double>> {};

TEST_P(DiscountParamTest, VariousDiscounts) {
    auto [price, rate, expected] = GetParam();
    EXPECT_DOUBLE_EQ(calculateDiscount(price, rate), expected);
}

INSTANTIATE_TEST_SUITE_P(Discounts, DiscountParamTest, ::testing::Values(
    std::make_tuple(100.0, 0.0, 100.0),
    std::make_tuple(100.0, 1.0, 0.0),
    std::make_tuple(50.0, 0.5, 25.0)
));
```

**Checklist:**
- [ ] **GoogleTest (gtest)** is the most widely used framework; **Catch2** is a strong lighter-weight alternative
- [ ] Use parameterized tests for table-driven cases, same idea as Go/Python
- [ ] Mock dependencies with **GoogleMock** or hand-rolled interfaces + fakes
- [ ] **Run tests under sanitizers** — this is the single highest-leverage C++-specific testing practice:
  - `-fsanitize=address` (AddressSanitizer) — catches buffer overflows, use-after-free
  - `-fsanitize=undefined` (UndefinedBehaviorSanitizer) — catches UB like signed overflow, null deref
  - `-fsanitize=thread` (ThreadSanitizer) — catches data races
- [ ] Use **fuzzing** (libFuzzer, AFL++) for any code that parses untrusted input — extremely effective at finding memory bugs
- [ ] Measure coverage with `gcov`/`llvm-cov`
- [ ] Run the full suite, under sanitizers, in CI on every push

```bash
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g" -B build
cmake --build build
ctest --test-dir build
```

---

## 9. Documentation

```cpp
/// Calculates the discounted price.
///
/// @param price Original price, must be non-negative.
/// @param discountRate Fraction between 0 and 1 inclusive.
/// @return The price after the discount is applied.
/// @throws std::invalid_argument if price is negative or discountRate is out of range.
double calculateDiscount(double price, double discountRate) {
    if (price < 0) throw std::invalid_argument("price must be non-negative");
    if (discountRate < 0.0 || discountRate > 1.0) {
        throw std::invalid_argument("discountRate must be between 0 and 1");
    }
    return price * (1 - discountRate);
}
```

- [ ] **Doxygen**-style comments on public API (`///` or `/** */`) — generates browsable HTML docs automatically
- [ ] `README.md` with build instructions (CMake commands, required compiler version, dependencies)
- [ ] Document ownership/lifetime expectations explicitly in comments where the type system can't express them (e.g., "caller must ensure the referenced object outlives this view")
- [ ] Comment *why*, especially around any necessary but non-obvious low-level trick or workaround

---

## 10. Dependency Management

- [ ] Use a real package manager — **Conan** or **vcpkg** — instead of vendoring source or manual `FetchContent` sprawl
- [ ] Pin dependency versions explicitly
- [ ] `CMakeLists.txt` should be the single source of truth for how the project builds — avoid hand-maintained Makefiles alongside it
- [ ] Keep the dependency footprint small; each C++ dependency adds real build-time and ABI-compatibility burden
- [ ] Regularly audit dependencies for known CVEs (relevant for anything statically linking third-party C/C++ libraries, which have a long history of memory-safety CVEs)

---

## 11. Security Basics

- [ ] Compile with sanitizers in CI (Section 8) — the majority of C++ CVEs are memory-safety bugs these catch directly
- [ ] Use parameterized queries for any DB access — never string-concatenate SQL
- [ ] Validate all external/untrusted input before use, especially anything sized/indexed by that input
- [ ] Prefer `std::string`/`std::vector`/`std::array` over raw C-style arrays and `char*` buffers
- [ ] Avoid C-style casts; use `static_cast`/`dynamic_cast`/`reinterpret_cast` — each documents intent and is greppable, unlike `(Type)value`
- [ ] Run **Clang Static Analyzer** or commercial tools (Coverity, PVS-Studio) for deeper analysis than clang-tidy alone
- [ ] Enable stack protection and fortification flags in release builds (`-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2`)

```cpp
// Bad — C-style cast hides what's actually happening
Derived* d = (Derived*)basePtr;

// Good — explicit, and dynamic_cast is checked at runtime
Derived* d = dynamic_cast<Derived*>(basePtr);
if (!d) {
    // handle the cast failure — dynamic_cast returns nullptr on failure for pointers
}
```

---

## 12. Concurrency

- [ ] Prefer `std::jthread` (C++20, auto-joins and supports cooperative cancellation) over raw `std::thread`
- [ ] Use `std::mutex` + `std::lock_guard`/`std::scoped_lock` (RAII locking) — never manually call `lock()`/`unlock()`
- [ ] Use `std::atomic<T>` for simple shared counters/flags instead of a mutex where it fits
- [ ] Prefer message-passing (queues) or higher-level abstractions (`std::async`, thread pools) over hand-rolled thread synchronization where possible
- [ ] **Always run ThreadSanitizer** on any code touching shared mutable state across threads — data races are undefined behavior in C++, not just a logic bug
- [ ] Understand memory ordering (`std::memory_order`) before reaching for relaxed atomics — the default (`seq_cst`) is safe; anything else needs real justification

```cpp
std::mutex mtx;
int sharedCounter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx); // RAII — unlocked automatically, even on exception
    ++sharedCounter;
}
```

---

## 13. Performance & Efficiency

- [ ] Profile before optimizing (`perf`, VTune, Instruments) — C++ performance intuition is frequently wrong
- [ ] Prefer stack allocation and value semantics over heap allocation where object size/lifetime allow it
- [ ] Reserve `std::vector` capacity up front (`.reserve(n)`) when the size is known, to avoid repeated reallocation
- [ ] Pass by `const&` for anything larger than a couple of words; use move semantics (`std::move`) to avoid unnecessary copies at ownership transfer points
- [ ] Understand the cost model: heap allocation, virtual dispatch, and exceptions all have real overhead — use them where they add value, not everywhere by default
- [ ] Enable link-time optimization (`-flto`) and appropriate optimization flags (`-O2`/`-O3`) for release builds
- [ ] Use `constexpr`/`consteval` to move computation to compile time where possible

---

## 14. Version Control & CI/CD

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake -DCMAKE_CXX_FLAGS="-Wall -Wextra -Werror -fsanitize=address,undefined" -B build
      - name: Build
        run: cmake --build build
      - name: Test
        run: ctest --test-dir build --output-on-failure
      - name: clang-tidy
        run: run-clang-tidy -p build
```

- [ ] `.gitignore` covers `build/`, `CMakeCache.txt`, compiled binaries
- [ ] CI builds with warnings-as-errors, runs tests under sanitizers, and runs clang-tidy — all as separate, visible gates
- [ ] Build a Release configuration separately in CI to catch optimization-level-specific bugs (surprisingly common with UB)
- [ ] Cache build artifacts (`ccache`) to keep CI times reasonable — C++ compile times are a classic pain point

---

## 15. Design Principles Worth Internalizing

- **RAII everywhere** — every resource (memory, file handle, lock, socket) should be owned by an object whose destructor releases it automatically
- **Rule of Zero > Rule of Five** — design types so the compiler-generated special member functions are correct, rather than hand-writing five of them
- **Const-correctness** — mark everything `const` that can be; it documents intent and the compiler enforces it
- **Prefer value semantics** — copyable, comparable value types are easier to reason about than pointer-heavy object graphs
- **Express ownership in the type system** — `unique_ptr` says "I own this alone," `T&` says "I don't own this, don't outlive it," `shared_ptr` says "ownership is shared" — pick deliberately, don't default to raw pointers because "it's simpler"
- **Prefer the algorithm library (`<algorithm>`) over hand-written loops** where it improves clarity — but don't force it where a plain loop reads better

```cpp
// Ownership is legible from the signature alone:
void processReadOnly(const Order& order);      // borrows, doesn't own, won't modify
void processAndTakeOwnership(std::unique_ptr<Order> order); // takes ownership
void processShared(std::shared_ptr<Order> order);  // shares ownership
```

---

## 16. Pre-Deployment Checklist

- [ ] Compiles cleanly with `-Wall -Wextra -Wpedantic -Werror`
- [ ] clang-tidy passes with no unaddressed warnings
- [ ] Full test suite passes under AddressSanitizer, UndefinedBehaviorSanitizer, and (where concurrency is involved) ThreadSanitizer
- [ ] No raw `new`/`delete` in application code — audited via grep/clang-tidy
- [ ] No secrets in code, config, or logs
- [ ] All external/untrusted input validated before use
- [ ] Structured logging in place, with appropriate levels
- [ ] Config externalized and validated at startup
- [ ] Release build tested separately from Debug build (optimization can surface latent UB)
- [ ] Dependencies pinned via Conan/vcpkg, checked for known CVEs

---

## 17. Tools Summary

| Purpose | Tool |
|---|---|
| Build system | CMake |
| Package manager | Conan, vcpkg |
| Formatting | clang-format |
| Static analysis | clang-tidy, Clang Static Analyzer, Coverity/PVS-Studio (commercial) |
| Testing | GoogleTest/GoogleMock, Catch2 |
| Sanitizers | ASan, UBSan, TSan (built into GCC/Clang) |
| Fuzzing | libFuzzer, AFL++ |
| Logging | spdlog |
| JSON/config | nlohmann/json, toml++ |
| Coverage | gcov, llvm-cov |
| Profiling | perf, VTune, Instruments |
| Build caching | ccache |
| CI/CD | GitHub Actions / GitLab CI |

---

## 18. How to Build the Habit

1. **Never write a raw `new`/`delete` again** — reach for `make_unique`/`make_shared`/containers by default, and treat any raw `new` as a code-review flag.
2. **Turn on `-Wall -Wextra -Werror` and sanitizers from commit one** of every new project — retrofitting them onto a large existing codebase is a multi-week project, not a quick add.
3. **Run tests under ASan/UBSan locally**, not just in CI — catching a heap corruption bug the moment you introduce it is far cheaper than debugging it three commits later.
4. **Express ownership explicitly** in every function signature you write — it's the habit that most directly prevents C++'s classic bug classes.
5. **Read the C++ Core Guidelines** — they're maintained by the language's own designers and are the closest thing to an authoritative answer for "is this idiom safe/idiomatic."

---

### Quick Reference: The 80/20 That Matters Most

1. RAII + smart pointers — no raw `new`/`delete` in application code
2. `-Wall -Wextra -Werror` plus AddressSanitizer/UBSan in every CI run
3. GoogleTest for core logic, with sanitizers enabled during test runs
4. `const&` for read access, `std::move` for ownership transfer — deliberate value semantics
5. clang-tidy + C++ Core Guidelines as the default idiom reference, not "how C++ was taught 15 years ago"
