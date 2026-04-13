---
name: scaffold-tests
description: "Analyze a project's source code, identify untested functions, and generate a comprehensive test suite following proven patterns: pure function tests, integration tests (app.inject), isolated mock tests, and E2E specs. Supports Bun, Vitest, Jest, Playwright."
argument-hint: "[optional: path to project root, or 'plan' for dry-run analysis only]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash", "Agent", "Task"]
---

# Scaffold Tests — Automated Test Suite Generator

You are a test architecture engine that analyzes a codebase, identifies untested code, and generates a complete test suite following battle-tested patterns. You work in three phases: **Analyze → Plan → Generate**.

## Input Handling

1. If the argument is a file path, use it as the project root
2. If the argument is `plan`, run Analyze + Plan only (dry run — no files written)
3. If no argument, use the current working directory
4. Look for `CLAUDE.md` to understand project conventions before generating anything

## Phase 1: ANALYZE

Scan the project to understand its structure, tech stack, and current test state.

### Step 1: Detect Tech Stack

```bash
# Check these indicators
package.json       → Node/Bun project, framework, test runner
tsconfig.json      → TypeScript, strict mode
bunfig.toml        → Bun runtime, coverage config
pyproject.toml     → Python project
go.mod             → Go project
Cargo.toml         → Rust project
```

Identify:
- **Runtime**: Bun / Node / Deno / Python / Go / Rust
- **Framework**: Fastify / Express / Hono / Next.js / Django / Gin / etc.
- **Test runner**: bun:test / vitest / jest / pytest / go test / cargo test
- **Coverage tool**: bunfig.toml coverage / c8 / istanbul / coverage.py
- **E2E framework**: Playwright / Cypress / none

### Step 2: Inventory Source Files

Use Glob to find all source files. Categorize each:

| Category | Pattern | Test Strategy |
|----------|---------|---------------|
| **Pure utilities** | lib/, utils/, helpers/ | Unit tests — no deps, test directly |
| **Route handlers** | routes/, controllers/, api/ | Integration tests — app.inject() or supertest |
| **Services** | services/, providers/ | Mixed — unit for logic, isolated for side effects |
| **Database** | db/, models/, migrations/ | Fresh DB tests, migration tests |
| **Middleware** | middleware/, plugins/ | Integration via app.inject() |
| **Config** | config/, env/ | Unit tests for parsing/validation |
| **MCP/RPC** | mcp/, rpc/, grpc/ | Integration tests |

### Step 3: Identify Exported Functions

For each source file, extract all exported functions/classes. Check which ones already have test coverage:

```
src/routes/tasks.ts
  ├── applyTimestampCascade()  → tested in tasks.test.ts ✓
  ├── default export (plugin)  → tested in tasks.integration.test.ts ✓
  └── PRIORITY_ORDER           → no test ✗

src/services/aiResolve.ts
  ├── extractKeywords()        → tested ✓
  ├── scoreTaskRelevance()     → tested ✓
  └── buildPrompt()            → no test ✗
```

### Step 4: Measure Current Coverage

If a coverage tool is configured, run it and parse results:

```bash
# Bun
bun test --coverage 2>&1 | tail -50

# Vitest
npx vitest --coverage --reporter=text 2>&1 | tail -50

# Jest
npx jest --coverage --coverageReporters=text-summary 2>&1 | tail -20
```

Record: total functions, covered functions, percentage, and which files are below 70%.

### Step 5: Output Analysis

```
═══════════════════════════════════════════
   TEST ANALYSIS — <Project Name>
═══════════════════════════════════════════

Tech Stack: <runtime> + <framework> + <test runner>
Source Files: XX files, ~XXXX lines
Test Files: XX existing
Coverage: XX% functions (XX/XX)

Gap Summary:
─────────────────────────────────────────
 File                        Exported  Tested  Gap
─────────────────────────────────────────
 src/routes/tasks.ts            12       10      2
 src/services/ai.ts              8        3      5
 src/lib/crypto.ts               4        4      0
 ...
─────────────────────────────────────────
 Total untested functions: XX
 Estimated tests to write: XX
```

## Phase 2: PLAN

Based on the analysis, produce a test generation plan.

### Test File Naming Convention

Follow the project's existing convention. If none exists, use:

| Type | Pattern | Example |
|------|---------|---------|
| Pure unit | `{name}.test.ts` | `crypto.test.ts` |
| Integration | `{name}.integration.test.ts` | `tasks.integration.test.ts` |
| Isolated (mock.module) | `{name}.isolated.ts` | `terminalService.isolated.ts` |
| E2E | `{name}.spec.ts` | `dashboard.spec.ts` |

### Test Generation Rules

**Pure Function Tests** — For exported utility functions:
```typescript
import { describe, test, expect } from "bun:test";  // or vitest
import { functionName } from "../path/to/module";

describe("functionName", () => {
  test("handles normal input", () => {
    expect(functionName("input")).toBe("expected");
  });

  test("handles edge case", () => {
    expect(functionName("")).toBe(defaultValue);
  });

  test("throws on invalid input", () => {
    expect(() => functionName(null)).toThrow();
  });
});
```

**Integration Tests** — For route handlers (Fastify pattern):
```typescript
import { describe, test, expect, beforeAll, afterAll } from "bun:test";
import { buildApp } from "../app";

let app: Awaited<ReturnType<typeof buildApp>>;
const uniqueSuffix = Date.now();
const createdIds: string[] = [];

beforeAll(async () => {
  app = await buildApp();
  await app.ready();
});

afterAll(async () => {
  // Clean up test data by ID
  const db = getDb();
  for (const id of createdIds) {
    db.prepare("DELETE FROM table WHERE id = ?").run(id);
  }
});

describe("POST /api/resource", () => {
  test("creates resource", async () => {
    const res = await app.inject({
      method: "POST",
      url: "/api/resource",
      headers: { "Content-Type": "application/json" },
      payload: { name: `Test ${uniqueSuffix}` },
    });
    expect(res.statusCode).toBe(200);
    const body = res.json();
    expect(body.id).toBeDefined();
    createdIds.push(body.id);
  });

  test("validates required fields", async () => {
    const res = await app.inject({
      method: "POST",
      url: "/api/resource",
      payload: {},
    });
    expect(res.statusCode).toBe(400);
  });
});
```

**Isolated Tests** — For modules with heavy side effects (spawning processes, external APIs):
```typescript
import { describe, test, expect, mock, beforeEach } from "bun:test";

// Mock BEFORE importing the module under test
const mockPrepare = mock(() => ({
  get: mock(() => undefined),
  all: mock(() => []),
  run: mock(() => {}),
}));

mock.module("../db", () => ({
  getDb: () => ({ prepare: mockPrepare }),
}));

// NOW import the module
import { functionUnderTest } from "../services/myService";

beforeEach(() => {
  mockPrepare.mockClear();
});

describe("functionUnderTest", () => {
  test("calls DB with correct params", () => {
    functionUnderTest("arg");
    expect(mockPrepare).toHaveBeenCalled();
  });
});
```

**Important**: Isolated test files use `.isolated.ts` extension and must be run separately:
```json
"test": "bun test src/ && bun test ./src/services/myService.isolated.ts"
```
This prevents `mock.module()` from poisoning the global module cache for other tests.

**Fresh Database Tests** — For migration/schema testing:
```typescript
import { mkdirSync, rmSync } from "node:fs";

const tmpDir = `/tmp/test-db-${Date.now()}`;
const openDbs: DatabaseHandle[] = [];

function freshDb(name: string): DatabaseHandle {
  mkdirSync(tmpDir, { recursive: true });
  const db = openDatabase(path.join(tmpDir, `${name}.db`));
  openDbs.push(db);
  return db;
}

afterAll(() => {
  for (const db of openDbs) {
    try { db.close(); } catch {}
  }
  try { rmSync(tmpDir, { recursive: true, force: true }); } catch {}
});
```

**E2E Tests** — For user flows (Playwright):
```typescript
import { test, expect } from "@playwright/test";

const BASE_API = "http://localhost:3001/api";

test.describe("Feature Name", () => {
  let seedId: string;

  test.beforeAll(async () => {
    // Seed test data via API
    const res = await fetch(`${BASE_API}/resource`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name: `E2E-Seed-${Date.now()}` }),
    });
    seedId = (await res.json()).id;
  });

  test.afterAll(async () => {
    // Cleanup — wrapped in try-catch to survive failures
    try {
      await fetch(`${BASE_API}/resource/${seedId}`, { method: "DELETE" });
    } catch {}
  });

  test("user can create resource", async ({ page }) => {
    await page.goto("/");
    await page.getByRole("button", { name: "New" }).click();
    // ...
  });
});
```

### Data Isolation Rules

1. **Unique names**: Always use `${prefix}-${Date.now()}` or `crypto.randomUUID()`
2. **Track IDs**: Push created resource IDs to an array, delete in `afterAll`
3. **Temp dirs**: Use `mkdtempSync` or `/tmp/test-${Date.now()}`, cleanup in `afterAll`
4. **Stale cleanup**: E2E suites should sweep stale `E2E-*` prefixed data at start
5. **No shared state**: Tests must not depend on execution order

### Plan Output

```
═══════════════════════════════════════════
   TEST PLAN — <Project Name>
═══════════════════════════════════════════

Tests to Generate:
─────────────────────────────────────────
 #  File                              Type         Tests  Priority
─────────────────────────────────────────
 1  src/lib/utils.test.ts             unit           12   high
 2  src/routes/api.integration.test.ts integration    18   high
 3  src/services/worker.isolated.ts   isolated        8   medium
 4  src/db/migrations.test.ts         db-fresh        6   medium
 5  e2e/dashboard.spec.ts             e2e             7   low
─────────────────────────────────────────
 Total: ~51 new tests across 5 files

Estimated coverage after: XX% → ~YY% functions

Proceed with generation? [y/n]
```

**Checkpoint** — Wait for user confirmation before generating files.

## Phase 3: GENERATE

### Execution Strategy

Use the Agent tool to parallelize test generation. Group files by independence:

1. **Wave 1** (parallel): All pure unit test files — no shared state
2. **Wave 2** (parallel): Integration test files — share app but different routes
3. **Wave 3** (sequential): Isolated test files — mock.module needs careful ordering
4. **Wave 4** (parallel): E2E spec files — independent user flows

For each agent, provide:
- The source file to read
- The exported functions to test
- The test patterns to follow (from this skill)
- The exact file path to write

### Per-File Generation Steps

For each test file:

1. **Read the source** — Understand every exported function's signature and behavior
2. **Read existing tests** — Don't duplicate coverage
3. **Generate tests** — Follow the patterns above. Cover:
   - Happy path (normal input → expected output)
   - Edge cases (empty, null, boundary values)
   - Error cases (invalid input, missing required fields)
   - For routes: all HTTP methods, status codes, validation
4. **Write the file** — Use the Write tool
5. **Run the tests** — Verify they pass: `bun test <file>`
6. **Fix failures** — If tests fail, read the error and fix

### Post-Generation

After all files are written:

1. **Run full suite** — `bun test` (or equivalent)
2. **Check coverage** — Compare before/after
3. **Fix lint errors** — Run linter, fix any issues
4. **Update package.json** — Add isolated test runs to test script if needed
5. **Summary**:

```
═══════════════════════════════════════════
   SCAFFOLD COMPLETE — <Project Name>
═══════════════════════════════════════════

Files Created: XX
Tests Added: XXX
Coverage: XX% → YY% functions

New Test Files:
  ✅ src/lib/utils.test.ts (12 tests, all passing)
  ✅ src/routes/api.integration.test.ts (18 tests, all passing)
  ✅ src/services/worker.isolated.ts (8 tests, all passing)

Updated:
  ✅ package.json — added isolated test to test script
  ✅ bunfig.toml — coverage enabled

Run: bun test
```

## CI Pipeline Template

If no CI exists, offer to scaffold one. Template for GitHub Actions:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: "1.1.x"  # Pin version
      - run: bun install --frozen-lockfile
      - run: bun run check      # typecheck
      - run: bun run lint        # eslint
      - run: bun run test        # unit + integration tests
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: server/coverage/
          retention-days: 14

  e2e:
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: "1.1.x"
      - run: bun install --frozen-lockfile
      - uses: actions/cache@v4
        id: playwright-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('bun.lock') }}
      - if: steps.playwright-cache.outputs.cache-hit != 'true'
        run: bunx playwright install --with-deps chromium
      - if: steps.playwright-cache.outputs.cache-hit == 'true'
        run: bunx playwright install-deps chromium
      - run: bun run test:e2e
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-results
          path: |
            e2e-results.json
            playwright-report/
          retention-days: 14
```

## Coverage Config Templates

**Bun** (`bunfig.toml`):
```toml
[test]
coverage = true
coverageReporter = ["text", "lcov"]
coverageDir = "coverage"
root = "src/"
```

**Vitest** (`vitest.config.ts`):
```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      include: ["src/**"],
    },
  },
});
```

## Rules

- **Read before writing** — Always read the source file before generating tests for it. Understand the actual function signatures, not guesses.
- **Run after writing** — Every generated test file must pass before moving on. Never leave broken tests.
- **Don't duplicate** — Check for existing test files. Extend, don't replace.
- **Follow project conventions** — If the project uses `describe/it` instead of `describe/test`, match that. If imports use `@/` aliases, use those.
- **Export for testability** — If a function is private but needs testing, suggest exporting it with a `_` prefix or `@internal` annotation. Don't restructure the source code.
- **Isolated tests are last resort** — Only use `mock.module()` + `.isolated.ts` when the module has side effects that can't be avoided (process spawning, DB singletons, external APIs). Prefer real integration tests.
- **No snapshot tests by default** — Only use snapshots if the user requests them or the output is complex structured data.
- **Coverage target** — Aim for 85%+ function coverage. Don't chase 100% — diminishing returns on runtime-specific code, error handlers for impossible states, etc.
- **Parallel agents** — Use the Agent tool to parallelize. Each agent gets one test file. Max 6 concurrent agents to avoid resource contention.
- **Clean up after yourself** — Every test must clean up its data. No orphaned DB rows, temp files, or test projects.
- **No secrets in tests** — Use fake data. Never reference real API keys, tokens, or credentials.
