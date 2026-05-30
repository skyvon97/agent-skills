# agent-skills

Reusable AI agent skill files for Claude Code, Codex (OpenAI), and Gemini CLI. Designed to be installed into individual project repositories.

## Structure

```
<category>/
├── claude/          # Claude Code skills (.claude/skills/ format)
│   └── <skill>/SKILL.md
├── codex/           # Codex skills (.agents/skills/ format)
│   └── <skill>/
│       ├── SKILL.md
│       └── agents/openai.yaml
└── gemini/          # Gemini CLI skills (.gemini/skills/ format)
    └── <skill>/SKILL.md
.github/workflows/
└── agent-pr-review.yml  # GitHub Actions review identity for review-pr
```

## Categories

### `github-ops/`

GitHub issue lifecycle skills — audit, triage, fix, and review.

| Skill | Purpose |
|-------|---------|
| `audit` | Full-cycle: triage all open audit issues, resolve actionable ones, produce report |
| `triage` | Classify `needs-triage` issues into ready-to-fix, deferred, or rejected |
| `fix-issues` | Resolve `ready-to-fix` issues with atomic commits and PRs |
| `review-pr` | Review PRs through a GitHub Actions review identity, merge if approved, handle post-merge hygiene |

## Installation

See the **[Installation Guide](docs/installation.md)** for full instructions, including a **copy-paste prompt** you can give to your coding agent to have it install the skills automatically.

**Quick start — paste this into your agent:**

> Install the github-ops AI agent skills into this project from https://github.com/skylordafk/agent-skills.git — clone it to /tmp, copy Claude Code skills to .claude/skills/, Codex skills to .agents/skills/, Gemini CLI skills to .gemini/skills/, shared references to all three reference directories, and .github/workflows/agent-pr-review.yml to .github/workflows/. Then add .claude/ and .agents/ to .geminiignore to silence warnings. Finally clean up the clone and verify with ls -R.

### Repository auto-detection

Skills use `gh repo view --json nameWithOwner` to detect the current repository at runtime. No per-project configuration needed — just install and go.

`review-pr` also requires `.github/workflows/agent-pr-review.yml`. The skill triggers it with `gh workflow run` so approvals and change requests are posted by `github-actions[bot]` instead of the human operator's `gh` token.

## Adding new skills

1. Create a new category directory (or use an existing one)
2. Add `claude/`, `codex/`, and/or `gemini/` subdirectories
3. Write `SKILL.md` files following each agent's format conventions
4. For Codex skills, include `agents/openai.yaml` with invocation policy

## Agent format differences

| Feature | Claude Code | Codex | Gemini CLI |
|---------|-------------|-------|------------|
| Location | `.claude/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` | `.gemini/skills/<name>/SKILL.md` |
| Frontmatter | `name`, `description`, `argument-hint`, `allowed-tools`, `disable-model-invocation` | `name`, `description` | `name`, `description` |
| Arguments | `$ARGUMENTS` variable | Passed via invocation context | Passed via conversation context |
| Activation | Slash commands or model matching | Skill picker or implicit | Model-driven via `activate_skill` |
| Config sidecar | None | `agents/openai.yaml` | None |
| Context file | `CLAUDE.md` | `AGENTS.md` | `GEMINI.md` |
