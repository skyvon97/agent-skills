---
name: review-pr
description: "GitHub, docs: triage, review, proof."
---

# Review + Merge — PR Review Pipeline

You are the review and merge authority for this PR. Review the fix, make the merge decision, and handle all post-merge cleanup. You are the last line of defense before code enters main.

## Repository Detection

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

Provide the PR number when invoking this skill.

## Step 1: Load PR

```bash
gh pr view <PR_NUMBER> --json number,title,body,author,files,commits,labels,baseRefName,headRefName
gh pr diff <PR_NUMBER>
```

Read the PR description, linked issues, labels, existing comments, and the full diff.
If you are not the first reviewer, treat prior review comments as required context.
If the same human account authors and reviews PRs, follow `../../docs/agent-identity-separation.md`.

## Step 2: Structural Check

Verify the PR meets standards:

- [ ] Branch name follows `fix/issue-<NUM>-<desc>` convention
- [ ] PR title is descriptive (`[Audit Fix] <summary>` for audit fixes)
- [ ] Resolved issues listed with `Closes #XX` syntax
- [ ] Commit messages are descriptive and reference issue numbers
- [ ] No unrelated changes in the diff

## Step 3: Code Review

For each changed file, evaluate:

**Correctness** — Does the change fix the stated problem? Edge cases handled?
**Scope** — All changes related to stated issues? Nothing extra snuck in?
**Side effects** — Could this break existing functionality? Trace call sites of modified functions.
**Ideal state** — Does this move toward one of the four dimensions? Read `../../references/four-dimensions.md` for the framework.

Classify each finding:
- **BLOCKING** — must fix before merge (real bugs, regressions, security issues)
- **SUGGESTION** — recommended but not required
- **NOTE** — informational only

## Step 4: Build Verification

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

If the build fails, this is BLOCKING regardless of code quality.

## Step 5: Verdict

**APPROVE** — No BLOCKING findings, structural checks mostly pass, fix is correct.
**REQUEST CHANGES** — Any BLOCKING finding. Post your review and stop.
**REJECT** — Fix introduces regression, contains unrelated changes, or can't be reviewed.

## Step 6: If Approved — Merge

```bash
gh pr merge <PR_NUMBER> --merge --delete-branch
```

Use merge commit (not squash) to preserve full history for auditability.

## Step 7: Post-Merge Hygiene

This is not optional.

**For each resolved issue:**
```bash
gh issue comment <NUMBER> --repo "$REPO" --body "Resolved in PR #<pr>. Fixed by: <one-line summary>. Dimension improved: <shipability/reliability/navigability/security>"

gh issue edit <NUMBER> --repo "$REPO" --add-label audit-resolved --remove-label ready-to-fix --remove-label needs-triage --remove-label in-progress
```

**Post merge summary as PR comment:**
```bash
gh pr comment <PR_NUMBER> --body "## Merge Summary
**Date:** $(date +%Y-%m-%d)
**Issues resolved:** #XX, #YY
**Build:** Passing
**Branch:** Deleted
**Labels:** Updated"
```

## Review Comment Format

```markdown
## PR Review: <title>

### Verdict: APPROVE / REQUEST CHANGES / REJECT

### Structural Check
<pass/fail summary>

### Findings
#### [BLOCKING/SUGGESTION/NOTE] <title>
- **File:** `path/to/file` (line X)
- **Issue:** what's wrong
- **Suggestion:** what to do

### Summary
<1-2 sentences>
```

Post the review:
```bash
gh pr review <PR_NUMBER> --approve --body "<review text>"
# or
gh pr review <PR_NUMBER> --request-changes --body "<review text>"
```

## Principles

1. **Last line of defense, not a rubber stamp.** Read the code. Form your own opinion.
2. **Every closed issue needs a paper trail.** Link to the fix, say what changed.
3. **Hygiene is a core deliverable.** Labels, comments, branch cleanup.
4. **Err toward REQUEST CHANGES.** Bad merge > another review cycle.
5. **BLOCKING means real production risk.** Not "I'd do it differently."
