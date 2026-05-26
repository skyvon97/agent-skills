---
name: triage
description: "GitHub: triage, review, proof."
---

# Issue Triage — Classification & Routing

Review issues tagged `needs-triage` and classify them. You are the quality gate between audit sweep and fix pipeline.

## Repository Detection

Determine the target repository:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

If the project spans multiple repos, identify them from the project structure and triage both unless a specific repo is requested.

## Before You Start

Verify required labels exist. Read `../../references/label-taxonomy.md` and run the bootstrap script to create any missing labels.

## Process

1. **Fetch issues:**
   ```bash
   gh issue list --repo "$REPO" --label needs-triage --state open --json number,title,body,labels
   ```

2. **Evaluate each issue:**
   - Is this real? (verify by reading the referenced code — don't trust the audit agent blindly)
   - Does it matter? Read `../../references/four-dimensions.md` for the framework.
   - What's the risk trajectory? Read `../../references/risk-potential.md`. Is this a latent defect (pattern known to breed bugs) or an artificial constraint (design choice that unnecessarily narrows future options)? Either is a valid reason to promote an issue that would otherwise be rejected.
   - Is it actionable? (specific fix, completable in ~1 hour)
   - Is it a duplicate or symptom of something larger?
   - **Calibration:** assess project stage — reject theoretical risks, style nits in admin scripts, and anything requiring env compromise to exploit

3. **Classify into one disposition:**

   **→ `ready-to-fix`** — Real, scoped, actionable.
   Remove `needs-triage`, add `ready-to-fix`. Add acceptance criteria if missing.
   Issues with genuine risk potential qualify even if they'd otherwise seem like style nits or over-engineering — the calibration criteria in the risk-potential framework are the gatekeeper.

   **→ `deferred`** — Valid but not now.
   Remove `needs-triage`, add `deferred`. Comment with reason and unblock criteria.

   **→ Close (reject)** — Not real, intentional behavior, duplicate, or style nit.
   Close with rationale comment. Add `audit-rejected`.

4. **Auto-triage rules** (apply without manual review):
   - CRITICAL + `fix` → `ready-to-fix`
   - HIGH + `fix` → `ready-to-fix`
   - Exact title match with closed issue → close as duplicate
   - `risk:hazard` label → always manual review (never auto-triaged)

5. **Output summary** with disposition counts and pattern notes.

## Severity

Read `../../references/severity-definitions.md` for severity criteria.

## Principles

- You are the quality gate. Don't let junk into `ready-to-fix`.
- Rejection is a feature. Bad issues waste fix time.
- When in doubt, defer.
- Surface patterns — if the same low-value issue keeps appearing, note it.
- Connect the dots — group symptoms under umbrella issues.
