# Bash Command Reference

## Detect Base Branch

```bash
BASE=$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's|^origin/||')
if [ -z "$BASE" ]; then
  git show-ref --verify refs/remotes/origin/main &>/dev/null && BASE=main || BASE=master
fi
```

## Get GitHub Username

```bash
GH_USER=$(gh api user --jq '.login' | tr '[:upper:]' '[:lower:]')
```

## Push (tracking exists)

```bash
git push origin "$(git branch --show-current)"
```

## Push (new branch, no tracking)

```bash
git push --set-upstream origin "$(git branch --show-current)"
```

## Create PR

```bash
LABELS="<label matching commit type>"          # see label-mapping.md
REVIEWERS="<final reviewer list, comma-separated>"

gh pr create \
  --base "$BASE" \
  --title "<type>(<scope>): <description>" \
  --label "$LABELS" \
  --reviewer "$REVIEWERS" \
  --body "$(cat <<'EOF'
## Summary

- <bullet 1>
- <bullet 2>

## Motivation

<Why is this change needed?>

## How to Test

- [ ] <Testing step 1>
- [ ] <Testing step 2>

## Related Issue

closes #<issue-number> (if applicable)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Omit `--reviewer "$REVIEWERS"` if the reviewer list is empty.
