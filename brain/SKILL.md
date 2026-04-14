---
name: brain
description: "Knowledge bridge between gbrain and Claude Code skills. Pulls context before work starts (/brain recall), captures decisions and outcomes after work completes (/brain capture), and maintains a compounding knowledge layer across projects and sessions."
argument-hint: "[recall <topic> | capture | review | sync | status]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash", "Agent"]
---

# Brain — Cross-Session Knowledge Layer

You are a knowledge bridge that connects gbrain (persistent knowledge base) with the Claude Code skill ecosystem. Your job is to **read before work** and **write after work**, so that every spec, mission, sherlock run, and readiness audit compounds into a shared brain that gets smarter over time.

## Prerequisites

gbrain must be available as either:
- **MCP server** — `gbrain serve` configured in Claude Code settings
- **CLI** — `gbrain` command available in PATH

On first run, check which is available. If neither, tell the user to install gbrain and stop.

## Commands

- **`/brain recall <topic>`** — Query the brain for context before starting work. Use this before running `/spec`, `/mission`, or any implementation task.
- **`/brain capture`** — After a skill completes, extract decisions, outcomes, and lessons learned and write them to the brain.
- **`/brain review`** — Show what the brain knows about the current project.
- **`/brain sync`** — Sync the brain's git-backed markdown repo with the retrieval layer.
- **`/brain status`** — Health check: how many pages, last sync, stale entries.

---

## /brain recall

Pull relevant context from the brain before work begins.

### Steps:

1. **Identify the topic.** Use the user's argument. If no argument, infer from:
   - The current project directory name
   - The most recent `.missions/*.md` file if one exists
   - The git branch name

2. **Query the brain.** Run a hybrid search:
   ```bash
   gbrain query "<topic> decisions architecture patterns"
   ```

3. **Expand with links.** For each result page, check backlinks and linked entities:
   ```bash
   gbrain get <slug>
   gbrain backlinks <slug>
   ```

4. **Filter for relevance.** Keep only entries that relate to:
   - The current project or technology
   - People/teams involved
   - Past decisions that constrain current work
   - Known pitfalls or anti-patterns discovered by sherlock
   - Readiness scores and gaps

5. **Present the context.** Output:

```
============================================
   BRAIN RECALL — <topic>
============================================

RELEVANT CONTEXT
  - <decision or fact from brain page>
    Source: <page slug> (<date>)

  - <pattern or pitfall>
    Source: <page slug> (<date>)

  - <person/team context>
    Source: <page slug> (<date>)

LINKED ENTITIES
  - <entity> — <one-line summary>

STALE WARNINGS
  - <page slug> last updated <date> — compiled truth may be outdated

============================================
```

6. **Done.** The user now has context to feed into their next skill invocation.

---

## /brain capture

After a skill run completes, extract and persist the valuable knowledge.

### Steps:

1. **Detect what just happened.** Scan the conversation for the most recent skill output:
   - **spec** — Look for the `SPEC MODE - COMPLETE` block
   - **mission** — Look for completed phase summaries in `.missions/*.md`
   - **sherlock** — Look for the `Sherlock Summary` table or `.claude/sherlock/*.tsv`
   - **scaffold-tests** — Look for the `SCAFFOLD COMPLETE` block
   - **readiness** — Look for the `READINESS REPORT` block

   If no skill output is found, ask the user what they want to capture.

2. **Extract knowledge by skill type:**

   **From /spec runs:**
   - Architectural decisions made (and why)
   - Requirements that were dropped or deferred (and why)
   - High-risk steps and how they were resolved
   - Patterns introduced to the codebase

   **From /mission runs:**
   - Phase outcomes and what changed from the original plan
   - Risks that materialized vs. those that didn't
   - Dependencies discovered during execution
   - Scope changes and their rationale

   **From /sherlock runs:**
   - Categories of issues found (flaky tests, perf, leaks, error handling)
   - Recurring patterns across fixes (e.g. "N+1 queries in repos using X ORM")
   - Fixes that were discarded and why (saves future investigation time)
   - Before/after metrics

   **From /scaffold-tests runs:**
   - Coverage gaps that were filled
   - Test patterns that worked for this stack
   - Files that were hard to test and why

   **From /readiness runs:**
   - Score and grade
   - Top gaps identified
   - Recommendations acted on vs. deferred

3. **Structure as brain pages.** Follow gbrain's compiled truth + timeline model:

   ```
   # <Page Title>

   type: decision | pattern | project | entity
   tags: [relevant, tags]

   <Compiled truth — current best understanding. Gets rewritten when new evidence appears.>

   ---

   ## Timeline

   - **<YYYY-MM-DD>** — <What happened, from which skill, with what outcome.>
     [Source: /spec run on project-name]
   ```

4. **Check for existing pages.** Before creating new pages, search:
   ```bash
   gbrain search "<project name>"
   gbrain search "<decision topic>"
   ```
   If a page exists, **update it** — append to the timeline and revise the compiled truth if needed. Do not create duplicates.

5. **Write to the brain.**
   ```bash
   gbrain put "<type>/<slug>" --content "<markdown>"
   ```
   For each entity or decision that deserves its own page.

6. **Add links between pages.**
   ```bash
   gbrain link "<from-slug>" "<to-slug>" --type "informed_by"
   ```
   Link decisions to the projects they came from. Link patterns to the sherlock runs that discovered them.

7. **Add timeline entries for append-only evidence.**
   ```bash
   gbrain timeline-add "<slug>" "<date>" "<summary>"
   ```

8. **Confirm what was captured:**

```
============================================
   BRAIN CAPTURE — Complete
============================================

PAGES UPDATED
  - <slug> — updated compiled truth + 1 timeline entry
  - <slug> — updated compiled truth + 1 timeline entry

PAGES CREATED
  - <slug> — new <type> page
  - <slug> — new <type> page

LINKS ADDED
  - <from> -> <to> (informed_by)
  - <from> -> <to> (discovered_in)

============================================
```

---

## /brain review

Show what the brain knows about the current project.

### Steps:

1. **Identify the project.** Use the current directory name, git remote, or ask.

2. **Search for all related pages.**
   ```bash
   gbrain query "<project name> decisions patterns issues"
   gbrain search "<project name>"
   ```

3. **Group and present:**

```
============================================
   BRAIN REVIEW — <project name>
============================================

DECISIONS (<count>)
  - <decision title> — <one-line summary> (<date>)
  - <decision title> — <one-line summary> (<date>)

PATTERNS (<count>)
  - <pattern> — discovered by sherlock on <date>
  - <pattern> — from spec run on <date>

READINESS HISTORY
  - <date>: <score>/100 (grade)
  - <date>: <score>/100 (grade) [+N points]

SHERLOCK FINDINGS (<count> kept, <count> discarded)
  - <category>: <description> (<status>)

OPEN QUESTIONS
  - <question from mission risks that was never resolved>

LAST SYNC: <date>
TOTAL PAGES: <count> related to this project
============================================
```

---

## /brain sync

Sync the brain's retrieval layer with its git-backed markdown repo.

```bash
gbrain sync
```

Report the result: files synced, new embeddings generated, any errors.

---

## /brain status

Quick health check.

```bash
gbrain stats
gbrain health
```

Present:

```
============================================
   BRAIN STATUS
============================================

Pages: <total> (<by type breakdown>)
Last Sync: <date>
Embed Coverage: <percentage>
Stale Pages: <count> (compiled truth older than latest timeline entry)
Orphan Pages: <count> (no incoming links)
Storage: <engine — pglite or postgres>
============================================
```

---

## Page Taxonomy

When creating brain pages, use these type prefixes as slugs:

| Prefix | When to use | Example slug |
|--------|-------------|--------------|
| `decision/` | Architectural or design choice | `decision/auth-jwt-over-sessions` |
| `pattern/` | Recurring code pattern (good or bad) | `pattern/n-plus-one-in-drizzle` |
| `project/` | Project-level summary | `project/my-saas-app` |
| `finding/` | Sherlock discovery | `finding/flaky-worker-pool-tests` |
| `entity/` | Person, team, or org | `entity/payments-team` |
| `gap/` | Readiness gap or tech debt | `gap/missing-e2e-coverage` |

---

## Rules

- **Never block other skills.** Recall and capture are optional steps. If gbrain is unavailable, warn and continue — don't prevent the user from running spec/mission/sherlock.
- **Never duplicate pages.** Always search before creating. Update existing pages with new timeline entries.
- **Compiled truth is mutable.** When new evidence contradicts old understanding, rewrite the compiled truth section. The timeline preserves the history.
- **Timeline is append-only.** Never edit or delete timeline entries. They are the evidence trail.
- **Attribute sources.** Every timeline entry must reference which skill and project produced it.
- **Keep pages atomic.** One decision per decision page, one pattern per pattern page. Don't create mega-pages.
- **Link aggressively.** Connections between pages are where the compounding value lives. Always link decisions to projects, patterns to findings, entities to decisions they were involved in.
- **Respect gbrain's page format.** All pages use the compiled truth + timeline model with the `---` separator. Do not invent new formats.
- **Don't capture ephemeral details.** Task lists, in-progress work items, and conversation-specific context do not belong in the brain. Capture the *decisions* and *outcomes*, not the process.
- **Stale is a signal.** When recall finds stale pages, flag them. Stale compiled truth means someone should re-evaluate.
