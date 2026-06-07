# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code skills — prompt-based modes that add structured workflows to Claude Code. Each skill is a `SKILL.md` file (plus optional supporting files) in its own directory. There is no build system, no package.json, and no tests. One exception to "no runtime code": `_lib/checkpoint.py`, a dependency-free helper the vendored security skills shell out to for crash-safe resumable state.

## Repository Structure

```
skills/
├── spec/SKILL.md           # Specification-driven development (Plan → Review → Execute)
├── mission/SKILL.md        # Multi-phase project orchestration with .missions/ tracking
├── readiness/SKILL.md      # Project readiness auditor (scored report, 0-100)
├── scaffold-tests/SKILL.md # Automated test suite generator (Analyze → Plan → Generate)
├── sherlock/SKILL.md       # Autonomous codebase health agent (investigate → fix → verify loop)
├── design/SKILL.md         # AI-driven UI design system generator (Discover → Generate → Export)
├── brain/SKILL.md          # Knowledge bridge to gbrain (recall → capture → review)
├── threat-model/           # Build THREAT_MODEL.md (bootstrap from code / interview owner)
├── vuln-scan/SKILL.md      # Static security scan → VULN-FINDINGS.json (read-only, language-agnostic)
├── triage/                 # Verify/dedupe/rank findings → TRIAGE.json (N-vote adversarial verify)
├── patch/SKILL.md          # Candidate fixes for verified findings → PATCHES/ (static mode)
├── _lib/checkpoint.py      # Shared resumable-state helper used by threat-model, triage, patch
```

The four security skills (`threat-model`, `vuln-scan`, `triage`, `patch`) are vendored from [anthropics/defending-code-reference-harness](https://github.com/anthropics/defending-code-reference-harness) @ `9e0f6c6` (Apache-2.0, license in `_lib/LICENSE`). They are pure-prompt, read/write-only, and language-agnostic. One local modification: all `checkpoint.py` invocations are patched from the upstream cwd-relative `.claude/skills/_lib/checkpoint.py` to `~/.claude/skills/_lib/checkpoint.py` so the skills work from any project when installed via the `~/.claude/skills` symlinks. Keep other drift from upstream minimal.

## Skill File Format

Every skill is a Markdown file with YAML frontmatter:

```yaml
---
name: <skill-name>
description: "<short> + <longer description>"
argument-hint: "[usage hint shown to user]"
---
```

The body defines behavior instructions, output formats, and rules that Claude follows when the skill is invoked.

## Cross-Cutting Patterns

- **spec** and **mission** both enforce mandatory test gates: tests must pass after every step/phase, new exports need tests, bug fixes need regression tests, linter must be clean. These gates reference `CLAUDE.md` in the target project for test commands.
- **scaffold-tests** generates tests that **spec** and **mission** enforce. The mission skill explicitly suggests running `/scaffold-tests` for new modules.
- **sherlock** is the post-build quality pass — after scaffold-tests generates tests and spec/mission enforce them, sherlock autonomously finds and fixes flaky tests, slow tests, performance issues, and reliability problems. Runs in a loop until interrupted.
- **brain** is the cross-session memory layer — it bridges gbrain (persistent knowledge base) with all other skills. `/brain recall` pulls context before work starts, `/brain capture` writes decisions and outcomes back after any skill completes. Over time, the brain accumulates architectural decisions, recurring patterns, sherlock findings, and readiness history that compound across projects.
- **design** is the upstream visual step — before **spec** builds features or **mission** orchestrates phases, `/design` establishes the visual language (tokens, colors, typography, spacing). It auto-detects the project's frontend stack and exports tokens in the right format (CSS vars, Tailwind config, Sass variables, CSS-in-JS theme, etc.). Every subsequent UI prompt can reference the generated style guide.
- **readiness** audits the same project configuration that the other skills depend on (CLAUDE.md, test setup, CI, type safety) — including a Security Posture category that detects the security skills' artifacts and recommends the next step in the chain.
- The **security chain** runs threat-model → vuln-scan → triage → patch: `/threat-model` writes `THREAT_MODEL.md` (scope + trust boundaries), `/vuln-scan` produces `VULN-FINDINGS.json` scoped by it, `/triage` verifies/dedupes/ranks into `TRIAGE.json`, `/patch` drafts fixes into `PATCHES/`. **scaffold-tests** closes the loop: pointed at a `TRIAGE.json` it enters a security-regression sub-mode and turns verified findings into executable guard tests.
- The **fresh-context verifier** pattern (borrowed from the harness) runs across spec, mission, sherlock, scaffold-tests, and design: the agent that verifies work sees only the artifact (diff, test file, showcase) plus the acceptance criteria — never the producing agent's reasoning. **readiness** applies the related discipline as its evidence-before-score rule: written findings precede every category score.
- All multi-step skills use explicit checkpoints with user confirmation between phases. Exception: **sherlock** is fully autonomous — it never pauses to ask, by design (its verifier subagent is also non-interactive).

## Conventions When Editing Skills

- Keep each skill self-contained in its own directory — skills should not import or reference each other's files (cross-references by name like "run `/scaffold-tests`" are fine). The only shared file is `_lib/checkpoint.py`, used by the vendored security skills.
- Preserve the phase structure each skill uses (Plan → Review → Execute, Analyze → Plan → Generate, etc.).
- Output format blocks (the ASCII box/table templates) are part of the skill's UX — keep them consistent.
- Rules sections at the bottom of each skill are behavioral constraints — changes there affect how Claude behaves during execution.
