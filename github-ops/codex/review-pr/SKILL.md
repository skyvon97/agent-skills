---
name: review-pr
description: "GitHub, docs: triage, review, proof."
---

# Review + Merge — PR Review Pipeline

You are the review and merge authority for this PR. Review the fix, make the merge decision, and handle all post-merge cleanup. You are the last line of defense before code enters main.

If the operator requested `--dry-run`, complete the review and report the intended verdict, but do not post reviews, merge, or comment on GitHub.

## Repository Detection

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

Provide the PR number when invoking this skill.

## Step 1: Load PR

```bash
gh pr view <PR_NUMBER> --repo "$REPO" --comments --json number,title,body,author,files,commits,labels,baseRefName,headRefName,headRefOid,comments,reviews,latestReviews,closingIssuesReferences,statusCheckRollup,mergeable,mergeStateStatus,isDraft,state,url
gh pr diff <PR_NUMBER> --repo "$REPO"
```

Read the PR description, linked issues, labels, existing comments, and the full diff.
If you are not the first reviewer, treat prior review comments as required context.
If the same human account authors and reviews PRs, follow `../../docs/agent-identity-separation.md`.

Before local verification, make sure the worktree is clean:

```bash
git status --short
```

If `git status --short` prints anything, stop and resolve with the operator before checking out the PR. If clean, check out the PR:

```bash
gh pr checkout <PR_NUMBER> --repo "$REPO"
```

## Step 2: Structural Check

Verify the PR meets standards. Flag gaps, but do not block on minor template issues:

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

## Step 4: Verification

Check required GitHub checks:

```bash
gh pr checks <PR_NUMBER> --repo "$REPO" --required
```

If required checks are pending and you intend to merge, wait for them:

```bash
gh pr checks <PR_NUMBER> --repo "$REPO" --required --watch --fail-fast
```

Run the relevant local verification. If there is no clearer repo-specific command, use this fallback:

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

If required checks, tests, or build fail, this is BLOCKING regardless of code quality.

## Step 5: Verdict

**APPROVE** — No BLOCKING findings, structural checks mostly pass, fix is correct.
**REQUEST CHANGES** — Any BLOCKING finding. Post your review and stop.
**REJECT** — Fix introduces regression, contains unrelated changes, or can't be reviewed.

Use a review body file:

```bash
REVIEW_BODY=$(mktemp)
$EDITOR "$REVIEW_BODY"
```

If this is a dry run, print the review body and intended action instead of mutating GitHub.

## Step 6: If Approved — Merge

```bash
HEAD_SHA=$(gh pr view <PR_NUMBER> --repo "$REPO" --json headRefOid -q .headRefOid)
PR_AUTHOR=$(gh pr view <PR_NUMBER> --repo "$REPO" --json author -q .author.login)
CURRENT_USER=$(gh api user -q .login)
if [ "$PR_AUTHOR" != "$CURRENT_USER" ]; then
  gh pr review <PR_NUMBER> --repo "$REPO" --approve --body-file "$REVIEW_BODY"
else
  gh pr comment <PR_NUMBER> --repo "$REPO" --body-file "$REVIEW_BODY"
fi
gh pr merge <PR_NUMBER> --repo "$REPO" --merge --delete-branch --match-head-commit "$HEAD_SHA"
```

Use merge commit (not squash) to preserve full history for auditability.
If the repository requires a merge queue, omit the merge strategy and let `gh pr merge` queue the PR.
If `PR_AUTHOR` equals `CURRENT_USER`, do not attempt an approval review. GitHub rejects self-approval with `GraphQL: Review cannot approve your own pull request`; treat that as an expected platform rule, skip the approval command, and do not announce it as a noteworthy event. Record the approved review body as a normal PR comment, then continue directly to merge after required checks. Do not block just because formal approval is unavailable. Only report a blocker if `gh pr merge` itself fails because repository branch protection requires approval from a different GitHub account.

If requesting changes:

```bash
gh pr review <PR_NUMBER> --repo "$REPO" --request-changes --body-file "$REVIEW_BODY"
```

## Step 7: Post-Merge Hygiene

This is not optional.

Derive resolved issues from GitHub's linked closing references:

```bash
gh pr view <PR_NUMBER> --repo "$REPO" --json closingIssuesReferences -q '.closingIssuesReferences[].number'
```

For each resolved issue:

```bash
gh issue comment <NUMBER> --repo "$REPO" --body "Resolved in PR #<PR_NUMBER>. Fixed by: <one-line summary>. Dimension improved: <shipability/reliability/navigability/security>"
```

Post merge summary as a PR comment using a body file:

```bash
SUMMARY_BODY=$(mktemp)
cat > "$SUMMARY_BODY" <<EOF
## Merge Summary
**Date:** $(date +%Y-%m-%d)
**Issues resolved:** #<NUM>
**Build:** Passing
**Branch:** Deleted
EOF

gh pr comment <PR_NUMBER> --repo "$REPO" --body-file "$SUMMARY_BODY"
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
if [ "$PR_AUTHOR" != "$CURRENT_USER" ]; then
  gh pr review <PR_NUMBER> --repo "$REPO" --approve --body-file "$REVIEW_BODY"
else
  gh pr comment <PR_NUMBER> --repo "$REPO" --body-file "$REVIEW_BODY"
fi
# or
gh pr review <PR_NUMBER> --repo "$REPO" --request-changes --body-file "$REVIEW_BODY"
```

## Principles

1. **Last line of defense, not a rubber stamp.** Read the code. Form your own opinion.
2. **Every closed issue needs a paper trail.** Link to the fix, say what changed.
3. **Hygiene is a core deliverable.** Comments, linked issues, branch cleanup.
4. **Err toward REQUEST CHANGES.** Bad merge > another review cycle.
5. **BLOCKING means real production risk.** Not "I'd do it differently."
