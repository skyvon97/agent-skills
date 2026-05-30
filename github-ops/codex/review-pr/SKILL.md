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
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
AGENT_REVIEW_WORKFLOW=agent-pr-review.yml
```

Provide the PR number when invoking this skill.
Use `$REPO` for all `gh` commands below.

## Step 1: Load PR

```bash
gh pr view <PR_NUMBER> --repo "$REPO" --comments --json number,title,body,author,files,commits,labels,baseRefName,headRefName,headRefOid,comments,reviews,latestReviews,closingIssuesReferences,statusCheckRollup,mergeable,mergeStateStatus,isDraft,state,url
gh pr diff <PR_NUMBER> --repo "$REPO"
```

Read the PR description, linked issues, labels, existing comments, and the full diff.
If you are not the first reviewer, treat prior review comments as required context.
Reviews are posted through the `Agent PR Review` GitHub Actions workflow using a separate review identity. Prefer the GitHub App setup with the `AGENT_REVIEW_APP_ID` repository variable and `AGENT_REVIEW_APP_PRIVATE_KEY` secret. A separate bot account PAT in `AGENT_REVIEW_TOKEN` is also supported. GitHub does not allow the workflow's default `GITHUB_TOKEN` to approve pull requests.

Verify the workflow exists before posting a review:

```bash
gh workflow view "$AGENT_REVIEW_WORKFLOW" --repo "$REPO"
```

If the workflow is missing, stop and install `.github/workflows/agent-pr-review.yml` from this repository before reviewing. If neither GitHub App secrets nor `AGENT_REVIEW_TOKEN` are configured, stop and configure a separate review identity before approving or requesting changes. Do not fall back to `gh pr review` from the human operator account for approvals or requested changes.

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
REQUEST_ID="agent-review-$(date +%s)-$$"
printf '\n\n<!-- agent-review-request: %s -->\n' "$REQUEST_ID" >> "$REVIEW_BODY"
REVIEW_BODY_B64=$(base64 < "$REVIEW_BODY" | tr -d '\n')

gh workflow run "$AGENT_REVIEW_WORKFLOW" --repo "$REPO" --ref "$DEFAULT_BRANCH" \
  --raw-field pr_number="<PR_NUMBER>" \
  --raw-field verdict="APPROVE" \
  --raw-field body_b64="$REVIEW_BODY_B64" \
  --raw-field head_sha="$HEAD_SHA" \
  --raw-field agent_name="Codex review-pr"

POSTED_STATE=""
for _ in $(seq 1 24); do
  POSTED_STATE=$(gh pr view <PR_NUMBER> --repo "$REPO" --json reviews -q ".reviews[] | select(.body | contains(\"$REQUEST_ID\")) | .state" | tail -n 1)
  if [ "$POSTED_STATE" = "APPROVED" ]; then
    break
  fi
  sleep 5
done
if [ "$POSTED_STATE" != "APPROVED" ]; then
  gh run list --repo "$REPO" --workflow "$AGENT_REVIEW_WORKFLOW" --limit 5
  echo "Timed out waiting for agent approval review to post." >&2
  exit 1
fi

gh pr merge <PR_NUMBER> --repo "$REPO" --merge --delete-branch --match-head-commit "$HEAD_SHA"
```

Use merge commit (not squash) to preserve full history for auditability.
If the repository requires a merge queue, omit the merge strategy and let `gh pr merge` queue the PR.
Because the review is posted by GitHub Actions, do not switch to a normal PR comment when the PR author matches the human operator. Only stop if the workflow review fails to post or `gh pr merge` fails.

If requesting changes:

```bash
HEAD_SHA=$(gh pr view <PR_NUMBER> --repo "$REPO" --json headRefOid -q .headRefOid)
REQUEST_ID="agent-review-$(date +%s)-$$"
printf '\n\n<!-- agent-review-request: %s -->\n' "$REQUEST_ID" >> "$REVIEW_BODY"
REVIEW_BODY_B64=$(base64 < "$REVIEW_BODY" | tr -d '\n')

gh workflow run "$AGENT_REVIEW_WORKFLOW" --repo "$REPO" --ref "$DEFAULT_BRANCH" \
  --raw-field pr_number="<PR_NUMBER>" \
  --raw-field verdict="REQUEST_CHANGES" \
  --raw-field body_b64="$REVIEW_BODY_B64" \
  --raw-field head_sha="$HEAD_SHA" \
  --raw-field agent_name="Codex review-pr"

POSTED_STATE=""
for _ in $(seq 1 24); do
  POSTED_STATE=$(gh pr view <PR_NUMBER> --repo "$REPO" --json reviews -q ".reviews[] | select(.body | contains(\"$REQUEST_ID\")) | .state" | tail -n 1)
  if [ "$POSTED_STATE" = "CHANGES_REQUESTED" ]; then
    break
  fi
  sleep 5
done
if [ "$POSTED_STATE" != "CHANGES_REQUESTED" ]; then
  gh run list --repo "$REPO" --workflow "$AGENT_REVIEW_WORKFLOW" --limit 5
  echo "Timed out waiting for agent request-changes review to post." >&2
  exit 1
fi
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
REVIEW_VERDICT=APPROVE  # or REQUEST_CHANGES
gh workflow run "$AGENT_REVIEW_WORKFLOW" --repo "$REPO" --ref "$DEFAULT_BRANCH" \
  --raw-field pr_number="<PR_NUMBER>" \
  --raw-field verdict="$REVIEW_VERDICT" \
  --raw-field body_b64="$REVIEW_BODY_B64" \
  --raw-field head_sha="$HEAD_SHA" \
  --raw-field agent_name="Codex review-pr"
```

## Principles

1. **Last line of defense, not a rubber stamp.** Read the code. Form your own opinion.
2. **Every closed issue needs a paper trail.** Link to the fix, say what changed.
3. **Hygiene is a core deliverable.** Comments, linked issues, branch cleanup.
4. **Err toward REQUEST CHANGES.** Bad merge > another review cycle.
5. **BLOCKING means real production risk.** Not "I'd do it differently."
