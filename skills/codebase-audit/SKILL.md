---
name: codebase-audit
description: >
  Produce a rated, prioritized, written audit of an existing codebase and save it to a
  *_AUDIT.md file. Use when the user asks to "audit", "review the codebase for", "find
  smells / complexity / issues", or "assess quality / security / test coverage / performance"
  of a whole project — NOT a single diff or PR (use a diff-level review skill for that). Takes a LENS
  argument: complexity | quality | security | testing | performance | a11y (default: ask).
  Read-only: it catalogs, rates, and plans — it never edits source.
---

# Codebase Audit

A read-only harness that turns "audit this project" into a consistent, scannable
`*_AUDIT.md` report: findings with `file:line` evidence, a rating, a one-line fix, a
prioritized execution order, and a verification plan.

## When to use / not use

- **Use** for whole-project assessment that produces a _document_: "audit the codebase",
  "find complexity / smells", "quality audit", "where are the security gaps".
- **Don't use** for a single diff/PR (use a diff-level review skill/command if your setup has
  one) or to _apply_ fixes. This skill writes a report and **stops** — it ends by offering to
  fix, never by fixing.

## Step 0 — Pick the lens

If the user didn't say, ask which lens (see Lens Packs below). One report = one lens.
Name the output `COMPLEXITY_AUDIT.md`, `QUALITY_AUDIT.md`, `SECURITY_AUDIT.md`, etc.

## The method (9 steps)

1. **Survey.** Get the shape before diving in — adapt commands to the project's language(s):
   - `git ls-files | grep -vE 'node_modules|vendor|dist|build|\.venv'` for the tree.
   - Line counts of source, sorted desc — the biggest files are usually the hotspots. Match the
     project's source extensions (`ts|tsx|js|py|go|rb|java|kt|cs|rs|php|...`):
     `git ls-files | grep -E '\.(ts|tsx|js|py|go|...)$' | grep -vE 'node_modules|vendor|test|spec|stories' | xargs wc -l | sort -rn | head -40`
   - Read the project's manifest / build config (`package.json`, `pyproject.toml`/`requirements.txt`,
     `go.mod`, `pom.xml`/`build.gradle`, `Cargo.toml`, `*.csproj`, …), type/lint/format config,
     and CI workflows. Identify the stack first; everything below adapts to it.
2. **Check for existing audit docs FIRST.** Glob `*.md` for `AGENTS.md`, `CLAUDE.md`, `DESIGN.md`,
   `ARCHITECTURE.md`, prior `*_AUDIT.md`. Read them. **Cross-reference, never duplicate** —
   if an item is already tracked there, point at it instead of re-listing it. State this in
   the report's intro.
3. **Fan out.** Launch ≤3 `Explore` agents IN PARALLEL (one message), each scoped to a slice
   of the lens's dimensions (see packs). Tell each: read files in full, quote code, give
   `file:line` references, do NOT propose fixes — just catalog. For signal-counting (type
   suppressions, casts, debug prints, `TODO`/`FIXME`, swallowed errors — pick patterns that fit
   the project's language) run targeted `grep` yourself in parallel — and **sanity-check counts**
   (a greedy regex inflates them; verify suspicious numbers with a tighter pattern).
4. **Catalog.** Every finding needs: a `file:line`, a quoted snippet or precise description,
   and _why it matters_ (the failure it invites), not just _what it is_.
5. **Rate.** Use the lens's rating vocabulary (below), consistently. Higher = worse.
6. **Prioritize.** Sort into a summary table by impact × recurrence. Recurrence matters: a
   small smell repeated ×5 outranks a medium one-off.
7. **Propose.** One refined proposal per finding (or per cluster of identical findings), with
   concrete step-by-step fix instructions that reuse existing utilities where they exist.
8. **Plan the order.** Sequence the proposals as independently shippable, behavior-preserving
   steps — easiest/highest-leverage first; note dependencies between them.
9. **Verify.** A section describing how to prove no regression: the project's test/lint/build
   commands, any visual/manual checks the stack supports (e.g. Storybook or visual diffs for UI),
   and any lens-specific reproduction (e.g. for a prod-only bug, how to reproduce it locally).

## Report skeleton (write this to `<LENS>_AUDIT.md`)

```
# <Lens> Audit
**Date / Scope / Method** (one line each)
> Cross-references: which existing docs this complements and does NOT duplicate.

## How to read the ratings   (the rubric — define every scale explicitly)

## Summary table            (sorted by priority; columns vary by lens)

## Findings                 (grouped by priority/severity)
  ### <ID> — <title>   file:line
  Smell · Why it matters · Proposal · Steps · Effort/Rating

## What's genuinely good    (verified strengths — keep the report honest & balanced)

## Suggested execution order (numbered, shippable, dependency-aware)

## Verification             (commands + manual/visual + lens-specific repro)
```

Keep it **scannable**: a reader should get the whole picture from the rubric + table, then
drill into findings. Use IDs (H1, P0-1, …) so findings are referenceable from PRs.

## Lens Packs

Each pack = the dimensions to scan + the rating vocabulary to use.

### complexity / simplification

Dimensions: duplication (copy-pasted state machines, mock/real twins, mobile/desktop twins),
god-functions/components, deep nesting, derived-state-stored-in-state, prop drilling,
parallel lists that must stay in sync, magic numbers/strings, dead code, hidden contracts.
**Rating:** two axes, 1–5 each, higher = worse —
_Complexity_ (structural / effort to untangle) and _Cognitive load_ (cost to a reader).
Add a _Recurrence_ column. Priority = blend, weighted by recurrence.

### quality

Dimensions: type safety (untyped escapes / casts / suppressions, strictness), error handling &
resilience (swallowed errors, missing recovery boundaries, uniform failure handling), testing
(coverage gate, edge/failure paths), security & data hygiene (committed fixtures/PII, secret/token
exposure, input validation, dev-only paths that break in prod), accessibility where there's a UI
(lint enforcement + runtime focus), tooling & CI, dependency health, consistency, documentation.
**Rating:** a per-dimension **scorecard** (A–F) + per-finding **severity** (🔴 High / 🟠 Medium
/ 🟡 Low) with an **Effort** (S/M/L). Lead with an overall grade and a one-line verdict.

### security

Delegate the structured analysis to the `access-control-analysis` and `stride` skills if
present; this skill frames the scope and writes them up. Dimensions: trust boundaries,
authn/authz checkpoints, secret handling, input validation, dependency CVEs, dev/prod config
drift. **Rating:** severity (Critical/High/Medium/Low) + likelihood.

### testing

Delegate to `test-strategy` if present. Dimensions: what's tested vs. what's risky-and-untested,
coverage gate, happy-path bias, flaky/over-mocked tests, missing failure-path & edge tests.
**Rating:** per-area risk (High/Med/Low) × current coverage (None/Thin/Good).

### performance

Dimensions: `O(n·m)` loops, repeated work that could be cached/memoized, N+1 queries &
over-fetching, missing pagination/batching, missing DB indexes, blocking/serial I/O that could be
concurrent, connection/resource pooling, and (for UI) bundle/image weight & render thrash.
**Rating:** impact (1–5) × likelihood-of-hot-path (1–5).

### a11y

Cross-reference any `DESIGN.md` WCAG section. Dimensions: lint enforcement (jsx-a11y etc.),
focus management on dismiss/close, alt/aria/roles, contrast, keyboard nav. **Rating:** WCAG
level (A/AA/AAA) + severity.

## Hard rules

- **Read-only.** Do not edit source, run non-read tools, or commit. The only file you write is
  the `*_AUDIT.md` report. End by offering to implement fixes as a separate, explicit step.
- **Evidence over assertion.** Every finding cites `file:line`. Don't claim a bug you haven't
  located in the code.
- **Be honest both ways.** Include a "what's genuinely good" section; don't manufacture
  findings to pad the report, and don't soften a real one.
- **Don't repeat existing docs.** Cross-reference them.
