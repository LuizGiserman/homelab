# AGENTS.md

Instructions for AI coding agents (and humans) working in this repository.

## Git workflow

- **Never commit or push directly to `main`.** All changes go through a pull request.
- **Always create a new branch** for any feature, fix, or change. Do not reuse `main` or an existing unrelated branch.
- **Always use a git worktree** for a new branch of work, isolated from the primary checkout, rather than switching branches in place.
- **Always open pull requests as drafts** (`gh pr create --draft`). Never open a PR ready-for-review by default, and never merge a PR directly — leave merging to the repository owner.
- Never force-push to `main`, and never force-push a shared/published branch without explicit confirmation.
- Never use destructive git operations (`reset --hard`, `clean -f`, `branch -D`, etc.) without explicit confirmation, and always check `git status` first.

## Pull requests

- Keep PR titles short and descriptive; put details in the PR body.
- Include a summary of the change and a test plan/checklist in the PR body.
- Do not close or merge PRs on your own — that's a human decision.
