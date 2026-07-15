# Writing Production-Level TypeScript: A Practical Guide

## 1. Mindset Shift

| Quick Code | Production Code |
|---|---|
| `any` everywhere | Precise types, `unknown` at boundaries |
| `console.log` debugging | Structured logging |
| Hardcoded config | Env vars / typed config, validated |
| No tests | Unit + integration tests |
| Loose `tsconfig.json` | Strict mode fully enabled |
| "It compiles" | "It compiles under `strict`, is tested, and the types actually mean something" |

TypeScript's biggest production risk is **fake safety** — types that compile but don't reflect runtime reality (e.g., trusting an API response's shape without validating it). Production discipline is largely about closing that gap.

---

## 2. Project Structure

```
my-service/
├── src/
│   ├── index.ts            # thin entrypoint
│   ├── config/
│   ├── routes/ (or controllers/)
│   ├── services/
│   ├── repositories/
│   ├── models/ (or types/)
│   ├── middleware/
│   └── utils/
├── tests/
│   ├── unit/
│   └── integration/
├── .env.example
├── .eslintrc.json (or eslint.config.js)
├── .prettierrc
├── tsconfig.json
├── package.json
└── README.md
```

**Tips:**
- Keep `index.ts`/`main.ts` minimal — wiring only. Business logic in testable modules.
- Colocate types with the module that owns them (`services/order.types.ts`) rather than one giant `types.ts` dumping ground, for anything beyond a small project.
- Use path aliases (`@/services/*`) via `tsconfig.json` `paths` to avoid `../../../` import chains.

---

## 3. `tsconfig.json` — Get Strictness Right From Day One

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

- [ ] `"strict": true` — non-negotiable for production code; it bundles `noImplicitAny`, `strictNullChecks`, etc.
- [ ] `"noUncheckedIndexedAccess": true` — makes array/object index access return `T | undefined`, catching a huge class of runtime bugs at compile time
- [ ] Retrofitting strict mode onto a loose codebase later is painful — turn it on from day one

---

## 4. Code Quality Checklist

- [ ] **ESLint** with `@typescript-eslint` rules, plus `eslint-plugin-import` for import hygiene
- [ ] **Prettier** for formatting — don't hand-format, don't debate style in review
- [ ] Avoid `any` — if the type is genuinely unknown, use `unknown` and narrow it before use
- [ ] Avoid non-null assertions (`!`) except where you can prove safety in a comment — they're a common source of production `undefined` crashes
- [ ] Prefer `type` aliases and `interface` deliberately: `interface` for object shapes that might be extended, `type` for unions/intersections/utility compositions

```typescript
// Bad
function processOrder(order: any) {
  return order.total * 1.1;
}

// Good
interface Order {
  id: string;
  total: number;
  items: OrderItem[];
}

function processOrder(order: Order): number {
  return order.total * 1.1;
}
```

---

## 5. Runtime Validation (The TypeScript-Specific Gap)

Types disappear at runtime — anything crossing a boundary (API request, DB response, env var, third-party API) must be validated, not just typed.

```typescript
import { z } from "zod";

const OrderSchema = z.object({
  id: z.string().uuid(),
  total: z.number().nonnegative(),
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive(),
  })),
});

type Order = z.infer<typeof OrderSchema>;

function parseOrder(raw: unknown): Order {
  return OrderSchema.parse(raw); // throws ZodError with details if invalid
}
```

**Checklist:**
- [ ] Use **Zod**, **Valibot**, or **io-ts** to validate anything entering the system from outside (HTTP requests, external API responses, env vars, file/DB reads)
- [ ] Infer TypeScript types from the schema (`z.infer<typeof Schema>`) so validation and typing never drift apart
- [ ] Never `as` cast an external response into a type without validating it first — that's the single most common source of "it typechecked but crashed in production" bugs
- [ ] Validate at the boundary once, then trust the type internally

---

## 6. Error Handling

```typescript
// Custom error hierarchy
class AppError extends Error {
  constructor(message: string, public readonly statusCode: number) {
    super(message);
    this.name = this.constructor.name;
  }
}

class OrderNotFoundError extends AppError {
  constructor(orderId: string) {
    super(`Order not found: ${orderId}`, 404);
  }
}

async function getOrder(orderId: string): Promise<Order> {
  const order = await db.orders.findById(orderId);
  if (!order) {
    throw new OrderNotFoundError(orderId);
  }
  return order;
}

// Centralized handling (Express example)
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    logger.warn(err.message, { statusCode: err.statusCode });
    return res.status(err.statusCode).json({ error: err.message });
  }
  logger.error("Unhandled error", { error: err });
  res.status(500).json({ error: "Internal server error" });
});
```

**Best practices:**
- [ ] Build a small custom error class hierarchy rather than throwing raw strings or generic `Error`
- [ ] Use a centralized error-handling middleware (Express/Fastify/Koa) instead of try/catch sprawled across every route
- [ ] Always `await` promises or explicitly `.catch()` them — an unhandled promise rejection can crash a Node process
- [ ] In `catch` blocks, remember the caught value's type is `unknown` in strict TS — narrow before using it (`if (err instanceof Error)`)
- [ ] Never swallow errors silently (`catch {}` with nothing inside)

```typescript
try {
  await processPayment(order);
} catch (err) {
  if (err instanceof PaymentGatewayError) {
    logger.error("Payment gateway failure", { orderId: order.id, error: err.message });
    throw new AppError("Payment could not be processed", 502);
  }
  throw err; // re-throw unexpected errors, don't mask them
}
```

---

## 7. Logging

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  formatters: { level: (label) => ({ level: label }) },
});

logger.info({ orderId, userId }, "Processing order");
logger.error({ orderId, err }, "Payment failed");
```

**Checklist:**
- [ ] Use a structured logger (**Pino**, **Winston**) — never bare `console.log` in production code
- [ ] JSON output in production (machine-parseable for log aggregators), pretty-printed in dev
- [ ] Include request/correlation IDs (via `AsyncLocalStorage` or middleware) to trace a request across logs
- [ ] Never log secrets, tokens, full request bodies containing PII
- [ ] Appropriate levels: `debug`, `info`, `warn`, `error`, `fatal`

---

## 8. Configuration Management

```typescript
import { z } from "zod";

const EnvSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "staging", "production"]),
});

export const env = EnvSchema.parse(process.env); // fails fast at startup if misconfigured
```

- [ ] Validate `process.env` at startup with a schema (Zod) — catch misconfiguration immediately, not mid-request
- [ ] Never hardcode secrets/URLs; `.env` for local dev only (gitignored), real secrets via a secrets manager in deployed environments
- [ ] Separate config per environment (`NODE_ENV`-driven or explicit config files)
- [ ] Type the validated config once and import it everywhere — no scattered `process.env.X` reads with no validation

---

## 9. Testing

```typescript
import { describe, it, expect, vi } from "vitest";

describe("calculateDiscount", () => {
  it("applies a percentage discount", () => {
    expect(calculateDiscount(100, 0.1)).toBe(90);
  });

  it("throws on negative price", () => {
    expect(() => calculateDiscount(-10, 0.1)).toThrow();
  });

  it.each([
    [100, 0.0, 100],
    [100, 1.0, 0],
    [50, 0.5, 25],
  ])("calculateDiscount(%d, %d) = %d", (price, rate, expected) => {
    expect(calculateDiscount(price, rate)).toBe(expected);
  });
});

// Mocking a dependency
vi.mock("../services/paymentGateway", () => ({
  charge: vi.fn().mockResolvedValue({ success: true }),
}));
```

**Checklist:**
- [ ] **Vitest** (fast, modern, Jest-compatible API) or **Jest** as the test runner
- [ ] Unit test pure business logic; mock I/O boundaries (DB, HTTP, filesystem)
- [ ] Integration tests against a real (containerized) database using **Testcontainers** for Node, rather than mocking the DB entirely
- [ ] For HTTP APIs, use **supertest** (Express/Fastify) to test routes end-to-end within the test process
- [ ] Aim for meaningful coverage (70–90%) on business logic — check with `vitest --coverage`
- [ ] Run `tsc --noEmit` in CI as a separate step from tests — type errors and test failures are different failure modes worth distinguishing

---

## 10. Documentation

```typescript
/**
 * Calculates the discounted price.
 *
 * @param price - Original price, must be non-negative.
 * @param discountRate - Fraction between 0 and 1 inclusive.
 * @returns The price after the discount is applied.
 * @throws {RangeError} If price is negative or discountRate is out of range.
 */
function calculateDiscount(price: number, discountRate: number): number {
  if (price < 0) throw new RangeError("price must be non-negative");
  if (discountRate < 0 || discountRate > 1) {
    throw new RangeError("discountRate must be between 0 and 1");
  }
  return price * (1 - discountRate);
}
```

- [ ] TSDoc/JSDoc comments on exported functions/types — editors surface these automatically on hover, which is a big DX win
- [ ] Well-typed public APIs are partially self-documenting — but still document *why*, constraints, and error conditions
- [ ] `README.md` with setup, scripts (`npm run dev/test/build`), and architecture notes
- [ ] For libraries, consider **TypeDoc** to generate browsable API docs from TSDoc comments

---

## 11. Dependency Management

- [ ] Use `package-lock.json`/`pnpm-lock.yaml`/`yarn.lock` — always commit the lockfile
- [ ] Prefer **pnpm** for monorepos/large projects (disk-efficient, strict dependency resolution catches phantom dependencies)
- [ ] Run `npm audit` / `pnpm audit` regularly, and wire it into CI
- [ ] Keep `devDependencies` separate from runtime `dependencies` — smaller production bundles/images
- [ ] Use `npm ci` (not `npm install`) in CI/CD — it installs exactly what's in the lockfile, no surprises

---

## 12. Security Basics

- [ ] Use parameterized queries / an ORM's query builder (Prisma, Drizzle, Knex) — never string-concatenate SQL
- [ ] Validate all external input with a schema (see Section 5) before it touches business logic
- [ ] Set security headers (`helmet` for Express) and CORS explicitly, don't default to `*`
- [ ] Never trust `req.body`/`req.query` types just because TypeScript says so — they're `any` at the framework boundary until validated
- [ ] Keep dependencies patched — run `npm audit fix` and monitor Dependabot/Snyk alerts
- [ ] Sanitize output where relevant (XSS) if rendering any user-provided content

```typescript
// Bad
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// Good (Prisma example)
const user = await prisma.user.findUnique({ where: { id: userId } });
```

---

## 13. Async & Performance

- [ ] Always `await` or `.catch()` promises — enable `no-floating-promises` in ESLint to catch this automatically
- [ ] Use `Promise.all`/`Promise.allSettled` for concurrent independent async work instead of sequential `await`s in a loop
- [ ] Set timeouts on outbound HTTP calls (`AbortController` with `fetch`, or Axios's `timeout` option) — an unbounded request is a production incident
- [ ] Profile with Node's built-in `--prof`/`clinic.js` before optimizing
- [ ] Watch for event-loop-blocking synchronous work (heavy JSON parsing, crypto, regex on large input) in a single-threaded Node process — offload to worker threads if needed

```typescript
// Bad — sequential, slow
for (const id of ids) {
  results.push(await fetchItem(id));
}

// Good — concurrent
const results = await Promise.all(ids.map(fetchItem));
```

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
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm test -- --coverage
      - run: npm run build
```

- [ ] `.gitignore` covers `node_modules/`, `dist/`, `.env`
- [ ] CI runs lint, typecheck, tests, and build on every push — treat each as a distinct, separately-visible gate
- [ ] Fail the build on ESLint errors and type errors, not only test failures

---

## 15. Design Principles Worth Internalizing

- **Parse, don't validate-and-hope** — turn `unknown` external data into a typed, validated value once at the boundary (see Section 5), and never re-check it deeper in the call stack
- **Discriminated unions** over optional-field soup for representing state

```typescript
// Bad — many invalid combinations are representable
interface RequestState {
  loading: boolean;
  data?: Order;
  error?: string;
}

// Good — illegal states unrepresentable
type RequestState =
  | { status: "loading" }
  | { status: "success"; data: Order }
  | { status: "error"; error: string };
```

- **Favor immutability** — `readonly` properties, `as const`, avoid mutating function parameters
- **Small, composable functions** over large multi-responsibility ones
- **Dependency injection** (even manually, via constructor params) over module-level singletons — makes testing dramatically easier

---

## 16. Pre-Deployment Checklist

- [ ] `tsc --noEmit` passes with `strict: true`
- [ ] ESLint passes with no errors
- [ ] All tests pass, including integration tests against real dependencies
- [ ] No secrets in code, `.env` committed, or logs
- [ ] All external input validated with a schema at the boundary
- [ ] All async operations have error handling (`no-floating-promises` enforced)
- [ ] Structured logging with correlation IDs in place
- [ ] Env vars validated at startup — app fails fast on misconfiguration
- [ ] `npm audit`/Snyk shows no critical vulnerabilities
- [ ] Health check endpoint exists for orchestration
- [ ] Graceful shutdown handled (drain connections on SIGTERM)

---

## 17. Tools Summary

| Purpose | Tool |
|---|---|
| Formatting | Prettier |
| Linting | ESLint + `@typescript-eslint` |
| Runtime validation | Zod (or Valibot, io-ts) |
| Testing | Vitest (or Jest) |
| API testing | supertest |
| Integration testing | Testcontainers |
| Logging | Pino (or Winston) |
| Config validation | Zod against `process.env` |
| ORM/query builder | Prisma, Drizzle, or Knex |
| Security headers | helmet |
| Dependency audit | `npm audit`, Snyk, Dependabot |
| CI/CD | GitHub Actions / GitLab CI |
| Package manager | pnpm (recommended for larger projects) |

---

## 18. How to Build the Habit

1. **Turn on `strict: true` from the very first commit** of every new project — never plan to "add strictness later."
2. **Validate every external input with Zod** before it touches your typed domain logic — make this the default reflex, not an afterthought.
3. **Write the test alongside the function**, using Vitest's fast watch mode to keep the loop tight.
4. **Enable `no-floating-promises` and `no-explicit-any` in ESLint** and treat violations as blocking, not warnings to ignore.
5. **Read well-typed open-source TS codebases** (`zod` itself, `trpc`, `drizzle-orm`) to see advanced type-safety patterns applied well.

---

### Quick Reference: The 80/20 That Matters Most

1. `strict: true` in `tsconfig.json`, no exceptions
2. Zod (or similar) validation at every external boundary — never trust `any`/`unknown` implicitly
3. Vitest tests for business logic + `tsc --noEmit` as a separate CI gate
4. Structured logging (Pino) instead of `console.log`
5. `no-floating-promises` ESLint rule — unhandled rejections are a top cause of Node production crashes
