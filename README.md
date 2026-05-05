# AI-setup

Project-level Claude Code configuration: permissions, MCP servers, and custom skills.

## Layout

```
.claude/
├── settings.json          # Shared, committed settings (permissions + skill overrides)
└── skills/
    └── create-pr/
        └── SKILL.md       # Custom skill: PR creation that respects repo PR templates
```

## `settings.json`

Committed defaults applied to every Claude Code session run from this repo.

- **`skillOverrides`** — forces `create-pr` on so the skill is always available.
- **`permissions.allow`** — pre-approves common read-only and dev-loop commands (`Read`, `Grep`, `npm run *`, `git status`/`diff`/`log`, `gh pr create *`, `Edit`, etc.) so they don't prompt.
- **`permissions.deny`** — hard-blocks risky operations: reading `.env*`/secrets, writing to `.github/workflows/*`, `rm -rf`, `sudo`, force pushes, pushes to `main`/`master`, `npm publish`, `docker`, `curl | sh`.

## Skills

### `create-pr`

Opens a GitHub PR from the current branch. Before drafting it:

1. Looks for a PR template (`.github/PULL_REQUEST_TEMPLATE.md` and common variants).
2. If found, fills each section from the branch diff and commit log — preserving headings, checkboxes, and HTML comments — instead of using a generic summary.
3. If not found, falls back to a simple `## Summary` / `## Test plan` body.

It also creates a feature branch if you're on `main`/`master`, confirms with you before committing dirty work, and surfaces an existing PR's URL rather than duplicating.

Trigger phrases: "create a PR", "open a pull request", "raise a PR".

## Using this setup in another repo

Copy `.claude/settings.json` (and optionally the `skills/` directory) into the target repo. Adjust the allow/deny lists to match that project's tooling.
