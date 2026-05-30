# Agent Identity Separation on GitHub

## Problem

Agents inherit the human operator's `gh` auth. GitHub can't distinguish human from agent actions. This breaks PR self-review restrictions, branch protection, CODEOWNERS, and audit trails.

## Goal

- Agent reviews come from a **different identity** than the PR author
- Contribution credit (green squares) stays with the **human operator**
- Different agents (Claude Code, Codex) are **distinguishable**

## Recommended Approach: GitHub Actions as Review Identity

**Commits**: Human = author (green squares). Agent name in committer field for `git blame` differentiation.

**PRs**: Created under human identity as normal.

**Reviews**: Agent triggers a `workflow_dispatch` GitHub Action. The action posts the review via `GITHUB_TOKEN`, appearing as `github-actions[bot]`. Review body includes a banner identifying the agent.

**Why this works**: Author/PR-creator = human (contribution credit). Reviewer = `github-actions[bot]` (branch protection satisfied). Per-agent differentiation via review body and committer field.

### Implemented Entity

This repository includes `.github/workflows/agent-pr-review.yml`.

It accepts:

- `pr_number` — target pull request number
- `verdict` — `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`
- `body_b64` — base64-encoded Markdown review body
- `head_sha` — optional expected PR head SHA, used to fail if the PR moved
- `agent_name` — human-readable agent identity included in the review body

The `review-pr` skill triggers it with `gh workflow run`, waits for the review marker to appear on the PR, and only then merges an approved PR.

### Remaining Setup

1. Install `.github/workflows/agent-pr-review.yml` into each target repository.
2. Ensure repository Actions permissions allow workflows to create pull request reviews (`pull-requests: write` is declared in the workflow).
3. Optionally set committer env vars (`GIT_COMMITTER_NAME`) per agent in skill wrappers for commit attribution.

### Upgrade Path

If per-agent **visual identity** in GitHub UI matters later, register GitHub Apps per agent type. Each shows as `agent-name[bot]` with its own avatar. The workflow YAML becomes the app's handler — incremental migration, not a rewrite.

## Alternatives Considered

| Approach | Pros | Cons |
|----------|------|------|
| Dedicated bot account | Full API separation | Consumes a seat, co-author credit less prominent |
| Per-agent GitHub Apps | True per-agent identity, no seats | Complex setup, token refresh, overkill for solo dev |
| Platform-level agent identity | Most seamless | Doesn't exist yet (as of early 2026) |
