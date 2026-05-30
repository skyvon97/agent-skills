# Agent Identity Separation on GitHub

## Problem

Agents inherit the human operator's `gh` auth. GitHub can't distinguish human from agent actions. This breaks PR self-review restrictions, branch protection, CODEOWNERS, and audit trails.

## Goal

- Agent reviews come from a **different identity** than the PR author
- Contribution credit (green squares) stays with the **human operator**
- Different agents (Claude Code, Codex) are **distinguishable**

## Recommended Approach: GitHub Actions as Review Bridge

**Commits**: Human = author (green squares). Agent name in committer field for `git blame` differentiation.

**PRs**: Created under human identity as normal.

**Reviews**: Agent triggers a `workflow_dispatch` GitHub Action. The action posts the review using a short-lived GitHub App installation token generated from `AGENT_REVIEW_APP_ID` and `AGENT_REVIEW_APP_PRIVATE_KEY`. A separate bot account PAT in `AGENT_REVIEW_TOKEN` is also supported as a fallback. Review body includes a banner identifying the agent and workflow run.

**Why this works**: Author/PR-creator = human (contribution credit). Reviewer = the separate token owner (branch protection satisfied). Per-agent differentiation via review body and committer field.

The workflow's default `GITHUB_TOKEN` is not enough for approvals: GitHub rejects approval attempts from it. It is only useful for non-approving `COMMENT` smoke tests.

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
2. Create and install a GitHub App review identity. See `docs/github-app-review-identity.md`.
3. Store the app ID as a repository variable named `AGENT_REVIEW_APP_ID`.
4. Store the app private key as a repository Actions secret named `AGENT_REVIEW_APP_PRIVATE_KEY`.
5. Ensure repository Actions permissions allow workflows to create pull request reviews (`pull-requests: write` is declared in the workflow).
6. Optionally set committer env vars (`GIT_COMMITTER_NAME`) per agent in skill wrappers for commit attribution.

### Upgrade Path

If per-agent **visual identity** in GitHub UI matters later, register GitHub Apps per agent type. Each shows as `agent-name[bot]` with its own avatar. The workflow YAML becomes the app's handler — incremental migration, not a rewrite.

## Alternatives Considered

| Approach | Pros | Cons |
|----------|------|------|
| Dedicated bot account | Full API separation | Consumes a seat, co-author credit less prominent |
| Per-agent GitHub Apps | True per-agent identity, no seats | Complex setup, token refresh, overkill for solo dev |
| Platform-level agent identity | Most seamless | Doesn't exist yet (as of early 2026) |
