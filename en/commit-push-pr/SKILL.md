---
name: commit-push-pr
description: |
  Commits code changes, pushes to remote branch, and creates or updates a GitHub Pull Request.
  Use after finishing a feature, bug fix, or refactor and wanting to submit changes.
  Trigger keywords (English): /ship, ship it, commit and push, push this, open a PR,
  create PR, submit changes, I'm done, push my changes, ready to merge, send it,
  wrap it up, commit push pr, done with this, time to ship, push and PR.
  Trigger keywords (Chinese): 幫我 commit、推上去、開 PR、送出變更、建立 PR、
  提交程式碼、push 到 remote、我做完了、送出去、上傳變更、交出去。
  Configure reviewers: set reviewer、設定 reviewer、update reviewers、
  change reviewers、更改預設 reviewer。
allowed-tools: Bash(git:*), Bash(gh:*)
---

# Commit → Push → PR Workflow

> For command details, see the references directory:
> - PowerShell commands → `references/commands-powershell.md`
> - Bash commands → `references/commands-bash.md`
> - Label mapping → `references/label-mapping.md`

## Trigger Commands

| Command | Effect |
|---------|--------|
| `/ship` | commit + push + ask whether to open PR |
| `/ship --pr` | commit + push + open PR directly |
| `/ship --no-pr` | commit + push only, skip PR |
| `/ship set-reviewers` | Update default reviewer list |

## Default Reviewers

Stored in `.claude/commit-push-pr.json`, empty by default.
Run `/ship set-reviewers` to configure.
Names appended after the command (e.g. `/ship --pr carol`) are added on top of the default list.

---

## Special Flow: Configure Default Reviewers

Triggered by: `/ship set-reviewers`, "set reviewer", "update reviewers", "change reviewers".
**Does not commit or push.**

1. Fetch collaborator list: `gh api repos/{owner}/{repo}/collaborators --jq '.[].login'`
2. Show current config; user selects from numbered list (enter `0` to clear all)
3. Update `defaultReviewers` in `.claude/commit-push-pr.json`
4. Confirm: `✅ Default reviewers updated: <list>`

---

## Steps

### 0. Detect Platform & Base Branch (run in parallel)

- Check for PowerShell (`$PSVersionTable`) to decide which command syntax to use
- Auto-detect base branch (see references for commands)
- Get GitHub username
- If detection fails, ask the user before continuing

### 1. Check Status (run in parallel)

Run simultaneously: `git status`, `git branch --show-current`,
`git log <base>..HEAD --oneline`, `git diff <base>...HEAD --stat`

If no changes exist, notify and stop.

### 2. Guard Checks

Refuse and explain if:
- ❌ Currently on the base branch → ask user to create a feature branch first
- ❌ No commits ahead of base branch → nothing to open a PR for

### 3. Analyze Changes

Read `git diff`, understand the purpose, group by logical concern (separate commits for different features or areas).

### 4. Determine Commit Type and Label

| Type | When to Use | PR Label |
|------|------------|----------|
| `feat` | New feature | `feat` |
| `fix` | Bug fix | `fix` |
| `style` | Style changes (no logic impact) | `style` |
| `refactor` | Refactor (no new feature, no bug fix) | `refactor` |
| `chore` | Config or dependency updates | `chore` |
| `docs` | Documentation updates | `docs` |
| `test` | Test-related changes | `test` |

Record the label for the primary type — it will be applied in Step 10.

### 5. Commit Message Format

```
<type>(<scope>): <short description>

<optional: explain WHY, not WHAT>
```

Rules: English description under 50 chars, lowercase scope, no period, split multiple concerns into separate commits.

### 6. Stage and Commit

```bash
git add <related files>
git commit -m "<type>(<scope>): <description>"
```

### 7. Determine Branch Name

Format: `<type>/<short-description>` (e.g. `feat/jwt-auth`, `fix/login-bug`)
Skip if already on a feature branch.

### 8. Push to Remote

1. `git branch -vv` to check remote tracking status
2. Tracking exists → push directly; no tracking → push with `--set-upstream`
3. ⚠️ VS Code "Publish Branch" button may be a UI sync issue — trust `git branch -vv`
4. Push fails → report reason and stop, do not proceed to PR

See `references/commands-powershell.md` or `references/commands-bash.md` for exact commands.

### 9. Ask Whether to Create a PR

If `--pr` / `--no-pr` not specified, ask after push:

```
✅ Push complete! Create a Pull Request? [Y/N]
```

- `N` → Skip to Step 11
- `Y` → Continue to Step 9a

### 9a. Validate and Confirm Reviewers

1. Read `defaultReviewers` from `.claude/commit-push-pr.json` (missing = empty)
2. If not empty, validate against collaborator list; remove invalid entries with a warning
3. Ask: `Add extra reviewers? [Y/N]`
4. If Y, show numbered collaborator list for multi-select; append to valid default list

### 10. Create Pull Request

Check for existing open PR first:
```bash
gh pr list --head <branch> --state open --json number,url
```

**PR exists** → Skip creation, show existing URL. Add reviewers via `gh pr edit --add-reviewer` if needed.
Notify: "PR already exists. This commit has been pushed to the same PR."

**No PR** → Create new PR with:
- `--label` matching the commit type (create label first if missing — see `references/label-mapping.md`)
- `--reviewer` final reviewer list (omit if empty)
- PR body with: Summary, Motivation, How to Test, Related Issue, 🤖 Claude Code footer

See `references/commands-powershell.md` or `references/commands-bash.md` for exact commands.

### 11. Output Summary

| Item | Detail |
|------|--------|
| ✅ Commit | `<type>(<scope>): <description>` |
| ✅ Push | `<branch> → origin/<branch>` |
| ✅ PR | `<PR URL>` (existing / newly created) |
| ✅ Label | `<type label>` |
| ✅ Reviewer | `<final list>` (or "none") |

---

## Notes

- Never use `git push --force` or `git commit --amend` unless explicitly requested
- Base branch auto-detected in Step 0; ask user if it fails
- Refuse to run on the base branch
- If a PR already exists, update it — never create a duplicate
- Never auto-merge — let the user decide
- Report pre-commit hook failures and wait for instruction
