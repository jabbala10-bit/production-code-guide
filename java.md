# Writing Production-Level Java: A Practical Guide

## 1. Mindset Shift: Script vs. Production Code

| Quick/Throwaway Code | Production Code |
|---|---|
| Works once, on your machine | Works reliably, everywhere, every time |
| Unchecked exceptions everywhere | Deliberate exception strategy |
| Hardcoded values | Externalized configuration |
| `System.out.println` | Structured logging (SLF4J) |
| No tests | Unit + integration tests |
| God classes, static everything | Modular, SOLID design |
| "It compiles" | "It compiles, is tested, documented, and maintainable" |

---

## 2. Project Structure (Maven/Gradle Standard Layout)

```
my-project/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/myapp/
│   │   │       ├── config/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── model/
│   │   │       ├── exception/
│   │   │       └── util/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/
│           └── com/company/myapp/
│               ├── service/
│               └── controller/
├── pom.xml (or build.gradle)
├── .gitignore
├── README.md
└── Dockerfile
```

**Tips:**
- Package by **feature/domain** for larger apps (`order/`, `payment/`) rather than only by layer, once the project grows.
- Keep `test/` mirroring `main/` package structure exactly — makes tests easy to locate.
- One public class per file, file name matches class name (enforced by the compiler anyway).

---

## 3. Code Quality Checklist

### Style & Formatting
- [ ] Follow **Google Java Style Guide** or your team's agreed convention
- [ ] Auto-format with **Spotless** or **google-java-format** (plugged into the build)
- [ ] Consistent brace style, 4-space indentation (standard)

### Static Analysis
- [ ] **Checkstyle** for style enforcement
- [ ] **SpotBugs** (successor to FindBugs) for bug pattern detection
- [ ] **PMD** for code smells and complexity
- [ ] **SonarQube/SonarLint** for combined quality + security scanning
- [ ] Enable and fix compiler warnings (`-Xlint:all`)

### Naming
- [ ] `camelCase` for methods/variables, `PascalCase` for classes/interfaces, `UPPER_SNAKE_CASE` for constants
- [ ] Interfaces named for what they do (`PaymentProcessor`), not prefixed with `I` (that's C#/legacy convention)
- [ ] Avoid abbreviations; prefer `orderRepository` over `ordRepo`

---

## 4. Exception Handling

Java's checked vs. unchecked exception distinction matters — use it deliberately.

```java
// Bad — swallowing exceptions
try {
    processPayment(order);
} catch (Exception e) {
    // nothing here — silent failure
}

// Bad — catching generic Exception and rethrowing generic
try {
    processPayment(order);
} catch (Exception e) {
    throw new RuntimeException(e);
}

// Good — specific, informative, preserves cause
public class PaymentProcessingException extends RuntimeException {
    public PaymentProcessingException(String message, Throwable cause) {
        super(message, cause);
    }
}

try {
    processPayment(order);
} catch (PaymentGatewayTimeoutException e) {
    logger.error("Payment gateway timed out for orderId={}", order.getId(), e);
    throw new PaymentProcessingException("Unable to process payment for order " + order.getId(), e);
}
```

**Best practices:**
- [ ] Prefer **unchecked (runtime) exceptions** for most application errors in modern Java codebases — checked exceptions tend to leak into every signature and get swallowed
- [ ] Create a small hierarchy of domain-specific exceptions rather than throwing raw `RuntimeException`
- [ ] Never catch `Exception` or `Throwable` broadly unless at a top-level boundary (e.g., a global exception handler)
- [ ] Always preserve the original cause (`new MyException(msg, cause)`)
- [ ] Use `try-with-resources` for anything `Closeable`/`AutoCloseable`

```java
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(SQL)) {
    // use resources
} // auto-closed, even on exception
```

- [ ] In Spring apps, use `@ControllerAdvice` / `@ExceptionHandler` for centralized error handling instead of per-controller try/catch sprawl

---

## 5. Logging (Not System.out.println)

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);

    public void processOrder(String orderId) {
        logger.info("Processing order orderId={}", orderId);
        try {
            // ...
        } catch (PaymentException e) {
            logger.error("Payment failed for orderId={}", orderId, e);
            throw e;
        }
    }
}
```

**Checklist:**
- [ ] Use **SLF4J** as the facade (with Logback or Log4j2 as the backend) — never `System.out`/`e.printStackTrace()`
- [ ] Use parameterized logging (`"orderId={}", id`) not string concatenation — avoids the cost of building strings when the log level is disabled
- [ ] Appropriate levels: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`
- [ ] Never log secrets, tokens, passwords, or full PII
- [ ] Use MDC (Mapped Diagnostic Context) to attach request/trace IDs across a request's log lines
- [ ] Consider structured (JSON) logging output for log aggregation (ELK, Splunk, Datadog)

---

## 6. Configuration Management

```java
// application.yml (Spring Boot)
app:
  payment-gateway:
    url: ${PAYMENT_GATEWAY_URL}
    timeout-ms: 5000
```

```java
@ConfigurationProperties(prefix = "app.payment-gateway")
public record PaymentGatewayProperties(String url, int timeoutMs) {}
```

- [ ] Externalize config via environment variables, `application-{profile}.yml`, or a config server (Spring Cloud Config, Consul)
- [ ] Never hardcode secrets, URLs, or credentials in code
- [ ] Use profiles for dev/staging/prod (`application-dev.yml`, `application-prod.yml`)
- [ ] Secrets via a secrets manager (AWS Secrets Manager, Vault, Kubernetes Secrets) — not committed files
- [ ] Validate configuration at startup (`@Validated` on config properties) so bad config fails fast, not at 2am in production

---

## 7. Testing

```java
class DiscountCalculatorTest {

    private final DiscountCalculator calculator = new DiscountCalculator();

    @Test
    void appliesPercentageDiscountCorrectly() {
        assertEquals(90.0, calculator.apply(100.0, 0.1));
    }

    @Test
    void throwsExceptionForNegativePrice() {
        assertThrows(IllegalArgumentException.class,
            () -> calculator.apply(-10.0, 0.1));
    }

    @ParameterizedTest
    @CsvSource({
        "100, 0.0, 100.0",
        "100, 1.0, 0.0",
        "50,  0.5, 25.0"
    })
    void appliesVariousDiscounts(double price, double rate, double expected) {
        assertEquals(expected, calculator.apply(price, rate));
    }
}
```

**Checklist:**
- [ ] **JUnit 5** as the standard testing framework
- [ ] **Mockito** for mocking dependencies
- [ ] **AssertJ** for fluent, readable assertions (`assertThat(x).isEqualTo(y)`)
- [ ] Aim for meaningful coverage (70–90%), not 100% for its own sake — use **JaCoCo** to measure
- [ ] Test edge cases: null, empty collections, boundary values, negative numbers
- [ ] Separate **unit tests** (fast, mocked dependencies) from **integration tests** (`@SpringBootTest`, Testcontainers for real DB/queue)
- [ ] Use **Testcontainers** for integration tests against real Postgres/Kafka/Redis instead of mocking everything
- [ ] Run the full suite in CI on every push/PR

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    // ...
}
```

---

## 8. Documentation

```java
/**
 * Calculates the discounted price for an order.
 *
 * @param price the original price, must be non-negative
 * @param discountRate a fraction between 0 and 1 inclusive
 * @return the price after the discount is applied
 * @throws IllegalArgumentException if price is negative or discountRate is out of range
 */
public double apply(double price, double discountRate) {
    if (price < 0) {
        throw new IllegalArgumentException("price cannot be negative");
    }
    if (discountRate < 0 || discountRate > 1) {
        throw new IllegalArgumentException("discountRate must be between 0 and 1");
    }
    return price * (1 - discountRate);
}
```

- [ ] Javadoc on all public classes/methods (generate HTML docs with `javadoc`/Maven `javadoc:javadoc`)
- [ ] A solid `README.md`: build instructions, how to run, how to test, architecture overview
- [ ] Comments explain **why**, not **what**
- [ ] Consider ADRs (Architecture Decision Records) for significant design choices

---

## 9. Dependency & Build Management

- [ ] **Maven** or **Gradle** — pick one, be consistent across the team
- [ ] Pin dependency versions explicitly; use a BOM (Bill of Materials) for related libraries (e.g., Spring Boot BOM) to avoid version drift
- [ ] Separate `test`-scoped dependencies from runtime/compile dependencies
- [ ] Regularly check for vulnerable dependencies (`mvn versions:display-dependency-updates`, OWASP Dependency-Check, Snyk, Dependabot)
- [ ] Avoid unnecessary dependencies — every dependency is a maintenance and security liability

---

## 10. Security Basics

- [ ] Never commit secrets — use `git-secrets` or pre-commit scanning
- [ ] Always use **parameterized queries / PreparedStatement** — never string-concatenate SQL
- [ ] Validate and sanitize all external input (`@Valid` + Bean Validation annotations in Spring)
- [ ] Keep dependencies patched — most Java CVEs come from outdated libraries (Log4Shell being the canonical example)
- [ ] Principle of least privilege for DB/service credentials
- [ ] Use HTTPS/TLS everywhere; don't disable certificate validation "just for now"

```java
// Bad — SQL injection risk
String sql = "SELECT * FROM users WHERE id = " + userId;

// Good
String sql = "SELECT * FROM users WHERE id = ?";
try (PreparedStatement stmt = connection.prepareStatement(sql)) {
    stmt.setString(1, userId);
    ResultSet rs = stmt.executeQuery();
}
```

---

## 11. Performance & Efficiency

- [ ] Profile before optimizing (JFR/Java Flight Recorder, async-profiler, VisualVM) — don't guess
- [ ] Choose the right collection: `ArrayList` vs `LinkedList`, `HashMap` vs `TreeMap`, understand Big-O of each operation you use
- [ ] Avoid unnecessary object creation in hot paths (autoboxing in tight loops, unnecessary `String` concatenation — use `StringBuilder`)
- [ ] Use streams for readability, but be aware they can be slower than plain loops in hot paths — profile if it matters
- [ ] Tune the JVM (heap size, GC algorithm — G1GC is default and fine for most; ZGC/Shenandoah for very low-latency needs)
- [ ] Use connection pooling (HikariCP) for databases — never open raw connections per request
- [ ] Cache expensive/repeated computations (Caffeine, Spring `@Cacheable`)

```java
@Cacheable("productPrices")
public BigDecimal getPrice(String productId) {
    return priceRepository.findById(productId);
}
```

---

## 12. Version Control Discipline

- [ ] Small, focused commits with clear messages (`fix: handle null response from payment gateway`, not `updates`)
- [ ] Feature branches + Pull Requests, even solo
- [ ] `.gitignore` covers `target/`, `build/`, `*.class`, `.idea/`, `*.iml`
- [ ] Conventional commits for team projects (`feat:`, `fix:`, `refactor:`, `test:`)

---

## 13. CI/CD Basics

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Build and test
        run: mvn -B verify
      - name: Static analysis
        run: mvn checkstyle:check spotbugs:check
```

- [ ] Build + test + static analysis run automatically on every push
- [ ] Fail the build on style/lint violations, not just test failures
- [ ] Use `mvn verify` (or Gradle `check`) as the CI gate — it runs tests + packaging together

---

## 14. Design Principles Worth Internalizing

- **SOLID** principles — especially Single Responsibility and Dependency Inversion (program to interfaces, inject dependencies rather than `new` them inside classes)
- **DRY**, but don't over-abstract prematurely
- **Favor immutability** — use `final` fields, records (Java 16+), and immutable collections where possible; immutable objects are inherently thread-safe
- **Composition over inheritance** — deep inheritance hierarchies are hard to reason about and modify
- **Dependency Injection** (Spring, or manual constructor injection) instead of static singletons/service locators
- **Fail fast** — validate inputs at the boundary, don't let bad state propagate deep into the call stack

```java
// Immutable value object using a record (Java 16+)
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("amount cannot be negative");
        }
    }
}
```

---

## 15. Concurrency Basics (Java-Specific Concern)

- [ ] Prefer immutable objects and higher-level concurrency utilities (`java.util.concurrent`) over raw `synchronized`/`wait`/`notify`
- [ ] Use `ExecutorService` / thread pools instead of manually spawning `Thread` objects
- [ ] Use `ConcurrentHashMap`, `CopyOnWriteArrayList`, `AtomicInteger`, etc. instead of manually synchronizing standard collections
- [ ] Be deliberate about thread-safety in Spring beans (singleton scope by default — avoid mutable instance state)
- [ ] Consider virtual threads (Java 21+, Project Loom) for high-throughput I/O-bound workloads — much cheaper than platform threads

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> processOrder(orderId));
}
```

---

## 16. Pre-Deployment Checklist

- [ ] All tests pass in CI (unit + integration)
- [ ] Static analysis (Checkstyle/SpotBugs/PMD/Sonar) passes with no new critical issues
- [ ] No secrets in code, config files, or logs
- [ ] Exception handling covers realistic failure modes (timeouts, downstream service failures, bad input)
- [ ] Logging in place with correlation/trace IDs for debugging in production
- [ ] Config externalized — no hardcoded environment-specific values
- [ ] Dependencies scanned for known vulnerabilities
- [ ] Health check / readiness & liveness endpoints exist (`/actuator/health` in Spring Boot)
- [ ] Monitoring/metrics wired up (Micrometer + Prometheus/Grafana, or equivalent)
- [ ] Rollback plan exists if deployment fails

---

## 17. Tools Summary (Recommended Stack)

| Purpose | Tool |
|---|---|
| Build | Maven or Gradle |
| Formatting | Spotless / google-java-format |
| Static analysis | Checkstyle, SpotBugs, PMD, SonarQube |
| Testing | JUnit 5, Mockito, AssertJ |
| Integration testing | Testcontainers |
| Coverage | JaCoCo |
| Logging | SLF4J + Logback |
| Config | Spring `@ConfigurationProperties`, or `.env` + environment variables |
| Dependency scanning | OWASP Dependency-Check, Snyk, Dependabot |
| DI/Framework | Spring Boot (most common), or Micronaut/Quarkus for lower-footprint services |
| Caching | Caffeine, Redis |
| CI/CD | GitHub Actions / GitLab CI / Jenkins |
| Monitoring | Micrometer + Prometheus/Grafana |

---

## 18. How to Build the Habit (Practical Path)

1. **Start every new project** from a Spring Initializr (or equivalent) skeleton with the standard layout above — don't build structure from scratch each time.
2. **Write the test alongside the method**, not "later."
3. **Set up Checkstyle + Spotless + SpotBugs** in your build file on day one, and let the build fail on violations.
4. **Use records and immutability by default** for data-carrying classes — mutable state should be the exception, not the default.
5. **Read a well-regarded open-source Java codebase** (Spring Framework itself, Apache Commons, Guava) to see these principles applied at scale.
6. **Review your own PRs** before asking someone else to — you'll catch a surprising amount.

---

### Quick Reference: The 80/20 That Matters Most

If you only adopt a few things, prioritize:
1. SLF4J logging instead of `System.out`/stack traces
2. A deliberate, small exception hierarchy — no bare `catch (Exception e) {}`
3. JUnit 5 + Mockito tests for core business logic
4. Externalized config (no hardcoded secrets/URLs)
5. Checkstyle/Spotless wired into the build so style is never a discussion

These five alone will make Java code noticeably more "production-grade" almost immediately.
