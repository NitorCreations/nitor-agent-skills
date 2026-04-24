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
  - Bash(cat *)
  - Bash(node *)
  - Bash(npm *)
  - Bash(pnpm *)
  - Bash(yarn *)
  - Bash(bun *)
  - Bash(git *)
---

# npm supply-chain security audit

Audit the current repo for npm supply-chain security issues, present findings to the user, and apply fixes only after confirmation.

## Step 1 — Detect the package manager

Check which package manager(s) are in use:

- `package-lock.json` -> npm
- `aube-lock.yaml` -> Aube
- `pnpm-lock.yaml` -> pnpm
- `yarn.lock` + `.yarnrc.yml` -> Yarn Berry; `yarn.lock` without `.yarnrc.yml` -> Yarn Classic
- `bun.lockb` or `bun.lock` -> Bun
- `packageManager` field in `package.json` confirms the PM and version

If no lockfile exists, default to npm.

## Step 2 — Audit

Work through the checks below. For each one, check the current state and record whether it passes or needs a fix. Do NOT make any changes yet.

### 2.1 Lockfile committed

If no lockfile exists (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, `bun.lock`), warn the user they should run their package manager's install command and commit the lockfile. Do NOT run install yourself.

### 2.2 Block lifecycle scripts

**npm** — ensure `.npmrc` contains:

```ini
ignore-scripts=true
```

**pnpm 10+** — blocks lifecycle scripts by default. Check that `pnpm-workspace.yaml` does NOT set `dangerouslyAllowAllBuilds: true`. If it does, warn the user.

**pnpm <= 9** — ensure `.npmrc` contains `ignore-scripts=true`.

**Yarn Classic (v1)** — ensure `.npmrc` contains `ignore-scripts=true`.

**Yarn Berry (v2+)** — ensure `.yarnrc.yml` contains:

```yaml
enableScripts: false
```

**Bun** — blocks lifecycle scripts by default. No action needed.

**Aube** — blocks lifecycle scripts by default. Check that `aube-workspace.yaml` does NOT set `dangerouslyAllowAllBuilds: true`. If it does, warn the user.

### 2.3 Build script allowlists

When lifecycle scripts are blocked, some packages legitimately need build scripts (e.g. `esbuild`, `sharp`, `better-sqlite3`, `@swc/core`, `bcrypt`, `canvas`). Check the dependency tree and warn if known packages that require build scripts are installed but not allowlisted.

**pnpm 10+ / Aube** — check `allowBuilds` (or `onlyBuiltDependencies`) in `pnpm-workspace.yaml` / `aube-workspace.yaml`. If packages that need builds are in the dependency tree but not listed, warn the user and suggest adding them:

```yaml
onlyBuiltDependencies:
  - esbuild
```

**Yarn Berry** — check `dependenciesMeta` in `package.json`. If packages that need builds are installed but not allowlisted, warn the user:

```json
"dependenciesMeta": {
  "esbuild": { "built": true }
}
```

**Bun** — check `trustedDependencies` in `package.json`. If packages that need builds are installed but not allowlisted, warn the user:

```json
"trustedDependencies": ["esbuild"]
```

Do NOT automatically add allowlist entries — just warn and suggest. The user should audit each package before trusting it.

### 2.4 Release-age cooldown

Check if a release-age cooldown is configured. If not, recommend adding one. If already configured, note the current value. When presenting findings, advise the user that 3 days is a good minimum. Mention that 7 days provides stronger protection but slows down updates.

The setting per package manager:

**npm 11.10+** — `min-release-age` in `.npmrc` (value in days):

```ini
min-release-age=3
```

**pnpm 10.16+** — `minimumReleaseAge` in `pnpm-workspace.yaml` (value in minutes):

```yaml
minimumReleaseAge: 4320
```

Also check if `.npmrc` contains a `minimumReleaseAge` setting — this is a common mistake. pnpm only reads this setting from `pnpm-workspace.yaml`, not `.npmrc`. If found, warn the user and suggest moving it to `pnpm-workspace.yaml`.

**Yarn Berry 4.10+** — `npmMinimalAgeGate` in `.yarnrc.yml` (value in minutes):

```yaml
npmMinimalAgeGate: 4320
```

**Bun 1.3+** — `minimumReleaseAge` in `bunfig.toml` (value in seconds):

```toml
[install]
minimumReleaseAge = 259200
```

**Aube** — defaults to 1440 minutes (1 day). `minimumReleaseAge` in `aube-workspace.yaml` (value in minutes):

```yaml
minimumReleaseAge: 4320
```

### 2.5 Block exotic subdeps (pnpm / Aube)

**pnpm** — ensure `pnpm-workspace.yaml` contains:

```yaml
blockExoticSubdeps: true
```

**Aube** — defaults to true. Verify it has not been explicitly disabled in `aube-workspace.yaml`.

This prevents transitive dependencies from using git or tarball URLs.

### 2.6 Trust policy (pnpm / Aube)

**pnpm / Aube** — ensure `pnpm-workspace.yaml` or `aube-workspace.yaml` contains:

```yaml
trustPolicy: no-downgrade
```

This blocks packages whose trust level has decreased.

### 2.7 Hardened mode (Yarn Berry only)

If using Yarn Berry, ensure `.yarnrc.yml` contains:

```yaml
enableHardenedMode: true
```

This validates the lockfile against the registry.

### 2.8 Dependency update cooldown (Renovate / Dependabot)

Check if the repo uses Renovate or Dependabot and whether a cooldown is configured.

**Renovate** — look for config in `renovate.json`, `renovate.json5`, `.github/renovate.json`, `.github/renovate.json5`, `.renovaterc`, `.renovaterc.json`, or a `"renovate"` key in `package.json`. Check if `"config:best-practices"` is in the `extends` array (this preset includes a 3-day cooldown for npm). Also check shared presets — if `extends` references a custom preset (e.g. `"local>myorg/renovate-config"`), note that the cooldown may already be configured there and the user should verify.

If Renovate config exists but doesn't extend `config:best-practices` or a shared preset, suggest adding it:

```json
{ "extends": ["config:best-practices"] }
```

**Dependabot** — look for `.github/dependabot.yml`. Check if npm ecosystem entries have a `cooldown` configured. If not, suggest adding:

```yaml
cooldown:
  default-days: 3
```

If neither Renovate nor Dependabot is configured, skip this check — don't add a dependency update tool unprompted.

## Step 3 — Present findings

Present a summary of the audit results to the user:

- List checks that already pass (no action needed).
- List checks that need fixes, with the specific changes that would be made to each file.
- List any issues that require manual action (e.g. missing lockfile, `dangerouslyAllowAllBuilds`).

Then ask the user for confirmation before proceeding.

## Step 4 — Apply fixes

After the user confirms, apply the agreed-upon changes. If the user wants to skip certain fixes, respect that. Summarize what was changed.
