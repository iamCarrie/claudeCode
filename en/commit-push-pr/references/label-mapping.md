# PR Label Mapping

When creating a PR, automatically apply a label based on the primary commit type.

## Type → Label Mapping

| Commit Type | GitHub Label | Description |
|-------------|--------------|-------------|
| `feat`      | `feat`       | New feature |
| `fix`       | `fix`        | Bug fix |
| `style`     | `style`      | Style changes (no logic impact) |
| `refactor`  | `refactor`   | Code refactor |
| `chore`     | `chore`      | Config or dependency updates |
| `docs`      | `docs`       | Documentation updates |
| `test`      | `test`       | Test-related changes |

## Execution Logic

### Step 1: Check if label exists

Before creating the PR, verify the label exists in the repo:

```bash
gh label list --repo {owner}/{repo} --json name --jq '.[].name'
```

### Step 2: Create label if missing

```bash
gh label create "feat"     --color "#0075ca" --description "New feature"
gh label create "fix"      --color "#e4e669" --description "Bug fix"
gh label create "style"    --color "#e99695" --description "Style changes"
gh label create "refactor" --color "#d93f0b" --description "Code refactor"
gh label create "chore"    --color "#cccccc" --description "Config or dependency updates"
gh label create "docs"     --color "#0075ca" --description "Documentation updates"
gh label create "test"     --color "#bfd4f2" --description "Test-related changes"
```

### Step 3: Apply label when creating PR

```bash
--label "feat"
```

## Notes

- If commits span multiple types, use the label for the type with the most changes
- For `feat!` or commits with `BREAKING CHANGE`, also add a `breaking-change` label
- Label colors are suggestions only — feel free to update them in the GitHub UI
