# nitor-agent-skills

A collection of AI agent skills created by [Nitor](https://nitor.com), designed for use with [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [GitHub Copilot](https://github.com/features/copilot), and other compatible agents.

## Installation

Install skills using the [skills CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add nitorcreations/nitor-agent-skills
```

To list available skills before installing:

```sh
npx skills add nitorcreations/nitor-agent-skills --list
```

To update installed skills to their latest versions:

```sh
npx skills update
```

## Usage

Once installed, skills are automatically available to your agent. Invoke them by name (e.g. `/npm-supply-chain-audit`) or just describe what you need (e.g. "harden my project against supply-chain attacks").

## Available skills

- **npm-supply-chain-audit** – Audit and fix npm supply-chain security issues. Checks for missing protections (lockfile, lifecycle scripts, release-age cooldown, and more) and applies fixes after confirmation. Supports npm, pnpm, Yarn, Bun, and Aube
- **pr-review-response** – Respond to code-review comments on a GitHub PR end to end. Fetches unresolved threads, triages real reviewers from CI/bot noise, applies fixes or pushes back with reasoning, then replies referencing the fix commit and resolves each thread. Uses the gh CLI + GitHub GraphQL.

## Contributing

### Adding a new skill

1. Add a new skill directory with a `SKILL.md` file under `skills/`
   - You can use the [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) skill to help with this
2. Update the **Available skills** list in this README
3. Open a PR for review

For more on the skills format, see [agentskills.io](https://agentskills.io/).

### Testing a skill from a branch

You can install skills directly from a feature branch before merging:

```sh
npx skills add nitorcreations/nitor-agent-skills#your-branch
```
