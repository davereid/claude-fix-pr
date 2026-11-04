# fix-dependency-pr

A Claude Code skill that diagnoses and fixes failing Renovate and Dependabot pull requests in DDEV projects. It reads the failed workflow's logs, classifies the cause (a patch that no longer applies, a config or schema update, a test regression, or something else), tests a fix locally with the failed job's own commands, and commits and pushes only after you approve.

An optional GitHub workflow runs the same diagnosis in CI and posts the proposed fix as a PR comment. It never pushes.

The design and its decisions are in [docs/brief.md](docs/brief.md).


## Requirements

- [Claude Code](https://claude.com/claude-code)
- [DDEV](https://ddev.com) in the project
- [GitHub CLI](https://cli.github.com) (`gh`), logged in to the project's GitHub host
- For CI: an `ANTHROPIC_API_KEY` secret


## Installation

As a Claude Code plugin:

```
/plugin install fix-dependency-pr --marketplace davereid/fix-dependency-pr
```

Or with Composer, which also provides the Taskfile tasks and the CI workflow template:

```
composer require --dev davereid/fix-dependency-pr
```

Use one or the other for the skill, not both. Full steps, including credentials and CI: [skills/fix-dependency-pr/SETUP.md](skills/fix-dependency-pr/SETUP.md).


## Usage

In Claude Code:

```
/fix-dependency-pr 123
```

With no PR number, it lists failing bot PRs and asks which one to fix.

With Composer, include the Taskfile from the project's root `Taskfile.yml`:

```yaml
includes:
  deps: ./vendor/davereid/fix-dependency-pr/Taskfile.yml
```

| Task | What it does |
|---|---|
| `task deps:fix PR=123` | Starts Claude Code with the skill for that PR |
| `task deps:install-skill` | Symlinks the skill into `.claude/skills/` |
| `task deps:install-workflow` | Copies the CI workflow into `.github/workflows/`, or shows a diff if it exists |


## What it will and won't do

- It only works on PRs whose branch **and** author match Renovate or Dependabot.
- It asks before every commit, push and PR comment.
- It makes at most 3 fix attempts per PR, then stops and drafts a comment.
- It never merges, force-pushes, pushes to other branches, or touches production or remote environments.
