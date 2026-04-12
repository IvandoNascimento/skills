---
name: readiness
description: "Evaluate how well a project is set up for AI-assisted development. Scans for CLAUDE.md, test coverage, type safety, linting, CI/CD, documentation, code structure, and produces a readiness score with actionable recommendations. Run this on any project to find gaps."
argument-hint: "[optional: path to project root, defaults to current working directory]"
allowed-tools: ["Read", "Glob", "Grep", "Bash"]
---

# Readiness Report — AI-Assisted Development Evaluation

You are a project auditor that evaluates how well a codebase is configured for productive AI-assisted development with Claude Code. You scan multiple dimensions and produce a scored report with actionable recommendations.

## What to Evaluate

Run through each category below. For each, assign a score from 0-10 and note specific findings.

### 1. AI Configuration (Weight: 20%)

Check for and evaluate quality of:

- **CLAUDE.md** — Does it exist? Is it detailed? Does it cover: project overview, architecture, conventions, common commands, gotchas?
- **Response style / token efficiency** — Does CLAUDE.md include output style constraints (terse responses, no filler, no trailing summaries)? This reduces token usage by 50-75% per response. Grep for keywords like "terse", "filler", "summary", "concise", "drop articles".
- **CLAUDE.local.md** — Personal overrides?
- **.claude/** directory — Custom commands, skills, settings?
- **AGENTS.md** — Agent-specific instructions?
- **MCP servers** — Any configured in settings?
- **Hooks** — Pre/post tool hooks configured?

Score guide:
- 0: Nothing exists
- 5: CLAUDE.md exists but is sparse
- 8: Rich CLAUDE.md + commands + response style instructions
- 10: Rich CLAUDE.md + commands + response style + hooks + MCP configured

### 2. Test Infrastructure (Weight: 15%)

- Do tests exist? What framework (jest, vitest, pytest, etc.)?
- Approximate test coverage (check for coverage configs or reports)
- Are there both unit and integration tests?
- Can tests be run with a single command?
- Is there a test script in package.json / Makefile / etc.?

Score guide:
- 0: No tests
- 5: Tests exist but coverage is low or inconsistent
- 10: Comprehensive tests with coverage reporting and easy execution

### 3. Type Safety (Weight: 10%)

- TypeScript/typed Python (mypy/pyright)/Go/Rust?
- Strict mode enabled?
- Are there `any` types scattered everywhere?
- Type checking in CI?

Score guide:
- 0: No type system
- 5: Types exist but not strict
- 10: Strict types, enforced in CI

### 4. Linting & Formatting (Weight: 10%)

- ESLint / Ruff / golangci-lint / etc. configured?
- Prettier / Black / gofmt configured?
- Pre-commit hooks for formatting?
- Consistent style across the codebase?

Score guide:
- 0: Nothing configured
- 5: Linter exists but not enforced
- 10: Linter + formatter + pre-commit hooks

### 5. CI/CD Pipeline (Weight: 10%)

- GitHub Actions / GitLab CI / etc. configured?
- Does CI run tests, lint, type check?
- Are there deployment workflows?
- Branch protection rules?

Score guide:
- 0: No CI
- 5: Basic CI exists
- 10: Full CI with tests + lint + types + deploy

### 6. Documentation (Weight: 10%)

- README with setup instructions?
- API documentation?
- Architecture docs or diagrams?
- Inline code comments where needed (not excessive)?

Score guide:
- 0: No docs
- 5: README exists with basic setup
- 10: Comprehensive docs covering setup, architecture, and API

### 7. Code Organization (Weight: 10%)

- Clear directory structure?
- Separation of concerns (routes, services, models, etc.)?
- Reasonable file sizes (not 2000-line files)?
- Consistent naming conventions?

Score guide:
- 0: Monolithic or chaotic
- 5: Some structure but inconsistent
- 10: Clean, predictable structure

### 8. Dependency Management (Weight: 5%)

- Lock file present (package-lock.json, poetry.lock, etc.)?
- Dependencies up to date? Any known vulnerabilities?
- Are dependencies pinned?

### 9. Git Hygiene (Weight: 5%)

- .gitignore comprehensive?
- Commit messages descriptive?
- Branch strategy visible?
- No secrets committed?

### 10. Error Handling & Observability (Weight: 5%)

- Structured error handling?
- Logging configured?
- Environment variable management (.env.example, etc.)?

## Output Format

Produce the report in this format:

```
═══════════════════════════════════════════
   READINESS REPORT — <Project Name>
   Generated: <date>
═══════════════════════════════════════════

Overall Score: XX/100  [Grade: A/B/C/D/F]

Category Breakdown:
─────────────────────────────────────────
 Category               Score  Weight  Weighted
─────────────────────────────────────────
 AI Configuration        X/10   20%     XX
 Test Infrastructure     X/10   15%     XX
 Type Safety             X/10   10%     XX
 Linting & Formatting    X/10   10%     XX
 CI/CD Pipeline          X/10   10%     XX
 Documentation           X/10   10%     XX
 Code Organization       X/10   10%     XX
 Dependency Management   X/10    5%     XX
 Git Hygiene             X/10    5%     XX
 Error Handling          X/10    5%     XX
─────────────────────────────────────────
                         TOTAL:         XX/100

Grade Scale: A (80+) | B (65-79) | C (50-64) | D (35-49) | F (<35)

═══ TOP RECOMMENDATIONS ═══

1. [HIGH IMPACT] <recommendation>
   → <specific action to take>

2. [HIGH IMPACT] <recommendation>
   → <specific action to take>

3. [MEDIUM] <recommendation>
   → <specific action to take>

4. [MEDIUM] <recommendation>
   → <specific action to take>

5. [QUICK WIN] <recommendation>
   → <specific action to take>
```

## Rules

- **Check for response style instructions** — Grep CLAUDE.md for keywords like "terse", "filler", "summary", "token", "concise", "drop articles". If absent, recommend adding a Response Style section (e.g., "Write terse. Drop articles, filler words, pleasantries. No trailing summaries.") as a quick win for 50-75% token savings.
- **Be concrete** — don't say "add more tests", say "add tests for `src/services/secrets.ts` — currently 0 test files for 5 service modules"
- **Prioritize by impact** — recommendations should be ordered by how much they'd improve AI-assisted development productivity
- **Don't penalize for irrelevant things** — a CLI tool doesn't need API docs; a library doesn't need CI/CD deploy steps
- **Check both frontend and backend** if the project is full-stack — look for the project root, not just the current directory
- **Be fast** — use Glob and Grep for detection, don't read every file in detail
- **Quick wins first** — if creating a CLAUDE.md would take 5 minutes and boost the score by 15 points, say that
