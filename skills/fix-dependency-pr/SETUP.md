# Setup

You connect accounts and grant access yourself; the skill can't.

## Local

**Plugin only:** `/plugin install fix-dependency-pr --marketplace davereid/fix-dependency-pr`, then do steps 4–5 below. Composer and the Taskfile are only needed for `task deps:*` and the CI workflow. Don't also run `install-skill`, or the skill is loaded twice.

1. `composer require --dev davereid/fix-dependency-pr`
2. Add to the project's root `Taskfile.yml`:
   ```yaml
   includes:
     deps: ./vendor/davereid/fix-dependency-pr/Taskfile.yml
   ```
3. `task deps:install-skill`. Commit the symlink `.claude/skills/fix-dependency-pr` to share it with the team, or ignore it in git.
4. GitHub: create a fine-grained token for this repository only, with Contents: read/write, Pull requests: read/write and Actions: read. Then run `gh auth login --hostname <your GitHub host>`.
5. Check that `git push` to the repo works with your SSH key (and signing, if required).
6. Run it with `task deps:fix PR=123`, or `/fix-dependency-pr 123` in Claude Code.

**Unverified: ddev-claude container** (`marcoscano/ddev-claude`):

- Allowlist your GitHub host in the add-on's firewall.
- Pass a `gh` token into the container, for example as `GH_TOKEN`.
- Make sure git can push from the container, and that commit signing works there if your repo requires it.

## CI (comment only)

1. `task deps:install-workflow`, then edit `workflows:` under `workflow_run` to list the workflows to watch.
2. If the project installs DDEV through its own action, replace the `setup-ddev` step with it.
3. Add `ANTHROPIC_API_KEY` as an **Actions secret** and a **Dependabot secret**. If it isn't set, the workflow exits early. For client projects, the client must own or approve this key.
4. **Unverified:**
   - Confirm that your GitHub host allows `anthropics/claude-code-action` and `marocchino/sticky-pull-request-comment`. GitHub Enterprise may block marketplace actions.5. Commit the workflow file.
