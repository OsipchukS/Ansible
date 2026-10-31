---
name: git-pr-workflow
description: Git workflow for this repository. Create a feature branch from master, commit with English messages, push the branch, and open a GitHub Pull Request via the GitHub MCP server. Use when the user asks to commit, push, create a branch, or open a pull request. Never push to master.
---

# Git PR Workflow

Base branch: `master`. Remote: `origin` (GitHub).

## Hard rules

- Never run `git push` to `master`, and never push directly to `master`.
- Always work in a feature branch and open a Pull Request.
- Ask for approval before every git command that changes state (`checkout -b`, `add`, `commit`, `push`) and before creating the PR via GitHub MCP tools.
- Never commit secrets: `*.pem`, keys, tokens, passwords.

## Steps

1. Read-only checks (no approval needed):
   ```bash
   git status
   git diff
   git log --oneline -10
   git remote -v
   ```
2. Show the planned branch name and list of files. Wait for approval.
3. Create a branch from up-to-date `master`:
   ```bash
   git checkout -b feature/<short-english-description>
   ```
4. Stage only the intended files by name (avoid `git add -A`).
5. Commit with an English message in imperative mood, e.g. `Add base packages role`.
6. Show the commit and wait for approval, then push the branch:
   ```bash
   git push -u origin feature/<short-english-description>
   ```
7. Open the PR after approval using the GitHub MCP server. In Code Mode the tool is `tools.github.create_pull_request`:
   - `owner`/`repo`: taken from `git remote -v` (origin).
   - `base`: `master`.
   - `head`: the feature branch name.
   - `title` and `body`: English, imperative mood.
8. Return the PR URL to the user.

## GitHub MCP

- The `github` MCP server is configured globally in OpenCode (`https://api.githubcopilot.com/mcp/`). It authenticates with a fine-grained PAT sent in the `Authorization` header from the `GITHUB_MCP_PAT` environment variable. The token never enters this repository or any command line.
- GitHub's remote MCP server does not support OAuth dynamic client registration, so the `/mcps` sign-in flow does not work for it.
- If the server is not connected (for example `opencode mcp list` reports an authentication error), ask the user to verify `GITHUB_MCP_PAT` and restart the OpenCode background service. Do not use the `gh` CLI as a replacement.
- The current PAT has read-only contents permission, so pull requests are merged by the user in the GitHub web interface. Do not attempt to merge a PR through MCP tools unless the token explicitly has `Contents: Read and write`.

## Before creating a PR

- Review all commits on the branch, not only the last one: `git log master..HEAD --oneline`.
- Check for a pull request template (`.github/pull_request_template.md` or `.github/PULL_REQUEST_TEMPLATE/`) and use it to structure the PR body when present.
- Confirm `ansible-playbook site.yml --syntax-check` passes.
