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
find . -name node_modules -prune -o \( -name "*.tsx" -o -name "*.vue" -o -name "*.svelte" \) -print \
  | grep -E "/(app|pages|routes|views)/" | head -40

# Top-level structure
ls -1

# Source file count — adjust extensions to the detected stack
find . -name node_modules -prune -o \( -name "*.ts" -o -name "*.tsx" -o -name "*.vue" -o -name "*.svelte" \) -print \
  | grep -vE "node_modules|\.next|stories|__tests__" | wc -l
```

Read `CLAUDE.md`, `AGENTS.md`, `DESIGN.md`, `ARCHITECTURE.md` if they exist — these
tell you which conventions are intentional vs. gaps. These four are a floor, not a
ceiling: also look for other docs that record deliberate decisions.

```bash
# Other docs that might record intentional decisions (ADRs, RFCs, design notes) —
# beyond the four canonical files already checked above
find . -name node_modules -prune -o -iname "*.md" -print \
  | grep -viE "node_modules|CHANGELOG|LICENSE" \
  | grep -iE "adr|decision|rfc|design|/docs/" | head -20
```

Read any hits before Step 1 so agents can route a matching finding to "Acknowledged, not
actioned" (Step 3) instead of reporting it as a gap.

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

## Step 1 — Parallel fan-out (4 roles)

Spawn all four as `Explore`-type agents (read-only — this skill never edits source), IN
PARALLEL in a single message; don't overlap their scopes. Each brief below uses React
terms as the default vocabulary — substitute the equivalent for the stack identified in
Step 0 (e.g. "hooks" → "composables" for Vue, "services"/stores for Angular).

**Scale the fan-out to the file counts from Step 0.** The four scopes below are roles,
not a fixed agent count: if a single role's scope exceeds roughly 150–200 files, split
that role by directory into parallel sub-agents (B1: `components/product`, B2:
`components/admin`, ...) rather than letting one agent sample silently. If sampling is
still unavoidable, pick a stated rule (e.g. every page plus the N most-imported
components) and record it in that agent's coverage line — never imply full coverage that
didn't happen.

**All four agents**, while reading, must also flag any comment, docstring, or nearby doc
reference that marks what looks like a gap as deliberate — `// intentional`, `// by
design`, an `eslint-disable` line with a justification, a `TODO` explaining why something
is deliberately left as-is, or a link to an ADR/design doc. Report these alongside the
finding they apply to, not as a separate list — Step 3 routes a finding with this kind of
evidence to "Acknowledged, not actioned" instead of ranking it as a gap.

**Report format — include this in every agent's brief.** Step 2 cross-references the four
reports mechanically and Step 4 cites their evidence verbatim, so a finding without a
location is unusable. Require each agent to:

- Attach a `path/to/file.ext:line` reference to **every** finding and inventory item —
  the exact line where the hook is called, the field is declared, the state is (not)
  handled.
- Structure the report as one block per file read, using the bullet headings of its brief
  as fixed subheadings — same shape for every file, so blocks are comparable.
- Return inventories (fields, hooks, nav links) as one line per item:
  `item — file:line — one-line note`. Agent C's field inventory in particular must be one
  line per field (`entity.field — file:line — fetched by <hook/service>`), because Step 2a
  matches it line-by-line against Agent A/B's component reports.
- End with a coverage line: which directories/files were read in full, which were skipped
  or sampled, and why — this feeds the report's scope/confidence note (Step 4).

### Agent A — Pages & routing

Read every page/route file. For each, report:

- Data hooks called and what they fetch
- Components rendered
- User actions available (buttons, forms, links)
- Navigation targets (where links/navigates go)
- What states are handled (loading / error / empty)
- What states are **not** handled that you'd expect
- ARIA/keyboard-accessibility gaps on any custom (non-native) control on the page —
  report this explicitly, don't leave it for Step 2c to catch opportunistically

### Agent B — Components & UI

Read every component file (product, admin, ui subdirectories) — **excluding**
header/footer/layout/navigation/auth chrome, which Agent D owns; don't report on those
files even in passing. For each:

- What data props/hooks it uses
- What user interactions it supports
- What states it handles
- What is visually present but has no interaction (display-only with no related action)
- Any obvious missing complementary action (e.g. search but no clear, sort but no reset)
- ARIA roles/labels and keyboard support on every custom (non-native) interactive
  element — dialogs, dropdowns, custom checkboxes/toggles, drag-and-drop targets. Report
  each one explicitly, whether it has them or not — don't only flag the missing ones

### Agent C — Data model & data-fetching layer

Read all hooks/composables/services (whatever the stack's data-fetching layer is called),
type files, API/seam files, and data-fetching utilities. If Step 0.5 produced a static-tool
report (knip, GraphQL Inspector/Hive, etc.), start from its findings and verify/extend them
rather than re-deriving everything from a blank slate. For each:

- All fields on every data type/entity
- What each hook/composable/service fetches and what mutations it provides
- Where each field is referenced *within the data layer itself* (selectors, transforms,
  mappers). Do **not** try to determine component-level consumption — the four agents run
  in parallel, so Agent A/B's findings don't exist yet from your point of view. Matching
  fields against what components actually render is Step 2a's job, done from your
  inventory plus theirs.

### Agent D — Global chrome & navigation

Read header, footer, layout, navigation, and auth/session components. Report:

- All nav links and where they go
- User identity signals present (name, avatar, role badge)
- Global state shown (notifications, counts, badges)
- What is in the data model that could/should surface globally but doesn't
- Whether the active nav item is exposed to assistive tech (`aria-current`), not just
  styled differently via CSS

---

## Step 2 — Synthesise findings

With all four agent reports in hand, cross-reference to find:

### 2a — Orphaned data fields

Fields that appear in type definitions or API responses but are never passed to or
rendered by any component. Determine this here, mechanically: match Agent C's field
inventory against the data props/hooks Agents A and B reported — this is the
cross-reference the parallel agents could not do themselves. Mark source file + line
(taken from Agent C's inventory entries). These are the clearest gaps.

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

**Don't let zero ARIA findings pass silently.** If none of the Step 1 agents surfaced an
accessibility gap, that's more often a sign the agents weren't asked to look than that
the app is clean — verify it yourself with a quick grep for custom interactive elements
(`<div onClick` — or the stack's equivalent: `@click` for Vue, `on:click` for Svelte,
`(click)` for Angular — plus `role=` and custom `Modal`/`Dialog`/`Dropdown` components)
before concluding there's nothing to report under #3/#9.

### 2d — Data used partially

Fields fetched and used in one context but not another where they'd be equally
relevant (e.g. user name shown on homepage but not in header).

---

## Step 3 — Rank gaps

Assign each gap a priority:

| Priority          | Criteria                                                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **P1 — Critical** | Prevents completion of a core workflow (checkout, sign-up, primary create/edit/delete action) or causes actual data loss/corruption               |
| **P2 — High**     | Workflow still completes, but with real friction, wrong/misleading information shown, or a clear break from an expectation set elsewhere in the app |
| **P3 — Medium**   | Missing feature users expect in an app of this type; reduces trust/engagement                                                                      |
| **P4 — Low**      | Polish, completeness, or edge-case correctness                                                                                                     |

"Confusing" or "suboptimal" alone never qualifies a gap for P1 — that's P2 at most. P1 is
reserved for a named workflow step the user cannot get past, or data that is actually lost.
Before finalizing, re-examine every P1 with a skeptical lens: would this really stop a user
from finishing the task, or would they grumble and continue anyway? If the workflow still
completes without the fix, downgrade it to P2. Every P1 card must state, in its "why this
matters" line, exactly which workflow and which step is blocked (see Step 4).

Within each priority, order by: frequency of user encounter → severity of consequence →
ease of fix (a quick win ranks above an equivalent slow one).

After ordering, assign each gap a **global sequential number** (G1, G2, G3, ...) running
across the entire ranked list — continuous across priority boundaries, not restarting at
P2/P3/P4. This is the number readers use to reference a specific gap ("issue 6"), so it
must stay stable and unique regardless of which priority section the gap sits in.

Tag each gap with a **type**, orthogonal to priority — this answers "what kind of finding
is this" separately from "how urgent is it":

- **Bug** — behavior contradicts something the app itself establishes elsewhere: a state
  handled on a parallel surface but not here (2b/2d), a broken/blank error or loading
  state, a dead-end action, an accessibility violation. Objectively wrong, not a matter of
  taste.
- **Code quality** — internal-only cleanup with no visible behavior change even after
  fixing (orphaned exports/fields, dead code, type mismatches, duplicated logic).
  Orphaned-data findings from 2a are almost always Code quality by default.
- **Enhancement** — nothing is broken; the app would simply be more complete, efficient,
  or polished with the addition (bulk actions, shortcuts, extra convenience the app
  doesn't establish or promise elsewhere). Most convention gaps (2c) that aren't tied to a
  broken state land here.

The Bug/Enhancement line isn't always sharp. When genuinely unsure, default to
Enhancement rather than Bug — the same evidence-over-assertion bar applies here as
everywhere else in this skill: don't call something a Bug unless you can point to the
specific established pattern or broken state it contradicts. Priority never constrains
type: a wholly missing capability (no password reset, no way to cancel an order) can be
P1 because it blocks a workflow, yet still be an Enhancement because nothing the app
establishes is contradicted. Don't adjust one to make the other "fit".

A gap that turns out to be an intentional product decision — confirmed via a project doc
found in Step 0 (`CLAUDE.md`/`AGENTS.md`/`DESIGN.md`/`ARCHITECTURE.md` or another doc
discovered there) or an in-code comment a Step 1 agent flagged — doesn't get a priority,
number, or type. Move it to the "Acknowledged, not actioned" section instead (Step 4) so
it stays visible without cluttering the ranked list. If you merely *suspect* a gap is
intentional but nothing documents it, keep it in the ranked list with the suspicion noted
on the card and raise it with the user at Step 5 — don't interrupt the analysis to ask,
and don't silently drop it.

---

## Step 4 — Write docs/GAP_ANALYSIS.html

Write a single self-contained HTML file to `docs/GAP_ANALYSIS.html`, creating the `docs/`
directory if it doesn't exist. No external dependencies — all CSS inline in a `<style>` block.

**Build the file incrementally, not in one giant write.** Long single-pass generations
degrade in their later sections. Write the shell first (head, styles, sticky header,
summary bar), then append each section as a separate operation — mind map, data-flow
table, one priority section at a time, orphaned data, acknowledged table, closing tags.
Then run the "Quality checks before finishing" against the assembled file on disk, not
against your memory of writing it.

**Always start fresh.** If `docs/GAP_ANALYSIS.html` already exists, do not read it or
treat it as a baseline — this run's findings come only from the current codebase. Delete
it first (`rm docs/GAP_ANALYSIS.html`), then write the new report from scratch. It's a
generated artifact, not source, and git history preserves the old version, so no backup
is needed. Deleting first also matters mechanically: the Write tool refuses to overwrite
a file it hasn't read this session, and reading the old report is exactly what this rule
forbids.

### Required sections

1. **Summary bar** — four cells: P1 count / P2 count / P3 count / P4 count with colour
   coding, a second row with Bug count / Code quality count / Enhancement count, plus a one-line
   scope/confidence note (stack detected, what was covered in full vs. sampled) so the
   report doesn't imply coverage it didn't achieve.

2. **App mind map** — visual map of every page/route with its data sources, user actions,
   and gaps called out inline (missing items in dashed red boxes). Use pure CSS/HTML —
   no SVG or canvas required.

3. **Data flow table** — every data entity and field, with columns:
   - Field name (monospace)
   - Fetched by (hook name badges)
   - Displayed on (page/component names)
   - Status: ✅ Full use / ⚠️ Partial / ✗ Orphaned

4. **Ranked gap cards** — one card per gap, grouped by priority with a coloured top border.
   Every priority section, including P3 and P4, uses this exact same card markup — never
   degrade to a plain `<ul>`/`<li>` list once a section has more items; if a priority has
   many entries, keep the card format and make it visually denser (tighter padding), not
   structurally different. Each card contains:
   - Global gap number from Step 3 (coloured circle, e.g. "G6") — unique and stable across
     the whole report, not renumbered per priority section
   - Title (one sharp sentence)
   - "What exists" block (grey inset — what's already there so readers understand context)
   - Description (why this matters, what the user experience is; for P1 cards, this must
     name the specific workflow and step that's blocked, per Step 3)
   - Meta badges (affected files, user impact label, type: Bug / Code quality / Enhancement)

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
- P1 red / P2 orange / P3 blue / P4 green — consistent throughout, applied via the top
  border and rank circle on every card regardless of priority
- Type badge uses a visually distinct style from the priority border (e.g. an outlined
  pill vs. the solid coloured border) so priority and type never get confused
- No JavaScript required for core content; a small scroll-spy for TOC is optional

---

## Step 5 — Offer next actions

After writing the HTML, present a short summary to the user:

- Total gap count by priority, and by type (Bug / Code quality / Enhancement)
- The top 3 highest-impact findings in plain text, by their global gap number
- Ask whether to file GitHub issues (individually or bundled by theme)

### If filing issues

Follow these conventions:

- **Bundle** gaps that touch the same files or the same user flow into one issue; prefer
  keeping Bug, Code quality, and Enhancement gaps in separate issues even if they touch
  the same file, since they likely have different reviewers/urgency
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
- The assembled HTML is verified mechanically against itself: summary-bar counts equal
  the number of rendered gap cards per priority and per type, every TOC/jump link points
  at an existing section id, every G-number appears exactly once as a card, and the file
  ends with its closing tags (no truncated final section)
- P1 gaps are never bundled away — each P1 gets its own issue or is the lead item
  in a bundle
- Every gap card, in every priority section from P1 through P4, has a numbered circle and
  a type badge — no section falls back to a plain bullet list
- Gap numbers are unique and sequential across the whole report, not restarted per priority
- Every P1 card's description names the specific workflow and step it blocks — a P1 with
  no named blocked workflow is a sign it should be P2

---

## Hard rules

- **Read-only on source.** Reading, running the project's own read-only tooling (e.g.
  type-checking to confirm a field's shape, `knip`/`eslint`/`graphql-inspector` per
  Step 0.5), and `git`/`gh` queries are fine. Never edit, format, or delete source files,
  and never let a tool run with an autofix flag or install anything that changes the
  lockfile. The only file you write is `docs/GAP_ANALYSIS.html` (plus GitHub issues, if
  the user opts in).
- **Evidence over assertion.** Every gap card and orphaned-field entry cites a real
  `file:line` located in the code during this run — by a Step 1 agent, a Step 0.5 static
  tool (knip, GraphQL Inspector/Hive), or your own verification grep in Step 2. Don't
  infer a gap you haven't located in the code, and don't cite a raw tool hit without a
  confirming read of that location.
- **Be honest both ways.** Don't manufacture gaps to pad the report, and don't soften a
  real one because it's inconvenient.
- **Always start fresh.** Never read or carry over a prior `docs/GAP_ANALYSIS.html` —
  see Step 4.
