---
name: review-pr
description: "GitHub, docs: triage, review, proof."
argument-hint: "<pr-number> [--dry-run]"
disable-model-invocation: true
allowed-tools: Read, Grep, Bash, ListDirectory
---

# Review + Merge — PR Review Pipeline

You are the review and merge authority for this PR. Review the fix, make the merge decision, and handle all post-merge cleanup. You are the last line of defense before code enters main.

## Repository Detection

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

Use `$REPO` for all `gh` commands below.

If `$ARGUMENTS` includes `--dry-run`, set `DRY_RUN=true`, remove that flag before passing arguments to `gh`, and do not post reviews, merge, edit labels, or comment on GitHub.

```bash
DRY_RUN=false
PR_ARGS="$ARGUMENTS"
if echo " $ARGUMENTS " | grep -q " --dry-run "; then
  DRY_RUN=true
  PR_ARGS=$(echo "$ARGUMENTS" | sed 's/[[:space:]]*--dry-run//g')
fi
```

## Step 1: Load PR

```bash
gh pr view $PR_ARGS --json number,title,body,author,files,commits,labels,baseRefName,headRefName
gh pr diff $PR_ARGS
```

Read the PR description, linked issues, labels, existing comments, and the full diff.
If you are not the first reviewer, treat prior review comments as required context.
If the same human account authors and reviews PRs, follow `../../docs/agent-identity-separation.md`.

## Step 2: Structural Check

Verify the PR meets standards. Flag gaps but don't block on minor template issues:

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
PR_AUTHOR=$(gh pr view $PR_ARGS --json author -q .author.login)
CURRENT_USER=$(gh api user -q .login)
if [ "$PR_AUTHOR" != "$CURRENT_USER" ]; then
  gh pr review $PR_ARGS --approve --body "<review text>"
else
  gh pr comment $PR_ARGS --body "<review text>"
fi
gh pr merge $PR_ARGS --merge --delete-branch
```

Use merge commit (not squash) to preserve full history for auditability.
In dry-run mode, report that you would approve and merge instead of running this command.
When `PR_AUTHOR` equals `CURRENT_USER`, publish the approved review body as a normal PR comment instead of an approval review, then continue directly to merge after required checks. Only stop if `gh pr merge` fails.

## Step 7: Post-Merge Hygiene

This is not optional. A merged PR with dangling labels is incomplete work.

**For each resolved issue:**
```bash
# Add closing comment with context
gh issue comment <NUMBER> --repo "$REPO" --body "Resolved in PR #<pr>. Fixed by: <one-line summary>. Dimension improved: <shipability/reliability/navigability/security>"

# Update labels
gh issue edit <NUMBER> --repo "$REPO" --add-label audit-resolved --remove-label ready-to-fix --remove-label needs-triage --remove-label in-progress
```

**Post merge summary as PR comment:**
```bash
gh pr comment $PR_ARGS --body "## Merge Summary
**Date:** $(date +%Y-%m-%d)
**Issues resolved:** #XX, #YY
**Build:** Passing
**Branch:** Deleted
**Labels:** Updated"
```

## Review Comment Format

When posting your review (whether approving or requesting changes):

```markdown
## PR Review: <title>

### Verdict: APPROVE / REQUEST CHANGES / REJECT

### Structural Check
<pass/fail summary — keep it brief>

### Findings
#### [BLOCKING/SUGGESTION/NOTE] <title>
- **File:** `path/to/file` (line X)
- **Issue:** what's wrong
- **Suggestion:** what to do

### Summary
<1-2 sentences: what's good, what needs attention>
```

Post the review:
```bash
if [ "$PR_AUTHOR" != "$CURRENT_USER" ]; then
  gh pr review $PR_ARGS --approve --body "<review text>"
else
  gh pr comment $PR_ARGS --body "<review text>"
fi
# or
gh pr review $PR_ARGS --request-changes --body "<review text>"
```

In dry-run mode, print the review text and intended action instead of running `gh pr review`.

## Principles

1. **Last line of defense, not a rubber stamp.** Read the code. Form your own opinion.
2. **Every closed issue needs a paper trail.** Link to the fix, say what changed.
3. **Hygiene is a core deliverable.** Labels, comments, branch cleanup.
4. **Err toward REQUEST CHANGES.** Bad merge > another review cycle.
5. **BLOCKING means real production risk.** Not "I'd do it differently."
