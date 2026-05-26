---
name: triage
description: "GitHub: triage, review, proof."
argument-hint: "[--auto-only] [--repo <owner/repo>]"
disable-model-invocation: true
allowed-tools: Read, Grep, Bash, ListDirectory
---

# Issue Triage — Classification & Routing

You review issues tagged `needs-triage` and determine which are worth fixing, which should be deferred, and which should be rejected. You are the quality gate between the audit sweep and the fix pipeline.

## Repository Detection

Determine the target repository:

```bash
# If --repo was specified in arguments, use that
# Otherwise, auto-detect from the current project:
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

If the project spans multiple repos (e.g., a website + API), identify them from the project structure and triage both unless `--repo` narrows the scope.

$ARGUMENTS

## Before You Start

- Confirm the repo is on the default branch (usually `main`).
- `git fetch --prune`.
- Check for local dirt before assuming a clean starting point.
- If the work depends on upstream state, refresh before editing.
- Verify required labels exist. Read `../../references/label-taxonomy.md` and run the bootstrap script to create any missing labels.

## Step 1: Fetch Issues

```bash
gh issue list --repo "$REPO" --label needs-triage --state open --json number,title,body,labels
```

## Step 2: Evaluate Each Issue

For each issue, assess:

### Is this a real finding?
- Does it describe a concrete, verifiable problem?
- Could the audit agent have misunderstood the code?
- If unsure, **read the referenced file/lines** to verify independently.

### Does it matter?
- Which dimension does this affect? Read `../../references/four-dimensions.md` for the framework.
- How significant is the impact? Would fixing this meaningfully close a gap?

### What's the risk trajectory?
- Read `../../references/risk-potential.md` for the framework.
- Is this a latent defect — a pattern with a known track record of producing bugs?
- Is this an artificial constraint — a design choice that unnecessarily narrows future options?
- If neither, move on. If either, this is a valid reason the issue deserves attention regardless of whether it's "broken" today.

### Is it actionable?
- Is there a specific, bounded fix? (Not "consider refactoring the architecture")
- Can it be completed in under ~1 hour of focused work?
- Are there prerequisites or dependencies?

### Is it a duplicate or symptom?
- Does an existing open issue already cover this?
- Is this a symptom of a deeper problem? Does an umbrella issue exist?

### Calibration
- Assess the project's stage. Early-stage projects prioritize pragmatism over perfection.
- One-off admin/dev scripts have a much higher bar for "worth fixing" — don't fix style nits in scripts only devs run.
- Focus on changes that prevent real problems, not theoretical ones.
- If the "attack" requires already owning the environment, it's not a real finding.

## Step 3: Classify

Assign each issue ONE disposition:

### → `ready-to-fix`
Real issue, clearly scoped, actionable, impacts a dimension.

Actions:
- Remove `needs-triage`, add `ready-to-fix`
- If the issue body is missing acceptance criteria, add them in a comment
- Issues with genuine risk potential (per the framework) qualify as `ready-to-fix` even if they would otherwise seem like style nits or over-engineering. The risk-potential calibration criteria are the gatekeeper.

### → `deferred`
Valid concern but not actionable now.

Reasons: needs architectural decision, shared root cause needing umbrella issue, too large for pipeline, low priority.

Actions:
- Remove `needs-triage`, add `deferred`
- Comment with: why deferred, what would unblock it, related issues
- If symptom of larger problem: also add `blocked`, create or link umbrella issue

### → Close (reject)
Not worth tracking.

Reasons: audit agent misread the code, intentional behavior, disproportionate fix, pure style nit, duplicate.

Actions:
- Close the issue
- Comment with: why rejected, one-line rationale
- Add `audit-rejected` label

## Step 4: Execute & Summarize

Execute all label changes and comments, then output:

```markdown
## Triage Summary — YYYY-MM-DD

**Repo:** <repo(s)>
**Issues reviewed:** <count>

| Disposition | Count | Issues |
|-------------|-------|--------|
| ready-to-fix | X | #1, #2, #3 |
| deferred | X | #4, #5 |
| rejected | X | #6, #7 |

### Notes
- <patterns observed, umbrella issues created, audit tuning recommendations>
```

## Auto-Triage Rules

Apply automatically without manual review:

- **CRITICAL + type `fix`** → `ready-to-fix`
- **HIGH + type `fix`** → `ready-to-fix`
- **Exact title match with a closed issue** → close as duplicate
- **`risk:hazard` label** → always manual review (never auto-triaged, regardless of severity)

Use `--auto-only` to apply only these rules.

## Severity

Read `../../references/severity-definitions.md` for severity criteria.

## Principles

1. **You are the quality gate.** `ready-to-fix` issues must be real, scoped, and worth the effort.
2. **Rejection is a feature.** Closing bad issues keeps the backlog clean.
3. **When in doubt, defer.** A deferred issue can be reconsidered. A bad `ready-to-fix` wastes fix time.
4. **Surface patterns.** If the sweep keeps creating the same low-value issue type, note it.
5. **Connect the dots.** Group symptoms under umbrella issues.
