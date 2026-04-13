# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code skills — prompt-based modes that add structured workflows to Claude Code. Each skill is a single `SKILL.md` file in its own directory. There is no build system, no package.json, no runtime code, and no tests — the repo is pure skill definitions.

## Repository Structure

```
skills/
├── spec/SKILL.md           # Specification-driven development (Plan → Review → Execute)
├── mission/SKILL.md        # Multi-phase project orchestration with .missions/ tracking
├── readiness/SKILL.md      # Project readiness auditor (scored report, 0-100)
├── scaffold-tests/SKILL.md # Automated test suite generator (Analyze → Plan → Generate)
├── sherlock/SKILL.md       # Autonomous performance & reliability doctor (investigate → fix → verify loop)
```

## Skill File Format

Every skill is a Markdown file with YAML frontmatter:

```yaml
---
name: <skill-name>
description: "<short> + <longer description>"
argument-hint: "[usage hint shown to user]"
allowed-tools: ["Read", "Write", ...]
---
```

The body defines behavior instructions, output formats, and rules that Claude follows when the skill is invoked.

## Cross-Cutting Patterns

- **spec** and **mission** both enforce mandatory test gates: tests must pass after every step/phase, new exports need tests, bug fixes need regression tests, linter must be clean. These gates reference `CLAUDE.md` in the target project for test commands.
- **scaffold-tests** generates tests that **spec** and **mission** enforce. The mission skill explicitly suggests running `/scaffold-tests` for new modules.
- **sherlock** is the post-build quality pass — after scaffold-tests generates tests and spec/mission enforce them, sherlock autonomously finds and fixes flaky tests, slow tests, performance issues, and reliability problems. Runs in a loop until interrupted.
- **readiness** audits the same project configuration that the other skills depend on (CLAUDE.md, test setup, CI, type safety).
- All multi-step skills use explicit checkpoints with user confirmation between phases. Exception: **sherlock** is fully autonomous — it never pauses to ask, by design.

## Conventions When Editing Skills

- Keep each skill self-contained in its own `SKILL.md` — skills should not import or reference each other's files (cross-references by name like "run `/scaffold-tests`" are fine).
- Preserve the three/two-phase structure (Plan → Review → Execute, or Analyze → Plan → Generate).
- Output format blocks (the ASCII box/table templates) are part of the skill's UX — keep them consistent.
- Rules sections at the bottom of each skill are behavioral constraints — changes there affect how Claude behaves during execution.
- `allowed-tools` in frontmatter controls which tools the skill can access — keep this minimal.
