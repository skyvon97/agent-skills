---
name: fix-issues
description: "GitHub, docs: triage, review, proof."
---

# Fix Issues — Audit Issue Resolution

Resolve one operator-provided GitHub issue. This skill is normally paired with `review-pr`: fix the issue, open the PR, then hand off for review.

## Repository Detection

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
```

Use `$REPO` for all `gh` commands below.

## Before You Start

- Require one issue number or URL from the operator. If none was supplied, ask for it; optionally show recent open issues:
  ```bash
  gh issue list --repo "$REPO" --state open --limit 20 --json number,title,url
  ```
- Do not depend on issue status labels; the issue number is the assignment.
- Start from a clean, current default branch:
  ```bash
  git status --short
  ```
  If `git status --short` prints anything, stop and resolve with the operator before proceeding.
  ```bash
  git fetch --prune origin
  git switch "$DEFAULT_BRANCH"
  git pull --ff-only origin "$DEFAULT_BRANCH"
  ```

## Step 1: Load the Issue

```bash
gh issue view <ISSUE> --repo "$REPO" --comments --json number,title,body,state,comments,closedByPullRequestsReferences,url
```

Verify:

- The issue is open.
- The issue is not already fixed by an open or merged PR.
- The requested fix is clear enough to implement safely.

If adjacent problems appear, mention them in PR notes or file a follow-up. Do not expand this PR unless the operator explicitly asks.

## Step 2: Start Work

Create a linked branch from the default branch:

```bash
gh issue develop <ISSUE> --repo "$REPO" --checkout --base "$DEFAULT_BRANCH" --name "fix/issue-<NUM>-<short-desc>"
```

If `gh issue develop` is unavailable or blocked by permissions:

```bash
git switch -c "fix/issue-<NUM>-<short-desc>"
```

Read `../../references/commit-and-pr-format.md` before committing or opening the PR.

## Step 3: Implement

- Make the smallest change that resolves the supplied issue.
- Avoid drive-by refactors and unrelated cleanup.
- If the issue cannot be fixed confidently, comment with the blocker and stop.
- Use structured parsers or repo-local helpers instead of brittle text manipulation when available.

## Step 4: Verify

Run the repo's relevant tests, lint, and build. If there is no clear project-specific command, use this fallback:

```bash
if [ -f "package.json" ]; then
  npm test -- --watch=false 2>/dev/null || npm test
  npm run build
elif [ -f "Cargo.toml" ]; then
  cargo test
  cargo build
elif [ -f "Makefile" ]; then
  make
elif [ -f "go.mod" ]; then
  go test ./...
  go build ./...
elif [ -f "pyproject.toml" ]; then
  if python -m pytest --version >/dev/null 2>&1; then
    python -m pytest
  fi
  pip install -e . 2>/dev/null || python -m build
else
  echo "No recognized build system. Run the most relevant repo-local verification manually."
fi
```

Fix failures before opening the PR. Record exactly what passed and what could not be run.

## Step 5: Commit and Open PR

Commit one logical change using the reference format, push, and create a PR with a body file:

```bash
git add <FILES>
git commit
git push -u origin HEAD

PR_BODY=$(mktemp)
cat > "$PR_BODY" <<'EOF'
## Summary
<brief description of the issue and fix>

## Issues Resolved
- Closes #<NUM> — <one-line description>

## Changes
- **File:** `<path>` — <what changed>

## Verification
- [x] Latest code pulled
- [x] <test/lint/build command>
- [x] No unrelated changes
EOF

gh pr create --repo "$REPO" --base "$DEFAULT_BRANCH" --head "$(git branch --show-current)" --title "[Audit Fix] <summary>" --body-file "$PR_BODY"
```

The commit message must include `Resolves #<NUM>` and the PR body must include `Closes #<NUM>` so `review-pr` can find the linked issue.

## Standards

Read `../../references/severity-definitions.md` for severity criteria.

Read `../../references/four-dimensions.md` for the ideal state framework. Every fix should move the codebase toward ideal state.

## Principles

Read `../../references/operating-principles.md` and follow all principles listed there.
