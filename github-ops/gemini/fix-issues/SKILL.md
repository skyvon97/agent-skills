---
name: fix-issues
description: "GitHub, docs: triage, review, proof."
argument-hint: "[issue-numbers or focus area]"
disable-model-invocation: true
---

# Fix Issues — Audit Issue Resolution

You are resolving GitHub issues from automated code audits. Work surgically, document thoroughly.

## Repository Detection

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

Use `$REPO` for all `gh` commands below.

$ARGUMENTS

## Before You Start

- Confirm the repo is on the default branch, usually `main`.
- `git fetch --prune`.
- Check for local dirt before assuming a clean starting point.
- If the task depends on upstream state, refresh before editing.

## Step 1: Find Issues

```bash
gh issue list --repo "$REPO" --label ready-to-fix --state open --json number,title,body,labels
```

If no `ready-to-fix` issues exist, check `needs-triage`:
```bash
gh issue list --repo "$REPO" --label needs-triage --state open --json number,title,body,labels
```

## Step 2: Select 3-5 Issues

- **Group related issues** affecting the same file/module into one PR
- **Prioritize:** security > reliability > shipability > navigability
- **Confidence matters.** If you're not sure about the fix, skip it.
- **Check for dependencies.** If issue A must be fixed before B, take A.

## Step 3: Implement

**Branch:**
```bash
git checkout -b fix/issue-<NUM>-<short-desc>
# or for grouped issues:
git checkout -b fix/issues-<NUM>-<NUM>-<short-desc>
```

**Commits and PR:** Read `../../references/commit-and-pr-format.md` for commit message format and PR body template. For smaller fix PRs, the "Issues Reviewed but Not Addressed", "Risk", and "Notes" sections are optional.

**Build check:**
```bash
if [ -f "package.json" ]; then
  npm run build
elif [ -f "Cargo.toml" ]; then
  cargo build
elif [ -f "Makefile" ]; then
  make
elif [ -f "go.mod" ]; then
  go build ./...
elif [ -f "pyproject.toml" ]; then
  pip install -e . 2>/dev/null || python -m build
else
  echo "No recognized build system. Skipping build verification."
fi
```
Fix any errors before proceeding.

## Step 4: Update Labels

```bash
# For each issue being fixed:
gh issue edit <NUMBER> --repo "$REPO" --add-label in-progress --remove-label ready-to-fix
```

## Standards

Read `../../references/severity-definitions.md` for severity criteria.

Read `../../references/four-dimensions.md` for the ideal state framework. Every fix should move the codebase toward ideal state.

## Principles

Read `../../references/operating-principles.md` and follow all principles listed there.
