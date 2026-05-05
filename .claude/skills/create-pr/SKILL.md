---
name: create-pr
description: >-
  Create a GitHub pull request from the current branch. Detects and complies with
  the repository's pull request template (.github/PULL_REQUEST_TEMPLATE.md or
  variants) by filling each section with branch-specific content rather than
  using a generic format. Use whenever the user asks to "create a PR", "open a
  pull request", "raise a PR", or similar.
metadata:
  author: james.ly@luxuryescapes.com
  version: 1.0.0
---

# Create Pull Request

Create a GitHub pull request that **complies with the repository's PR template** when one exists. A generic summary/test-plan format is only acceptable when no template is present.

## Step 1 — Discover the PR template

Before drafting anything, look for a template in this order (first match wins):

1. `.github/PULL_REQUEST_TEMPLATE.md`
2. `.github/pull_request_template.md`
3. `docs/PULL_REQUEST_TEMPLATE.md`
4. `PULL_REQUEST_TEMPLATE.md` (repo root)
5. Any file under `.github/PULL_REQUEST_TEMPLATE/` (multiple templates — pick the one matching the change type, or ask the user)

```bash
ls .github/PULL_REQUEST_TEMPLATE* .github/pull_request_template* PULL_REQUEST_TEMPLATE* docs/PULL_REQUEST_TEMPLATE* 2>/dev/null
```

If a template exists, **read it in full**. Treat every heading, checkbox, and placeholder as a required section unless the template itself marks it optional.

## Step 2 — Gather branch context (parallel)

Run these in parallel via Bash:

- `git status` (no `-uall`)
- `git diff <base>...HEAD` — full diff vs. base branch
- `git log <base>..HEAD --oneline` — every commit on this branch, not just the latest
- `git rev-parse --abbrev-ref HEAD` — current branch name
- `gh pr view --json url 2>/dev/null` — confirm no PR already exists
- Check upstream tracking: `git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null`

The base branch is usually `main` or `master`. Verify with `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.

### Step 2a — If currently on the default branch, create a feature branch

If `git rev-parse --abbrev-ref HEAD` returns the default branch (`main` / `master`), you cannot open a PR from it. Create a new branch instead:

1. **Derive a branch name** from the change. Format: `<type>/<short-kebab-summary>` (e.g. `feat/add-create-pr-skill`, `fix/login-redirect-loop`, `chore/bump-deps`). Pick `<type>` from the change: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`. Keep the summary under ~40 chars.
2. **Confirm with the user** before creating — show the proposed name and ask. Skip confirmation only if the user explicitly said "just do it" or similar.
3. **Handle uncommitted work**:
   - If there are staged or unstaged changes, `git checkout -b <name>` carries them across — that's usually fine. Then ask the user whether to commit them now (and what message) before pushing.
   - If there are only untracked files that should be in the PR, create the branch, then `git add` + `git commit` with a message you draft from the diff (confirm message with the user first).
   - If the working tree is clean and there are no commits ahead of base, **stop** — there's nothing to PR. Tell the user.
4. **Never** create a branch and silently commit without telling the user what's being committed.

After the branch exists and has commits, continue with Step 3.

## Step 3 — Fill the template

For each section/heading in the template:

- **Replace placeholders** (e.g. `<!-- describe what this PR does -->`, `[ ] description`) with content drawn from the diff and commits.
- **Preserve structure exactly**: keep all headings, checkboxes, HTML comments (unless the comment is a placeholder instruction meant to be replaced), and ordering.
- **Checkboxes**: tick `[x]` only items the change actually satisfies. Leave unchecked items as `[ ]` — do not delete them.
- **Ticket/issue links**: extract from branch name (e.g. `feature/ABC-123-foo` → `ABC-123`) or commit messages. If a ticket field exists and no ID is found, leave the placeholder and flag it to the user.
- **Screenshots / testing sections**: if you cannot fill them (no UI changes, no test evidence), write "N/A — <reason>" rather than leaving blank or fabricating.
- **Do not invent** acceptance criteria, ticket numbers, reviewers, or testing steps you didn't actually verify.

If the template has multiple variants under `.github/PULL_REQUEST_TEMPLATE/`, ask the user which one applies before proceeding.

## Step 4 — No template fallback

Only when no template file exists, use this format:

```markdown
## Summary
- <1-3 bullets describing what changed and why>

## Test plan
- [ ] <testing step>
```

## Step 5 — Create the PR

1. Push the branch if needed (`git push -u origin <branch>` when no upstream is set).
2. Pass the body via heredoc to preserve formatting:

```bash
gh pr create --title "<short title, <70 chars>" --body "$(cat <<'EOF'
<filled template content>
EOF
)"
```

3. Return the PR URL from the `gh pr create` output to the user.

## Rules

- **Title**: short (<70 chars), imperative mood, no trailing period. Conventional-commit prefix (e.g. `feat:`, `fix:`) only if the repo's existing PR titles use them.
- **Never** skip hooks (`--no-verify`) or force-push unless the user explicitly asks.
- **Never** push to `main`/`master` directly — PRs always come from a feature branch.
- **Do not** add a "Generated with Claude Code" footer unless the repo's existing PRs include one or the user asks for it.
- **Do not** commit changes as part of creating the PR. If the working tree is dirty, ask the user first.
- If a PR already exists for this branch, surface its URL instead of creating a duplicate.

## Common pitfalls

- Summarising only the latest commit. Always review **every** commit on the branch.
- Stripping HTML comments that are part of the template structure (keep them; only replace placeholder instructions).
- Ticking every checkbox blindly — only tick what's actually done.
- Fabricating ticket IDs when the branch name doesn't contain one.
