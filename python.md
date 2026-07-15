# Writing Production-Level Python: A Practical Guide

## 1. Mindset Shift: Script vs. Production Code

| Script/Notebook Code | Production Code |
|---|---|
| Works once, on your machine | Works reliably, everywhere, every time |
| No error handling | Anticipates and handles failure |
| Hardcoded values | Configurable via env vars/config files |
| No tests | Tested (unit, integration) |
| Print statements | Structured logging |
| One giant file | Modular, organized structure |
| "It works" | "It works, is documented, and the next person can maintain it" |

---

## 2. Project Structure

Use a consistent, standard layout — don't reinvent this every time.

```
my_project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── core.py
│       ├── config.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   ├── test_core.py
│   └── conftest.py
├── docs/
├── .env.example
├── .gitignore
├── pyproject.toml
├── README.md
└── requirements.txt / poetry.lock
```

**Tips:**
- Use `src/` layout (not flat) — it prevents accidental imports of uninstalled code and catches packaging bugs early.
- One package = one clear responsibility.
- Keep `tests/` mirroring your `src/` structure.

---

## 3. Code Quality Checklist

### Style & Formatting
- [ ] Follow **PEP 8** (or a house style guide)
- [ ] Use **Black** for auto-formatting (removes debates about style)
- [ ] Use **isort** for import ordering
- [ ] Line length consistent (Black default: 88 chars)

### Static Analysis
- [ ] **Ruff** or **Flake8** for linting (Ruff is much faster, increasingly the standard)
- [ ] **mypy** or **pyright** for static type checking
- [ ] Type hints on all public functions/methods

```python
# Bad
def process(data, threshold=0.5):
    ...

# Good
def process(data: list[float], threshold: float = 0.5) -> dict[str, float]:
    ...
```

### Naming
- [ ] Descriptive names over comments explaining bad names
- [ ] `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_CASE` for constants
- [ ] Avoid abbreviations unless universally understood (`cfg`, `df` for DataFrame are fine; `x1`, `tmp2` are not)

---

## 4. Error Handling

Production code fails gracefully — it never crashes silently or with a stack trace the user can't act on.

```python
# Bad
def get_user(user_id):
    return db.query(user_id)  # what if it doesn't exist? connection fails?

# Good
class UserNotFoundError(Exception):
    """Raised when a user cannot be found in the database."""

def get_user(user_id: str) -> User:
    try:
        user = db.query(user_id)
    except ConnectionError as e:
        logger.error("Database connection failed for user_id=%s", user_id)
        raise DatabaseUnavailableError("Could not reach database") from e

    if user is None:
        raise UserNotFoundError(f"No user found with id={user_id}")
    return user
```

**Best practices:**
- [ ] Catch **specific** exceptions, never bare `except:`
- [ ] Create custom exception classes for your domain
- [ ] Use `raise ... from e` to preserve the original traceback
- [ ] Fail fast — validate inputs early
- [ ] Never swallow exceptions silently (`except: pass` is a red flag)

---

## 5. Logging (Not print statements)

```python
import logging

logger = logging.getLogger(__name__)

def process_order(order_id: str) -> None:
    logger.info("Processing order", extra={"order_id": order_id})
    try:
        ...
    except PaymentError:
        logger.exception("Payment failed for order_id=%s", order_id)
        raise
```

**Checklist:**
- [ ] Use the `logging` module, not `print()`
- [ ] Use appropriate levels: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`
- [ ] Include context (IDs, timestamps) in log messages
- [ ] Never log secrets/passwords/tokens
- [ ] Consider structured logging (JSON) for log aggregation tools (ELK, Datadog, etc.)

---

## 6. Configuration Management

Never hardcode secrets, URLs, or environment-specific values.

```python
# Bad
API_KEY = "sk-abc123..."
DB_HOST = "prod-db.company.com"

# Good — use environment variables / pydantic settings
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_key: str
    db_host: str
    debug: bool = False

    class Config:
        env_file = ".env"

settings = Settings()
```

- [ ] Secrets in environment variables or a secrets manager (AWS Secrets Manager, Vault) — never in code or committed `.env` files
- [ ] Separate configs for dev/staging/prod
- [ ] `.env.example` committed, `.env` gitignored

---

## 7. Testing

Untested code is legacy code the moment it's written.

```python
# test_core.py
import pytest
from my_package.core import calculate_discount

def test_calculate_discount_applies_percentage():
    assert calculate_discount(100, 0.1) == 90

def test_calculate_discount_raises_on_negative_price():
    with pytest.raises(ValueError):
        calculate_discount(-10, 0.1)

@pytest.mark.parametrize("price,discount,expected", [
    (100, 0.0, 100),
    (100, 1.0, 0),
    (50, 0.5, 25),
])
def test_calculate_discount_various(price, discount, expected):
    assert calculate_discount(price, discount) == expected
```

**Checklist:**
- [ ] Use **pytest** (industry standard)
- [ ] Aim for meaningful coverage, not 100% for its own sake (70–90% is typical for solid production code)
- [ ] Test edge cases: empty inputs, `None`, negative numbers, huge inputs
- [ ] Use fixtures (`conftest.py`) to avoid duplication
- [ ] Mock external calls (APIs, DBs) with `unittest.mock` or `pytest-mock`
- [ ] Separate unit tests (fast, isolated) from integration tests (slower, real dependencies)
- [ ] Run tests in CI on every push/PR

---

## 8. Documentation

```python
def calculate_discount(price: float, discount_rate: float) -> float:
    """Calculate the discounted price.

    Args:
        price: Original price, must be non-negative.
        discount_rate: Fraction between 0 and 1.

    Returns:
        The price after discount is applied.

    Raises:
        ValueError: If price is negative or discount_rate is out of [0, 1].
    """
    if price < 0:
        raise ValueError("price cannot be negative")
    if not 0 <= discount_rate <= 1:
        raise ValueError("discount_rate must be between 0 and 1")
    return price * (1 - discount_rate)
```

- [ ] Docstrings on public functions/classes (Google, NumPy, or reST style — pick one, be consistent)
- [ ] A good `README.md`: what it does, how to install, how to run, how to test
- [ ] Inline comments explain **why**, not **what** (the code already says what)

---

## 9. Dependency Management

- [ ] Use **Poetry**, **uv**, or `pip-tools` — not loose `pip install` with no lockfile
- [ ] Pin versions in production (`requirements.txt` with exact versions or a lockfile)
- [ ] Separate dev dependencies (pytest, black) from runtime dependencies
- [ ] Regularly audit dependencies for vulnerabilities (`pip-audit`, `safety`, Dependabot)

---

## 10. Security Basics

- [ ] Never commit secrets — use `git-secrets` or pre-commit hooks to catch them
- [ ] Validate and sanitize all external input
- [ ] Use parameterized queries — never string-format SQL
- [ ] Keep dependencies updated (known CVEs are a common attack vector)
- [ ] Principle of least privilege for DB/API credentials

```python
# Bad — SQL injection risk
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Good
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

---

## 11. Performance & Efficiency

- [ ] Profile before optimizing (`cProfile`, `py-spy`, `line_profiler`) — don't guess
- [ ] Use built-in data structures wisely (`set` for membership tests, `dict` for lookups over `list`)
- [ ] Use generators for large datasets instead of loading everything into memory
- [ ] Vectorize with NumPy/Pandas instead of Python loops for numeric work
- [ ] Cache expensive, repeated computations (`functools.lru_cache`)
- [ ] Use `asyncio` or threading/multiprocessing appropriately for I/O-bound vs CPU-bound work

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_computation(x: int) -> int:
    ...
```

---

## 12. Version Control Discipline

- [ ] Small, focused commits with clear messages (`fix: handle empty response from API`, not `updates`)
- [ ] Feature branches + Pull Requests, even solo — PRs are a chance to self-review
- [ ] `.gitignore` covers `__pycache__/`, `.env`, `*.pyc`, virtual envs
- [ ] Use conventional commits if working in a team (`feat:`, `fix:`, `docs:`, `refactor:`)

---

## 13. CI/CD Basics

A minimal GitHub Actions pipeline:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src
```

- [ ] Lint + type-check + test run automatically on every push
- [ ] Block merges if CI fails
- [ ] Automate formatting checks (fail if `black --check` fails)

---

## 14. Pre-commit Hooks (Catch Issues Before They're Committed)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks: [{id: black}]
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks: [{id: ruff}]
```

Run `pre-commit install` once, and every commit is auto-checked.

---

## 15. Design Principles Worth Internalizing

- **SOLID** principles (especially Single Responsibility — one function/class does one thing well)
- **DRY** (Don't Repeat Yourself) — but don't over-abstract prematurely either
- **YAGNI** (You Aren't Gonna Need It) — don't build flexibility you don't need yet
- **Composition over inheritance** — favor small, composable functions/classes
- **Pure functions where possible** — same input → same output, no hidden side effects (easier to test and reason about)

---

## 16. Pre-Deployment Checklist

Before shipping anything to production:

- [ ] All tests pass in CI
- [ ] Linting and type checks pass
- [ ] No secrets in code or logs
- [ ] Error handling covers realistic failure modes (network timeout, bad input, service down)
- [ ] Logging is in place for debugging production issues
- [ ] Config is externalized (no hardcoded environment-specific values)
- [ ] Dependencies are pinned/locked
- [ ] README/docs are up to date
- [ ] Rollback plan exists if deployment fails
- [ ] Monitoring/alerting is set up (even basic uptime + error rate)

---

## 17. Tools Summary (Recommended Stack)

| Purpose | Tool |
|---|---|
| Formatting | Black |
| Import sorting | isort (or Ruff's built-in) |
| Linting | Ruff |
| Type checking | mypy or pyright |
| Testing | pytest + pytest-cov |
| Dependency mgmt | Poetry or uv |
| Pre-commit checks | pre-commit |
| CI/CD | GitHub Actions / GitLab CI |
| Logging | stdlib `logging` (+ structlog for structured logs) |
| Config | pydantic-settings / python-dotenv |
| Security scanning | pip-audit / bandit |

---

## 18. How to Build the Habit (Practical Path)

1. **Start every new script** using the project structure above, even for small tasks — repetition builds muscle memory.
2. **Add type hints** to everything you write for a month — it forces clearer thinking about interfaces.
3. **Write the test before or right after the function**, not "later" (later rarely comes).
4. **Set up Ruff + Black + mypy + pre-commit** on your next project and never look back at "no formatting" life.
5. **Read production codebases** (well-regarded open-source Python projects: `requests`, `httpx`, `fastapi`) to see these principles applied.
6. **Do code reviews** — even self-reviews via PRs — reading your own code a day later reveals a lot.

---

### Quick Reference: The 80/20 That Matters Most

If you only adopt a few things, prioritize:
1. Type hints + mypy
2. pytest for core logic
3. Proper logging instead of print
4. Specific exception handling
5. Black + Ruff on autosave

These five alone will make your code noticeably more "production-grade" almost immediately.
