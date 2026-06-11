---
name: accessibility-audit
description: >
  Run a WCAG 2.2 AA accessibility audit on a running web app and produce a markdown report.
  Combines Lighthouse (for category scores), axe-core (for static rule violations), and a
  scripted user-flow walkthrough to catch state-transition issues that static scans miss
  (loading announcements, route-change focus, disclosure behavior, keyboard traps).
  Works with either the Playwright MCP server or the Chrome DevTools MCP server.
  Use whenever the user asks about accessibility validation, WCAG compliance, screen-reader
  testing, axe-core, focus order, color contrast, keyboard navigation, ARIA correctness, or
  any a11y check on a running web app — even if they don't say the word "audit". Trigger on
  phrasings like "does this work with a screen reader", "is /checkout WCAG-compliant",
  "review the a11y here", "check this for contrast issues", "any focus traps?", or
  "regenerate the accessibility report".
allowed-tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - mcp__playwright__*
  - mcp__chrome-devtools__*
---

# Accessibility audit (WCAG 2.2 AA)

Audit a running web app for WCAG 2.2 AA conformance, combining three signal sources, and write the result to a stable markdown report at the project root.

## Step 0 — Gather what's needed (read context first, ask only what's missing)

This skill needs five inputs. **Parse them from the user's invocation, conversation context, and tool availability first. Only ask about what you genuinely don't know.** Batch any remaining questions into one message so the user isn't pinged repeatedly.

| Input                   | How to resolve without asking                                                                                                                                                                                                                                                                                          |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **App URL**             | Look in the user's message for a URL (`http://…`, `https://…`, or a hostname:port). Check earlier conversation. Don't default to `localhost:3000`. If still unknown, ask — this is the only input that's truly required.                                                                                               |
| **Pages to audit**      | If the user said "audit X" or "check the /foo page", use that. Otherwise default to the URL they gave; mention you'll also follow obvious in-app links during the flow audit. Don't ask unless the app clearly has multiple distinct surfaces (admin / public / etc.) and the user hasn't hinted.                      |
| **Authenticated state** | If the URL is `localhost`/`127.0.0.1` and the user didn't mention auth, assume none. If the URL is a remote or proxy host, ask once whether SSO/login is needed and remind the user to complete it in the MCP-driven Chrome before the skill navigates.                                                                |
| **Rule scope**          | Default to **strict WCAG 2.2 AA**. Only treat as "include best-practice" if the user explicitly used one of the phrasings listed in step 3. Don't ask.                                                                                                                                                                 |
| **MCP server**          | Detect from the available tools — if `mcp__chrome-devtools__lighthouse_audit` is reachable, prefer that (Lighthouse scores). If only `mcp__playwright__*` is reachable, use that and silently skip Lighthouse. Mention which one you're using in the report. Don't ask. **If neither is available, stop — see below.** |

If everything is resolved from context, proceed silently. If something is unclear, ask in one batched message — never one question at a time.

### Hard requirement: at least one browser-driving MCP server

The skill needs a browser to inject axe-core, capture the accessibility tree, and walk user flows. There is no fallback to static-only or source-only analysis here — those would miss most of the value (color contrast, focus order, state-transition issues).

If **neither** `mcp__playwright__*` nor `mcp__chrome-devtools__*` is available, stop immediately and tell the user:

> "I need a browser-driving MCP server to run this audit, and neither Playwright nor Chrome DevTools MCP is currently configured. Install one of these and try again:
>
> - **Chrome DevTools MCP** (recommended — includes Lighthouse scores): `claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest`
> - **Playwright MCP**: `claude mcp add playwright -- npx -y @playwright/mcp@latest`
>
> After installing, restart Claude Code so the new tools are picked up, then re-run this skill."

Don't try to substitute with static source review — that's a different kind of audit and the user should opt into it explicitly.

### Backing services

Don't start dev servers, proxies, or other backing services yourself — they're easy to leave running by accident and may already be up. If the URL doesn't respond, ask the user to start what's needed.

## Step 1 — Open the page and let the app reach a steady state

Navigate to the URL with the chosen MCP server and wait for the primary content to render (3–5 seconds is usually enough; for slower SPAs, use `wait_for` or `browser_wait_for` against a text marker the user identifies, e.g. the first item title).

If the snapshot still shows a "Loading…" state after a few seconds:

- Check console errors (`browser_console_messages` / `list_console_messages`) and network requests (`browser_network_requests` / `list_network_requests`).
- 401/403/500 on data endpoints usually means a stale auth session — surface this to the user and stop until resolved.

Take an initial snapshot. This is the **baseline state**.

## Step 2 — Lighthouse (Chrome DevTools MCP only)

If `mcp__chrome-devtools__lighthouse_audit` is available (and the user didn't ask to skip it):

```
mcp__chrome-devtools__lighthouse_audit {
  device: 'desktop',
  mode: 'navigation',
}
```

Record the **Accessibility** and **Best Practices** category scores. Lighthouse runs a subset of axe rules plus its own checks (e.g. `errors-in-console`, `inspector-issues`). Note any non-overlapping items, but expect most accessibility findings to also surface in step 3.

If Lighthouse isn't available, document that in the report and continue with axe + manual.

## Step 3 — axe-core with WCAG 2.2 AA scope

Inject axe-core from a CDN allowed by the page's Content Security Policy. Common choices:

- `https://cdn.jsdelivr.net/npm/axe-core@4.10.2/axe.min.js`
- `https://unpkg.com/axe-core@4.10.2/axe.min.js`
- `https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.10.0/axe.min.js`

If the first attempt fails with a CSP error in the console, try another CDN. If none are allowed, ask the user to whitelist one or to install axe-core locally and host it from the dev server.

### Rule scope (strict WCAG 2.2 AA by default)

Run axe with a `runOnly` filter so the report matches the declared standard:

```js
async () => {
  if (!window.axe) {
    await new Promise((resolve, reject) => {
      const s = document.createElement("script");
      s.src = "https://cdn.jsdelivr.net/npm/axe-core@4.10.2/axe.min.js";
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });
  }
  const r = await window.axe.run(document, {
    resultTypes: ["violations", "incomplete"],
    runOnly: {
      type: "tag",
      // WCAG 2.2 only ADDS criteria — it doesn't replace 2.0/2.1.
      // Full "2.2 AA" coverage requires all five tags below.
      values: ["wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "wcag22aa"],
    },
  });
  return {
    violations: r.violations.map((v) => ({
      id: v.id,
      impact: v.impact,
      help: v.help,
      helpUrl: v.helpUrl,
      tags: v.tags,
      nodes: v.nodes.map((n) => ({
        target: n.target,
        html: n.html.slice(0, 220),
        summary: n.failureSummary,
      })),
    })),
    incomplete: r.incomplete.map((v) => ({
      id: v.id,
      impact: v.impact,
      help: v.help,
      nodes: v.nodes.length,
    })),
  };
};
```

Notes:

- `wcag22aa` covers all WCAG 2.2 additions including the new Level A criteria (3.2.6, 3.3.7); axe-core groups them under this single tag (no separate `wcag22a`).
- Always return `v.tags` per violation so the report can show which WCAG version surfaced each rule.

**Opt-in: best-practice rules.** By default the run is strict WCAG 2.2 AA — axe's own "best practice" rules (notably `region`, which flags content outside any landmark) are **not** included. If the user asks to include them — phrasings to watch for: "include best practices", "also run best-practice rules", "use the full axe ruleset", "don't restrict to WCAG" — add `'best-practice'` to `values`:

```js
values: ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa', 'best-practice'],
```

When best-practice is included, mark those findings as "best-practice" in the report instead of citing a WCAG criterion they don't actually map to.

### Grouping output

When `color-contrast` returns many nodes, group them by foreground/background pair so the report stays scannable. Regex on `failureSummary`:

```
color contrast of ([\d.]+).*foreground color: (#[0-9a-f]+), background color: (#[0-9a-f]+)
```

Report each pair once with a node count, instead of listing every instance.

## Step 4 — WCAG 2.2-specific checks axe can't fully cover

Run `evaluate_script` / `browser_evaluate` to verify each:

- **[2.5.8 Target Size (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum) (AA)** — iterate `a, button, input, [role="button"], [tabindex]:not([tabindex="-1"])`; flag any visible target with `getBoundingClientRect()` width or height < 24 CSS px. Skip native `input[type=date]` and `input[type=time]` (browser-rendered, sizing varies).
- **[2.4.11 Focus Not Obscured (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/focus-not-obscured-minimum) (AA)** — scan for `position: fixed | sticky` elements and tall headers/footers. Note any that could clip the focus ring during Tab navigation.
- **[2.4.7 Focus Visible](https://www.w3.org/WAI/WCAG22/Understanding/focus-visible) (AA, carried from 2.1)** — for each focusable, call `el.focus()` then read `outlineStyle`, `outlineWidth`, `outlineColor`, and `boxShadow`. Note that `el.focus()` from JS may not match `:focus-visible` in all browsers — cross-check the source for `:focus-visible` styles, then verify with real Tab presses in step 6 (checkpoint 9).
- **[2.5.7 Dragging Movements](https://www.w3.org/WAI/WCAG22/Understanding/dragging-movements) (AA)** — N/A unless the app has drag interactions; if it does, confirm a single-pointer alternative exists.
- **[3.2.6 Consistent Help](https://www.w3.org/WAI/WCAG22/Understanding/consistent-help) (A)** — if the app exposes a help mechanism (link, button, chat), confirm it's in the same relative location on every page.
- **[3.3.7 Redundant Entry](https://www.w3.org/WAI/WCAG22/Understanding/redundant-entry) (A)** — if the app has multi-step forms, confirm the user isn't asked to re-enter information they've already provided in the same session.
- **[3.3.8 Accessible Authentication (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/accessible-authentication-minimum) (AA)** — confirm no cognitive function test (memorization, transcription, puzzle) is required without an alternative.

## Step 5 — Manual review of the accessibility tree

Read the snapshot output (`browser_snapshot` / `take_snapshot`) and the source for patterns axe doesn't flag:

- **Redundant accessible names.** Interactive elements whose accessible name concatenates duplicate content from multiple sources (visible text + `aria-label` + non-empty image `alt` on the same button). Fix by marking decorative icons `alt=""` and hiding redundant text from the accessible name with `aria-hidden="true"`.
- **State conveyed only by color/strikethrough.** Checklists, status pills, toggles where the only difference between states is a class change. Expose state via `role="checkbox" aria-checked` (or visually-hidden prefix text); mark color-only indicators `aria-hidden="true"`.
- **Section labels that should be headings.** A `<p>` styled like a heading is invisible to heading-navigation. Promote to the correct `<h*>`.
- **Nested interactives.** Any focusable inside another button or link (link inside a button, button inside a link) is illegal and causes inconsistent SR behavior.
- **`aria-live` coverage.** Search the DOM for `[aria-live], [role="status"], [role="alert"]`. Zero hits is a red flag if the app has any dynamic content (loading spinners, error messages, autocomplete results). Caveat: many UI libraries mount **a single shared announcer region** that components reuse via an `announce()` API. One live region can be correct if it's an announcer — don't flag low counts on count alone. Check what triggers the live region (is it actually updated when state changes?) rather than how many exist.

## Step 6 — User flow audit

Lighthouse and a one-shot axe run only see the page at load time. Real users move through state changes, and each state can introduce new violations. Walk through these checkpoints; at each one: snapshot → re-run axe → record any **new** violations not already in the baseline.

The checkpoint list below is a starting template — drop ones that don't apply, add app-specific ones the user mentions.

1. **Initial load.** Baseline (already captured).
2. **Expand a disclosure / open a dropdown.** Trigger should have `aria-expanded="true"` and `aria-controls="<panel id>"`. Panel exists, matches that id. Focus should move predictably (typically stays on the trigger for inline disclosures, moves into the panel for modal-like ones).
3. **Open a second instance of the same widget.** Confirms multi-open behavior. If the app auto-closes the previous one, focus should land on the newly-opened trigger.
4. **Collapse / close.** `aria-expanded` flips back; focus is not lost.
5. **Route change** (in-app navigation, not full reload). Use the link or button, not direct URL navigation, so the router is exercised. Confirm `aria-current="page"` moves; confirm focus moves or is announced (focus-management or `aria-live` are the two patterns).
6. **Apply a filter / search / form input.** Result count change should be announced via `aria-live="polite"`; the empty-result state should have a meaningful message.
7. **Loading state.** Throttle the network or trigger a slow fetch. Loading indicators should at minimum be inside a landmark; ideally have `aria-live="polite"` or `role="status"`.
8. **Error state.** Force a 4xx/5xx (block the request in DevTools, or have the user kill the backend). Error messages should have `role="alert"` (implies `aria-live="assertive"`).
9. **Keyboard-only pass.** Reload, then press `Tab` repeatedly. After each press, read `document.activeElement` and `getComputedStyle(...).outline*` / `boxShadow`. Flag:
   - Focus stops on a non-interactive element.
   - A focusable with no visible focus indicator.
   - Tab order that doesn't match visual order.
   - Skipped focusables (anything visually interactive that Tab passes over).
10. **`Escape` from an open disclosure / modal.** Should close. Modal dialogs must also trap focus while open and restore focus to the opener on close — verify both.

Record each checkpoint's outcome in a table in the report. Don't repeat findings already listed under the baseline.

## Step 7 — Write the report

### Pick the right location for _this_ repo

Don't hard-code `ACCESSIBILITY.md` at the project root — different repos have different conventions. Resolve the location in this order:

1. **Reuse an existing convention if there is one.** Glob for prior reports:
   - `ACCESSIBILITY.md` (root)
   - `docs/**/ACCESSIBILITY.md`, `docs/**/accessibility.md`
   - `docs/accessibility/*.md`, `audits/**/*.md`, `reports/**/a11y*.md`

   If one or more match, use the same location and naming.

2. **Follow the repo's docs convention** if there's no prior report but the repo has a clear docs structure:
   - A `docs/` directory → `docs/accessibility.md` (or `docs/audits/accessibility.md` if `docs/audits/` already exists).
   - A `reports/` or `audits/` directory → put it there.
   - Neither → default to `ACCESSIBILITY.md` at the project root.
3. **Ask the user** if the conventions point in different directions, or if it's the first audit in a repo with multiple plausible locations. One batched question — propose your best guess and let them override.

Whatever location is chosen, **the live report uses a stable filename** (no date in the name) so PR links and bookmarks stay valid.

### Preserve history before overwriting

If a prior live report exists at the resolved location:

1. Read its `**Date:**` line.
2. Move it to a sibling `accessibility/<that-date>.md` (or whatever archive convention the repo already uses — `docs/accessibility/`, `audits/archive/`, etc.).
3. Create the archive directory if it doesn't exist (`mkdir -p`).
4. If the previous date matches today's, append a short suffix (`-2`, `-3`, …) so reruns on the same day don't clobber each other.

Then write the fresh report to the live location. The stable filename keeps links valid; the dated archive preserves history.

### Report structure

```
# Accessibility Report — <app name>

**Date:** <YYYY-MM-DD>
**Tooling:** Lighthouse (via chrome-devtools MCP) + axe-core 4.10.2 + manual review of the accessibility tree and user flows
**Pages audited:** <list>
**Standard:** WCAG 2.2 AA (latest, supersedes 2.1)
**Rule scope:** strict WCAG 2.2 AA  ← or  WCAG 2.2 AA + axe best-practice rules

## Lighthouse score
- Accessibility: <0-100>
- Best Practices: <0-100>

## Summary
<table: #, Source (Lighthouse | axe-core | Manual | Flow audit), Severity, Count, Issue>

### WCAG 2.2-specific checks (passed or N/A)
<table covering 2.4.7 (focus visible), 2.4.11, 2.5.7, 2.5.8, 3.2.6, 3.3.7, 3.3.8>

## User flow audit
<table: Checkpoint, New violations, Notes>

## <N>. <Finding title> *(Lighthouse | axe-core | manual | flow audit)*

**WCAG:** <link to WCAG 2.2 Understanding doc> Level <A | AA>   ← or  *axe best-practice (advisory)*
**Severity:** <Serious | Moderate | Minor>
**Affected nodes:** <count + where>

<one-paragraph description of the violation, including a concrete excerpt of the failing markup if useful>

### Recommendation
<concrete fix — token names, code shape, or component change>

---

## Positive findings
<bullet list of things already done well>

## Suggested fix order
<numbered list, prioritized by impact-per-effort>

## Re-test
<one-paragraph note on how to re-run this skill>
```

Rules:

- **Always use `https://www.w3.org/WAI/WCAG22/Understanding/<slug>` for criterion links** (not `WCAG21`).
- Order findings by severity (Serious → Moderate → Minor). Within a severity, axe rules → manual → flow audit.
- Close with a "Positive findings" section and a "Suggested fix order" punch list, prioritized by impact-per-effort (e.g., a single token change that clears 80% of contrast violations goes near the top).

## Step 8 — Clean up

- Close any browser pages you opened: `mcp__playwright__browser_close` or `mcp__chrome-devtools__close_page`.
- **Do not kill dev servers, proxies, or other processes the user was running** — even if you (or an earlier session) started them. Closing browsers and reporting the new file path is enough. Ask the user if anything should be stopped.

## Reference: severity rubric

| Severity | When to use                                                                                                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Serious  | WCAG AA fail that blocks a user category — e.g. contrast below required ratio, state conveyed only by color, missing accessible name, missing `aria-live` for dynamic content. |
| Moderate | Structural defect that degrades AT navigation but content is reachable — missing landmark, heading skip, redundant alt that creates noisy SR output.                           |
| Minor    | Verbose/awkward accessible name, non-blocking semantic improvements, usability gaps that don't strictly fail a SC (e.g. Escape doesn't close a disclosure).                    |

## Reference: WCAG 2.2 vs 2.1 delta

- 2.2 **added** 9 criteria: 2.4.11, 2.4.12, 2.4.13, 2.5.7, 2.5.8, 3.2.6, 3.3.7, 3.3.8, 3.3.9.
- 2.2 **removed** SC 4.1.1 Parsing (now obsolete — modern HTML parsers handle malformed markup deterministically).
- All other 2.1 criteria carry over unchanged.
- Always report against 2.2. Update WCAG link URLs from `WCAG21` to `WCAG22`.

## Reference: choosing between Playwright MCP and Chrome DevTools MCP

| Capability                                                | Playwright MCP | Chrome DevTools MCP    |
| --------------------------------------------------------- | -------------- | ---------------------- |
| `browser_snapshot` / `take_snapshot` (accessibility tree) | ✓              | ✓                      |
| `evaluate` JS (inject axe-core)                           | ✓              | ✓                      |
| Lighthouse audit (category scores)                        | ✗              | ✓ (`lighthouse_audit`) |
| Console + network access                                  | ✓              | ✓                      |
| Click / type / press key for flow audit                   | ✓              | ✓                      |

Both can drive the audit. **Chrome DevTools is required for Lighthouse scores.** If both are available, prefer chrome-devtools for the Lighthouse step and either for the rest.
