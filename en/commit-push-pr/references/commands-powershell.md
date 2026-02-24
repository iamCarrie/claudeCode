# PowerShell Command Reference

## Detect Base Branch

```powershell
$base = (git rev-parse --abbrev-ref origin/HEAD 2>$null) -replace '^origin/', ''
if (-not $base) {
  if (git show-ref --verify refs/remotes/origin/main 2>$null) { $base = 'main' }
  elseif (git show-ref --verify refs/remotes/origin/master 2>$null) { $base = 'master' }
  else { Write-Error 'Could not determine base branch. Please specify it manually.' }
}
```

## Get GitHub Username

```powershell
$ghUser = (gh api user --jq '.login').ToLower()
```

## Push (tracking exists)

```powershell
git push origin (git branch --show-current)
```

## Push (new branch, no tracking)

```powershell
git push --set-upstream origin (git branch --show-current)
```

## Create PR

```powershell
$labels    = "<label matching commit type>"   # see label-mapping.md
$reviewers = "<final reviewer list, comma-separated>"
$body = @"
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
"@
gh pr create `
  --base $base `
  --title "<type>(<scope>): <description>" `
  --label $labels `
  --reviewer $reviewers `
  --body $body
```

Omit `--reviewer $reviewers` if the reviewer list is empty.
