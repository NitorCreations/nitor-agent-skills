---
name: npm-supply-chain-audit
description: >
  Audit and fix npm supply-chain security issues in the current repo. Detects the package manager,
  checks for missing protections (lockfile, lifecycle script blocking, release-age cooldown,
  pnpm exotic subdeps/trust policy, Yarn Berry hardened mode), presents findings, and applies
  fixes after user confirmation. Supports npm, pnpm, Yarn, Bun, and Aube.
  Use when asked to "harden npm", "fix supply chain", "secure dependencies", or "audit npm security".
allowed-tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash(node *)
  - Bash(npm *)
  - Bash(pnpm *)
  - Bash(yarn *)
  - Bash(bun *)
  - Bash(git *)
---

# npm supply-chain security audit

Audit the current repo for npm supply-chain security issues, present findings to the user, and apply fixes only after confirmation.

## Step 1 – Detect the package manager

Check which package manager(s) are in use:

- `package-lock.json` -> npm
- `aube-lock.yaml` -> Aube
- `pnpm-lock.yaml` -> pnpm
- `yarn.lock` + `.yarnrc.yml` -> Yarn Berry; `yarn.lock` without `.yarnrc.yml` -> Yarn Classic
- `bun.lockb` or `bun.lock` -> Bun
- `packageManager` field in `package.json` confirms the PM and version

Use the `packageManager` field to determine the version. If not present, run the PM's `--version` command. The version is needed for checks that depend on minimum PM versions (e.g. pnpm 10+, pnpm 11+, npm 11.10+, Yarn Berry 4.10+, Bun 1.3+).

pnpm 11 ships several supply-chain protections on by default (`minimumReleaseAge: 1440`, `blockExoticSubdeps: true`, `strictDepBuilds: true`) and only reads pnpm-specific settings from `pnpm-workspace.yaml` (or `~/.config/pnpm/config.yaml`), not from `.npmrc`. Many checks below differ between pnpm 10 and pnpm 11+.

If no lockfile exists, default to npm.

## Step 2 – Audit

Work through the checks below. For each one, check the current state and record whether it passes or needs a fix. Do NOT make any changes yet.

### 2.1 Lockfile committed

If no lockfile exists (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, `bun.lock`, `aube-lock.yaml`), warn the user they should run their package manager's install command and commit the lockfile. Do NOT run install yourself.

### 2.2 Block lifecycle scripts

**npm** – ensure `.npmrc` contains:

```ini
ignore-scripts=true
```

**pnpm 10+** – blocks lifecycle scripts by default. Check that `pnpm-workspace.yaml` does NOT set `dangerouslyAllowAllBuilds: true`. If it does, warn the user. pnpm 11+ additionally enables `strictDepBuilds: true` by default, so installs error (rather than just warn) when a dependency wants to run a build script that isn't in the allowlist – verify it has not been disabled in `pnpm-workspace.yaml`.

**pnpm <= 9** – ensure `.npmrc` contains `ignore-scripts=true`.

**Yarn Classic (v1)** – ensure `.npmrc` contains `ignore-scripts=true`.

**Yarn Berry (v2+)** – ensure `.yarnrc.yml` contains:

```yaml
enableScripts: false
```

**Bun** – blocks lifecycle scripts by default. No action needed.

**Aube** – blocks lifecycle scripts by default. Check that `aube-workspace.yaml` does NOT set `dangerouslyAllowAllBuilds: true`. If it does, warn the user.

### 2.3 Build script allowlists

When lifecycle scripts are blocked, some packages legitimately need build scripts. Check the lockfile for packages that require builds:

- **pnpm** – look for entries with `requiresBuild: true` in `pnpm-lock.yaml`
- **npm** – look for `hasInstallScript: true` in `package-lock.json`
- **Yarn Berry** – look for `hasBin` or build-related metadata in `yarn.lock`
- **Bun** – check `bun.lock` for packages with lifecycle scripts

Compare the list of packages that require builds against the current allowlist. Report any that are missing, but do NOT suggest adding them blindly – the user must verify each package is trusted before allowlisting it. Only well-known packages from reputable maintainers (e.g. `esbuild`, `sharp`, `@swc/core`) should be considered.

The allowlist setting per package manager:

**pnpm 11+** – `allowBuilds` map in `pnpm-workspace.yaml`. This replaces `onlyBuiltDependencies`, `onlyBuiltDependenciesFile`, `neverBuiltDependencies`, and `ignoredBuiltDependencies`, which are no longer supported:

```yaml
allowBuilds:
  esbuild: true
  core-js: false
```

**pnpm 10 / Aube** – `onlyBuiltDependencies` in `pnpm-workspace.yaml` / `aube-workspace.yaml`:

```yaml
onlyBuiltDependencies:
  - esbuild
```

**Yarn Berry** – `dependenciesMeta` in `package.json`:

```json
"dependenciesMeta": {
  "esbuild": { "built": true }
}
```

**Bun** – `trustedDependencies` in `package.json`:

```json
"trustedDependencies": ["esbuild"]
```

### 2.4 CI pipeline impact analysis (lifecycle script blocking)

Whenever lifecycle script blocking is being recommended (or is already configured), scan all CI workflow files (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `bitbucket-pipelines.yml`, `.circleci/config.yml`, etc.). For each job that installs dependencies, check whether subsequent steps invoke tools that depend on packages requiring build scripts (as identified from the lockfile in check 2.3). Also check `package.json` for lifecycle hooks (`prepublishOnly`, `prepare`, `prepack`) that run implicitly during publish/pack commands and would be silently skipped.

Report all affected workflows and hooks as part of the findings alongside the proposed blocking fix, specifying the package-manager-appropriate override:

- **npm** – `npm rebuild --ignore-scripts=false <pkg>` after install; `npm_config_ignore_scripts=false` env var on publish steps
- **pnpm** – packages in `allowBuilds` / `onlyBuiltDependencies` are rebuilt automatically; no extra step needed if the allowlist is correct
- **Yarn Berry** – packages with `dependenciesMeta.built: true` are rebuilt automatically after install
- **Bun** – packages in `trustedDependencies` are rebuilt automatically after install

Do not present findings from checks 2.2 or 2.3 until this check is complete — the user must see the blocking config, the allowlist, and any CI/hook impacts together in Step 3.

### 2.5 Release-age cooldown

Check if a release-age cooldown is configured. If not, recommend adding one. If already configured, note the current value. When presenting findings, advise the user that 3 days is a good minimum. Mention that 7 days provides stronger protection but slows down updates.

The setting per package manager:

**npm 11.10+** – `min-release-age` in `.npmrc` (value in days):

```ini
min-release-age=3
```

**pnpm 10.16+** – `minimumReleaseAge` in `pnpm-workspace.yaml` (value in minutes):

```yaml
minimumReleaseAge: 4320
```

**pnpm 11+** – defaults to `minimumReleaseAge: 1440` (1 day). The repo is already protected, but recommend overriding to a stronger value (e.g. `4320` for 3 days or `10080` for 7 days). Also verify the default has not been disabled with `minimumReleaseAge: 0`.

Also check if `.npmrc` contains a `minimumReleaseAge` setting – this is a common mistake. pnpm only reads this setting from `pnpm-workspace.yaml`, not `.npmrc` (and pnpm 11+ does not read any pnpm-specific settings from `.npmrc` at all). If found, warn the user and suggest moving it to `pnpm-workspace.yaml`.

**Yarn Berry 4.10+** – `npmMinimalAgeGate` in `.yarnrc.yml` (value in minutes):

```yaml
npmMinimalAgeGate: 4320
```

**Bun 1.3+** – `minimumReleaseAge` in `bunfig.toml` (value in seconds):

```toml
[install]
minimumReleaseAge = 259200
```

**Aube** – defaults to 1440 minutes (1 day). `minimumReleaseAge` in `aube-workspace.yaml` (value in minutes):

```yaml
minimumReleaseAge: 4320
```

### 2.6 Block exotic subdeps (pnpm / Aube)

**pnpm 10** – ensure `pnpm-workspace.yaml` contains:

```yaml
blockExoticSubdeps: true
```

**pnpm 11+ / Aube** – defaults to true. Verify it has not been explicitly disabled in `pnpm-workspace.yaml` / `aube-workspace.yaml`.

This prevents transitive dependencies from using git or tarball URLs.

### 2.7 Trust policy (pnpm / Aube)

**pnpm / Aube** – ensure `pnpm-workspace.yaml` or `aube-workspace.yaml` contains:

```yaml
trustPolicy: no-downgrade
```

This blocks packages whose trust level has decreased.

### 2.8 Hardened mode (Yarn Berry only)

If using Yarn Berry, ensure `.yarnrc.yml` contains:

```yaml
enableHardenedMode: true
```

This validates the lockfile against the registry.

### 2.9 Dependency update cooldown (Renovate / Dependabot)

Check if the repo uses Renovate or Dependabot and whether a cooldown is configured.

**Renovate** – look for config in `renovate.json`, `renovate.json5`, `.github/renovate.json`, `.github/renovate.json5`, `.renovaterc`, `.renovaterc.json`, or a `"renovate"` key in `package.json`. Check if the `extends` array includes any of the following (all of which include a 3-day npm cooldown):

- `"config:best-practices"`
- `"security:minimumReleaseAgeNpm"`
- `"NitorCreations/renovate-config:recommended"` (the Nitor recommended preset extends `config:best-practices`)

Also check shared presets – if `extends` references another custom preset (e.g. `"local>myorg/renovate-config"`), note that the cooldown may already be configured there and the user should verify.

If Renovate config exists but doesn't include any of the above presets, suggest adding `security:minimumReleaseAgeNpm`:

```json
{ "extends": ["security:minimumReleaseAgeNpm"] }
```

**Dependabot** – look for `.github/dependabot.yml`. Check if npm ecosystem entries have a `cooldown` configured. If not, suggest adding:

```yaml
cooldown:
  default-days: 3
```

If neither Renovate nor Dependabot is configured, skip this check – don't add a dependency update tool unprompted.

## Step 3 – Present findings

Present a summary of the audit results to the user:

1. Start with a one-line summary count (e.g. "3 passing, 4 need fixes")
2. List checks that already pass as bullet points (not numbered) – keep these brief
3. List checks that need fixes, with the specific changes that would be made
4. List any issues that require manual action (e.g. missing lockfile, `dangerouslyAllowAllBuilds`)
5. When multiple fixes target the same file, show a combined preview of the full proposed file content at the end
6. Do not use horizontal rules (`---`) – headings provide enough structure

IMPORTANT formatting rules – follow exactly:

- Use a numbered list for fixes. Put ALL explanation BEFORE the code block.
- Every proposed config change MUST be wrapped in a fenced code block using triple backticks and a language tag (e.g. yaml, json, toml, ini). RAW config text without triple-backtick fences is WRONG and hard to read.
- Indent code blocks with 3 spaces so they are nested inside the list item. This is critical – without the indent, the code block breaks out of the list and the numbering restarts (causing duplicate numbers like 1, 1, 2, 2).
- Leave a blank line before the opening triple backticks and after the closing triple backticks.

Then ask the user for confirmation before proceeding.

## Step 4 – Apply fixes

After the user confirms, apply the agreed-upon changes. If the user wants to skip certain fixes, respect that. Summarize what was changed.
