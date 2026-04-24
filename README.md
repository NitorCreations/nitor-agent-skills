# nitor-agent-skills

> **Note:** This repository is not yet public but will be made public soon.

A collection of AI agent skills created by [Nitor](https://nitor.com), designed for use with [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [GitHub Copilot](https://github.com/features/copilot), and other compatible agents.

## Installation

Install skills using the [skills CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add nitor/nitor-agent-skills
```

To list available skills before installing:

```sh
npx skills add nitor/nitor-agent-skills --list
```

## Usage

Once installed, skills are automatically available to your agent. You can invoke them by name (e.g. `/npm-supply-chain-audit`) or just describe what you need — the agent will pick the right skill based on your prompt.

## Available skills

- **npm-supply-chain-audit** — Audit and fix npm supply-chain security issues in the current repo. Detects the package manager, checks for missing protections (lockfile, lifecycle script blocking, release-age cooldown, and more), and applies fixes after user confirmation. Supports npm, pnpm, Yarn, Bun, and Aube.

## Contributing

### Adding a new skill

1. Add a new skill directory with a `SKILL.md` file under `skills/`.
   - You can use the [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) skill to help with this.
2. Update the **Available skills** list in this README.
3. Open a PR for review.

For more on the skills format, see [agentskills.io](https://agentskills.io/).

### Testing a skill from a branch

You can install skills directly from a feature branch before merging:

```sh
npx skills add nitor/nitor-agent-skills#your-branch
```
