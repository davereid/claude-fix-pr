---
name: fix-dependency-pr
description: Diagnose and fix a failing Renovate or Dependabot pull request in a DDEV project (patches that no longer apply, config or schema updates, dependency conflicts, test regressions), test the fix locally with the failed workflow's own commands, and push it only after the user approves. Use when asked to fix a failing Renovate/Dependabot/dependency PR, or via /fix-dependency-pr [PR number | all].
---

# Fix dependency PR

Arguments: a PR number, `all`, or nothing.

- A number: fix that PR.
- `all`: fix every open PR that has failing checks and matches the bot rules, one after another, oldest first. List them first. Then do steps 2–10 for each PR, with its own approvals and 3-attempt limit. Each PR starts from a clean checkout. After the last one, summarize each PR as fixed, given up (with the drafted comment), skipped or refused.
- Nothing: list the matching PRs and ask which one, or `all`.

## Rules

- Never merge, approve, close or edit a PR's title/body/labels; never force-push; never push to any branch except the matched PR's head branch.
- Never touch production or remote environments: no drush site aliases other than the local site, no `sql-sync`/`rsync` from remotes, no `ddev pull`, no Terminus or hosting CLIs.
- Ask the user before every commit + push and before every PR comment. An approval covers only that action on that PR.
- At most 3 fix-and-retest attempts. One PR per run unless the user asks for more (`all` or a list). The user can skip a PR, or stop the whole run, at any approval prompt. Don't wait for CI after pushing.
- When unsure, stop and ask.

## Allowed tools

- `gh`: `pr view`, `pr diff`, `pr checks`, `pr list`, `run view` (incl. `--log-failed`), `pr comment` (after approval), `api` with GET only. Nothing else.
- `git`: `fetch`, `checkout`/`switch` of the PR branch, `status`, `diff`, `log`, `add`, `commit`, `push` (no force).
- Drupal patches: the `drupalorg-cli` skill or `drupalorg` read commands, and `curl` GET requests to `git.drupalcode.org` and `www.drupal.org`.
- DDEV: `ddev start`, `ddev describe`, `ddev composer`, `ddev exec`, `ddev drush` (local site), `ddev task`, and any custom `ddev <command>` the failed workflow runs. Inside a DDEV container, the same commands without the `ddev` prefix.

## Environment

- **Inside a DDEV container** if `IS_DDEV_PROJECT=true`: run commands without `ddev`.
- **CI** if `CI=true`: comment-only mode (see below).
- Otherwise on the host: prefix commands with `ddev`.
- Host and repo: from `git remote get-url origin`. Set `GH_HOST` to that host for `gh`.

## First run checks

Before anything else, run only these non-mutating checks. If one fails, stop and tell the user how to fix it.

1. `gh auth status --hostname <host>`; if not logged in: `gh auth login --hostname <host>`.
2. `ddev --version` and `ddev describe` (host only); if not running: `ddev start`.
3. `git status --porcelain --untracked-files=no` is empty (untracked files are fine). Otherwise ask the user to commit or stash their changes first.

## Steps

1. **Pick the PR.** If no number was given, list open PRs whose checks fail and that match the bot rules (`gh pr list --json number,title,headRefName,author,statusCheckRollup`), and ask which one.
2. **Check the bot rules.** Both the branch and the author must match, or stop:
   - Renovate: the branch starts with `renovate/` or the `branchPrefix` from the project's Renovate config (`renovate.json`, `.github/renovate.json`, `renovate.json5`, `.renovaterc`); author is `renovate[bot]` or the bot named in `FIX_DEPENDENCY_PR_RENOVATE_BOT`.
   - Dependabot: the branch starts with `dependabot/`; author is `dependabot[bot]`.
3. **Read the failure.** `gh pr checks`, then `gh run view <run-id> --log-failed`. Open the failed workflow's YAML to see the exact commands it ran. Read `gh pr diff`.
4. **Classify** the cause:
   - A. A patch no longer applies. Check whether it's merged upstream (then remove it) or needs a re-roll (then find an updated patch in the upstream issue).
     - Drupal patches (`extra.patches` in `composer.json` or a patches file): take the issue number from the patch's description or URL. If the `drupalorg-cli` skill is available, use it. Otherwise, if `drupalorg` is installed, run `drupalorg issue:show <nid> --format=llm` and `drupalorg mr:list` to check whether the fix is in the new release and get the latest MR diff or patch. Otherwise use the GitLab API on `git.drupalcode.org` (`/api/v4/projects/project%2F<project>/issues/<nid>` and `/merge_requests`). Use a patch from a fixed URL or a local file, never a moving MR `.diff` URL. If nothing applies, re-roll the patch and say so in the PR comment.
   - B. A config or schema update. Run the project's update/deploy steps (Drupal: `drush updatedb`, then `drush config:export`) and keep the resulting changes.
   - C. A test regression. Find what the update changed and fix the project code or tests.
   - D. Something else, e.g. a dependency conflict or a changed constraint.

   Tell the user the cause before changing anything.
5. **Fix.** `git fetch origin <branch>` and check out the PR branch. Start DDEV, install the site as the project's README/docs describe, and make the smallest change that fixes the cause.
6. **Test** by running the failed job's commands through DDEV. If they fail, revise and retest. After the 3rd failed attempt, stop and draft a PR comment (what failed, what you tried, the likely cause). Ask before posting it.
7. **Ask for approval.** Show the cause, `git diff`, and the test output.
8. **Commit and push** after approval. Follow the project's commit convention (contributing docs, recent `git log`). Push to the PR's head branch only.
9. **Comment** after a separate confirmation: the cause, the fix and the tests run. For Dependabot, add: "Dependabot will no longer rebase this PR because it has commits from someone else."
10. Give the user the PR's checks URL and stop.

## CI mode (comment only)

Do steps 2–6 without asking. Never commit, push or comment. Write a Markdown report to `fix-dependency-pr-comment.md` in the repository root, under 60 lines, containing:

- the cause and its class (A–D)
- the proposed diff in a ```diff block
- the commands run and whether they passed
- "Run `/fix-dependency-pr <n>` in Claude Code (or `task <include>:fix PR=<n>`) to apply this fix locally."

The workflow posts it.
