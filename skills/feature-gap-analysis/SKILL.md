---
name: feature-gap-analysis
description: >
  Fan out across the entire application, map every user path and data field, find what
  is fetched/modelled but never rendered, identify missing features expected by consistency
  or convention, rank all gaps by user impact, and produce a self-contained HTML report
  with a mind map and prioritized gap cards. Adapts its fan-out to the project's actual
  frontend stack (React, Vue, Angular, Svelte, ...). Read-only on source — never edits
  files. Output is a single docs/GAP_ANALYSIS.html, plus an optional batch of GitHub
  issues. Each run writes a fresh report from scratch, overwriting any prior one.
  Invoke explicitly with `/feature-gap-analysis` — this is an expensive, multi-agent
  survey and must never trigger automatically from conversation.
disable-model-invocation: true
---

# Feature Gap Analysis

A read-only harness that turns "what's missing in this app?" into a prioritised,
evidence-backed HTML report: an app mind map, a data-flow table showing orphaned
fields, and ranked gap cards with file references and proposed fixes.

## When to use / not use

- **Use** when you want a broad survey of missing features, inconsistencies, and
  data that is fetched but never displayed — across a whole frontend or full-stack app.
- **Don't use** for a single component review, a PR diff, or a security/performance
  audit (use the `codebase-audit` skill with the relevant lens for those).
- **Output:** `docs/GAP_ANALYSIS.html` — self-contained, no dependencies, opens directly
  in any browser.

---

## Step 0 — Orientation

**Identify the stack first.** Read `package.json` (or the equivalent manifest for a
full-stack app) to see which frontend framework is in play — React/Next.js, Vue/Nuxt,
Angular, Svelte/SvelteKit, or something else. This determines the vocabulary and file
globs for the rest of this skill: "hooks" for React, "composables" for Vue, "services"/
"stores" for Angular, and so on. Adapt every glob below and every agent brief in Step 1
to match — the examples given use React as the default, but substitute freely.

Then do a rapid structural survey yourself:

```bash
# Route / page files — adjust the extension(s) to the detected stack
find . -path ./node_modules -prune -o \( -name "*.tsx" -o -name "*.vue" -o -name "*.svelte" \) -print \
  | grep -E "/(app|pages|routes|views)/" | head -40

# Top-level structure
ls -1

# Source file count — adjust extensions to the detected stack
find . -path ./node_modules -prune -o \( -name "*.ts" -o -name "*.tsx" -o -name "*.vue" -o -name "*.svelte" \) -print \
  | grep -vE "node_modules|\.next|stories|__tests__" | wc -l
```

Read `CLAUDE.md`, `AGENTS.md`, `DESIGN.md`, `ARCHITECTURE.md` if they exist — these
tell you which conventions are intentional vs. gaps.

---

## Step 0.5 — Ground-truth static analysis

Before spawning agents, run whatever static tools the stack already supports. A tool hit
is reproducible, file:line-backed evidence — it should anchor Agent C's work in Step 1
rather than being re-derived from scratch by an agent reading files:

- **JS/TS projects:** run `npx knip` (no install needed) to get unused files, unused
  exports, and unused dependencies. Feed its output to Agent C as a starting list of
  orphan candidates to verify and extend. Knip catches whole unused hooks/components/
  files; it does **not** catch a field that's part of a used type but only partially
  destructured or rendered — that finer-grained case is still Agent C's job.
- **GraphQL APIs:** if a schema/codegen setup exists, check for an existing Hive "unused
  schema" report or run `graphql-inspector` if it's already configured, to find unused
  types/fields/arguments directly from the schema.
- **React component prop usage:** `eslint` with `react/no-unused-prop-types` (if already
  configured) or `react-scanner` can flag props that are declared but never read.

Don't install heavyweight tooling the project doesn't already use. If none of the above
apply, skip this step, rely on the Step 1 agents alone, and note the absence in the
report's scope/confidence line (Step 4).

---

## Step 1 — Parallel fan-out (4 agents)

Spawn all four as `Explore`-type agents (read-only — this skill never edits source), IN
PARALLEL in a single message; don't overlap their scopes. Each brief below uses React
terms as the default vocabulary — substitute the equivalent for the stack identified in
Step 0 (e.g. "hooks" → "composables" for Vue, "services"/stores for Angular).

### Agent A — Pages & routing

Read every page/route file. For each, report:

- Data hooks called and what they fetch
- Components rendered
- User actions available (buttons, forms, links)
- Navigation targets (where links/navigates go)
- What states are handled (loading / error / empty)
- What states are **not** handled that you'd expect

### Agent B — Components & UI

Read every component file (product, layout, admin, ui subdirectories). For each:

- What data props/hooks it uses
- What user interactions it supports
- What states it handles
- What is visually present but has no interaction (display-only with no related action)
- Any obvious missing complementary action (e.g. search but no clear, sort but no reset)

### Agent C — Data model & data-fetching layer

Read all hooks/composables/services (whatever the stack's data-fetching layer is called),
type files, API/seam files, and data-fetching utilities. If Step 0.5 produced a static-tool
report (knip, GraphQL Inspector/Hive, etc.), start from its findings and verify/extend them
rather than re-deriving everything from a blank slate. For each:

- All fields on every data type/entity
- What each hook/composable/service fetches and what mutations it provides
- Which fields are fetched but **not** consumed by any component (cross-reference with Agent A/B findings)

### Agent D — Global chrome & navigation

Read header, footer, layout, navigation, and auth/session components. Report:

- All nav links and where they go
- User identity signals present (name, avatar, role badge)
- Global state shown (notifications, counts, badges)
- What is in the data model that could/should surface globally but doesn't

---

## Step 2 — Synthesise findings

With all four agent reports in hand, cross-reference to find:

### 2a — Orphaned data fields

Fields that appear in type definitions or API responses but are never passed to or
rendered by any component. Mark source file + line. These are the clearest gaps.

### 2b — Consistency gaps

Features present on one page/surface but absent on a parallel one where users would
expect the same. Examples: countdown on marketplace cards but not on My Account rows;
withdraw on product page but not from My Account; error state on one tab but blank
white on another.

### 2c — Convention gaps

Walk Nielsen Norman's 10 usability heuristics as the checklist — the industry-standard
baseline for "things every app has by default," not an ad-hoc list:

1. **Visibility of system status** — no loading/progress indicator; no confirmation
   after an action; no "last updated" on legal or time-sensitive documents.
2. **Match between system and the real world** — internal/technical jargon exposed to
   users instead of their own vocabulary.
3. **User control and freedom** — no cancel/undo/back-out of a multi-step flow; search
   with no clear button; no way to reset filters.
4. **Consistency and standards** — the same control behaving differently across
   surfaces (the cross-page variant of this is 2b); departure from platform conventions.
5. **Error prevention** — destructive actions with no confirmation step; no input
   validation before submit.
6. **Recognition rather than recall** — user must remember state from a prior screen
   instead of seeing it (no result count after filtering; filter/sort state not in the
   URL, so it's lost on refresh/back).
7. **Flexibility and efficiency of use** — no bulk actions, keyboard shortcuts, or
   power-user path for a repetitive task.
8. **Aesthetic and minimalist design** — usually out of scope for this skill; only flag
   if clutter itself obscures a needed action.
9. **Help users recognize, diagnose, and recover from errors** — a blank/white screen
   or raw error instead of an actionable error state; no empty state on a tab with zero
   results.
10. **Help and documentation** — no inline help/tooltip on a non-obvious control where
    the app's own conventions require domain knowledge.

Missing ARIA patterns and keyboard nav on custom controls are convention gaps too — file
them under whichever heuristic fits (usually #3 or #9). If the project has a dedicated
a11y need, point at the `codebase-audit` skill's `a11y` lens rather than duplicating a
full accessibility audit here.

### 2d — Data used partially

Fields fetched and used in one context but not another where they'd be equally
relevant (e.g. user name shown on homepage but not in header).

---

## Step 3 — Rank gaps

Assign each gap a priority:

| Priority          | Criteria                                                                        |
| ----------------- | ------------------------------------------------------------------------------- |
| **P1 — Critical** | Breaks or seriously impairs a core user workflow; causes confusion or data loss |
| **P2 — High**     | Significant usability or consistency gap; users notice and are frustrated       |
| **P3 — Medium**   | Missing feature users expect in an app of this type; reduces trust/engagement   |
| **P4 — Low**      | Polish, completeness, or edge-case correctness                                  |

Within each priority, order by: frequency of user encounter → severity of consequence →
ease of fix (a quick win ranks above an equivalent slow one).

A gap that turns out to be an intentional product decision (confirmed via `CLAUDE.md`/
`DESIGN.md`, a code comment, or asking the user) doesn't get a priority — move it to the
"Acknowledged, not actioned" section instead (Step 4) so it stays visible without
cluttering the ranked list.

---

## Step 4 — Write docs/GAP_ANALYSIS.html

Write a single self-contained HTML file to `docs/GAP_ANALYSIS.html`, creating the `docs/`
directory if it doesn't exist. No external dependencies — all CSS inline in a `<style>` block.

**Always start fresh.** If `docs/GAP_ANALYSIS.html` already exists, do not read it or
treat it as a baseline — this run's findings come only from the current codebase.
Overwrite it wholesale; git history preserves the old version, so no backup is needed.

### Required sections

1. **Summary bar** — four cells: P1 count / P2 count / P3 count / P4 count with colour
   coding, plus a one-line scope/confidence note (stack detected, what was covered in
   full vs. sampled) so the report doesn't imply coverage it didn't achieve.

2. **App mind map** — visual map of every page/route with its data sources, user actions,
   and gaps called out inline (missing items in dashed red boxes). Use pure CSS/HTML —
   no SVG or canvas required.

3. **Data flow table** — every data entity and field, with columns:
   - Field name (monospace)
   - Fetched by (hook name badges)
   - Displayed on (page/component names)
   - Status: ✅ Full use / ⚠️ Partial / ✗ Orphaned

4. **Ranked gap cards** — one card per gap, grouped by priority with a coloured top border.
   Each card contains:
   - Rank number (coloured circle)
   - Title (one sharp sentence)
   - "What exists" block (grey inset — what's already there so readers understand context)
   - Description (why this matters, what the user experience is)
   - Meta badges (affected files, user impact label)

5. **Orphaned data section** — grid of cards, one per orphaned field. Each shows:
   - Field path in monospace
   - Where it's fetched from
   - Where it _could_ be used

6. **Acknowledged, not actioned** — a small table for gaps that turned out to be
   intentional product decisions (see Step 3). Columns: gap title, rationale, where the
   decision is documented (if anywhere). Keeps these visible instead of silently
   dropping them from the ranked list.

### Design guidelines for the HTML

- Sticky header with section jump links
- Fixed TOC on the right (hidden on narrow screens)
- Colour palette: dark background for header, light grey page, white cards
- P1 red / P2 orange / P3 blue / P4 green — consistent throughout
- No JavaScript required for core content; a small scroll-spy for TOC is optional

---

## Step 5 — Offer next actions

After writing the HTML, present a short summary to the user:

- Total gap count by priority
- The top 3 highest-impact findings in plain text
- Ask whether to file GitHub issues (individually or bundled by theme)

### If filing issues

Follow these conventions:

- **Bundle** gaps that touch the same files or the same user flow into one issue
- **Separate** gaps that have different owners (frontend vs. backend, different subsystems)
- For gaps that require a backend API change, also offer to file a corresponding issue
  in the backend/sibling repository
- Issue body structure: brief problem statement, "What exists" context block, proposed fix
  with file references, acceptance criteria
- Keep tone technical and concise — reference specific component names and file paths
- Cross-link related issues (`#N`) and sibling-repo issues (`owner/repo#N`)
- Intentional non-gaps (known product decisions) belong in the report's "Acknowledged,
  not actioned" section (Step 4), not as filed issues — suggest documenting the
  rationale in code comments or design docs if it isn't already

---

## Quality checks before finishing

- Every gap card references at least one specific file path
- Orphaned data table has an entry for every field that is fetched but unused
- Where Step 0.5 ran a static tool (knip, GraphQL Inspector/Hive, etc.), its findings
  are reflected in the orphaned-data section, not just re-derived prose from the agents
- Every gap flagged as an intentional product decision during synthesis appears in the
  "Acknowledged, not actioned" section — none are silently dropped
- No gap is filed as an issue without first checking if it's already tracked
  (`gh issue list --search "keyword"`)
- The HTML opens cleanly in a browser with no console errors
- P1 gaps are never bundled away — each P1 gets its own issue or is the lead item
  in a bundle

---

## Hard rules

- **Read-only on source.** Reading, running the project's own read-only tooling (e.g.
  type-checking to confirm a field's shape, `knip`/`eslint`/`graphql-inspector` per
  Step 0.5), and `git`/`gh` queries are fine. Never edit, format, or delete source files,
  and never let a tool run with an autofix flag or install anything that changes the
  lockfile. The only file you write is `docs/GAP_ANALYSIS.html` (plus GitHub issues, if
  the user opts in).
- **Evidence over assertion.** Every gap card and orphaned-field entry cites a real
  `file:line` found by one of the Step 1 agents — don't infer a gap you haven't located
  in the code.
- **Be honest both ways.** Don't manufacture gaps to pad the report, and don't soften a
  real one because it's inconvenient.
- **Always start fresh.** Never read or carry over a prior `docs/GAP_ANALYSIS.html` —
  see Step 4.
