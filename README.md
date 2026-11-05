# fix-dependency-pr

A Claude Code skill that diagnoses and fixes failing Renovate and Dependabot pull requests in DDEV projects. It reads the failed workflow's logs, classifies the cause (a patch that no longer applies, a config or schema update, a test regression, or something else), tests a fix locally with the failed job's own commands, and commits and pushes only after you approve.

An optional GitHub workflow runs the same diagnosis in CI and posts the proposed fix as a PR comment. It never pushes.

The design and its decisions are in [docs/brief.md](docs/brief.md).


## Requirements

- [Claude Code](https://claude.com/claude-code)
- [DDEV](https://ddev.com) in the project
- [GitHub CLI](https://cli.github.com) (`gh`), logged in to the project's GitHub host. The skill uses it to read PRs and workflow logs. If you run Claude Code inside DDEV with the [ddev-claude](https://github.com/marcoscano/ddev-claude) add-on, also install the GitHub CLI add-on there: `ddev add-on get plopesc/ddev-gh-cli`
- For CI: an `ANTHROPIC_API_KEY` secret


## Installation

As a Claude Code plugin:

```
/plugin install fix-dependency-pr --marketplace davereid/fix-dependency-pr
```

Or with [`npx skills`](https://github.com/vercel-labs/skills), which installs just the skill:

```
npx skills add https://github.com/davereid/fix-dependency-pr
```

Choose Claude Code when it asks which agents to install for. It installs the skill into `.claude/skills/fix-dependency-pr/` in the project, or `~/.claude/skills/` with `-g`. Update it later with `npx skills update`.

Or with Composer, which also provides the Taskfile tasks and the CI workflow template:

```
composer require --dev davereid/fix-dependency-pr
```

On Drupal sites, `drupal/core-composer-scaffold` can copy the skill into `.claude/skills/` once `davereid/fix-dependency-pr` is in `extra.drupal-scaffold.allowed-packages`.

Use one or the other for the skill, not both. Full steps, including credentials and CI: [skills/fix-dependency-pr/SETUP.md](skills/fix-dependency-pr/SETUP.md).


## Usage

In Claude Code:

```
/fix-dependency-pr 123
```

`/fix-dependency-pr all` works through every failing bot PR in turn, asking before each push. With no argument, it lists failing bot PRs and asks which one to fix, or all.

With Composer, include the Taskfile from the project's root `Taskfile.yml`:

```yaml
includes:
  deps: ./vendor/davereid/fix-dependency-pr/Taskfile.yml
```

| Task | What it does |
|---|---|
| `task deps:fix` | Starts Claude Code with the skill for every failing bot PR |
| `task deps:fix PR=123` | The same, for one PR |
| `task deps:install-skill` | Symlinks the skill into `.claude/skills/` |
| `task deps:install-workflow` | Copies the CI workflow into `.github/workflows/`, or shows a diff if it exists |


## What it will and won't do

- It only works on PRs whose branch **and** author match Renovate or Dependabot.
- It asks before every commit, push and PR comment.
- It makes at most 3 fix attempts per PR, then stops and drafts a comment.
- It never merges, force-pushes, pushes to other branches, or touches production or remote environments.
