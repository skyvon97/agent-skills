# Installing GitHub-Ops Skills

This guide explains how to add the github-ops skills to any project directory. It includes a prompt you can paste directly into your coding agent (Claude Code, Codex, or Gemini CLI) to have it install the skills automatically.

## What You Get

Four skills for GitHub issue lifecycle management:

| Skill | What it does |
|-------|--------------|
| **audit** | Full-cycle issue triage and resolution with a summary report |
| **triage** | Classify `needs-triage` issues into ready-to-fix, deferred, or rejected |
| **fix-issues** | Resolve `ready-to-fix` issues with atomic commits and PRs |
| **review-pr** | Review pull requests, merge if approved, handle post-merge cleanup |

Plus six shared reference documents that the skills depend on (label taxonomy, severity definitions, quality dimensions, risk potential, commit/PR format, operating principles) and one GitHub Actions workflow used by `review-pr` to post reviews through a separate review identity.

## Prerequisites

- **`gh` CLI** — installed and authenticated (`gh auth status`)
- **`git`** — installed
- **Separate review identity** — required for `review-pr` approvals or requested changes. Prefer a GitHub App configured with `AGENT_REVIEW_APP_ID` and `AGENT_REVIEW_APP_PRIVATE_KEY` (see `docs/github-app-review-identity.md`). A separate bot account PAT in `AGENT_REVIEW_TOKEN` is also supported. GitHub does not allow the workflow's default `GITHUB_TOKEN` to approve PRs.

For a new repository, install `.github/workflows/agent-pr-review.yml`, install your GitHub App on that repository, and store the App ID/private key as that repository's Actions variable and secret. One private App can be reused across repositories you control, but each repository must have the workflow and the App installed. See `docs/github-app-review-identity.md#using-this-in-another-repository`.

## Agent-Executable Prompt

Copy the prompt below and paste it into your coding agent. It works in Claude Code, Codex, Gemini CLI, or any agent with shell access. The agent will clone the skill repository, copy the files into place, and clean up.

---

### Prompt: Install for all agents (Claude Code + Codex + Gemini CLI)

````
Install the github-ops AI agent skills into this project from https://github.com/skylordafk/agent-skills.git

These skills provide GitHub issue lifecycle management: audit, triage, fix-issues, and review-pr.

Follow these steps exactly:

1. Clone the repository to a temporary directory:
   ```
   git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-install
   ```

2. Create the required directories:
   ```
   mkdir -p .claude/skills .claude/references .agents/skills .agents/references .gemini/skills .gemini/references .github/workflows
   ```

3. Copy Claude Code skills (each is a directory containing SKILL.md):
   ```
   cp -r /tmp/agent-skills-install/github-ops/claude/audit .claude/skills/audit
   cp -r /tmp/agent-skills-install/github-ops/claude/triage .claude/skills/triage
   cp -r /tmp/agent-skills-install/github-ops/claude/fix-issues .claude/skills/fix-issues
   cp -r /tmp/agent-skills-install/github-ops/claude/review-pr .claude/skills/review-pr
   ```

4. Copy Codex skills (each is a directory containing SKILL.md + agents/openai.yaml):
   ```
   cp -r /tmp/agent-skills-install/github-ops/codex/audit .agents/skills/audit
   cp -r /tmp/agent-skills-install/github-ops/codex/triage .agents/skills/triage
   cp -r /tmp/agent-skills-install/github-ops/codex/fix-issues .agents/skills/fix-issues
   cp -r /tmp/agent-skills-install/github-ops/codex/review-pr .agents/skills/review-pr
   ```

5. Copy Gemini CLI skills (each is a directory containing SKILL.md):
   ```
   cp -r /tmp/agent-skills-install/github-ops/gemini/audit .gemini/skills/audit
   cp -r /tmp/agent-skills-install/github-ops/gemini/triage .gemini/skills/triage
   cp -r /tmp/agent-skills-install/github-ops/gemini/fix-issues .gemini/skills/fix-issues
   cp -r /tmp/agent-skills-install/github-ops/gemini/review-pr .gemini/skills/review-pr
   ```

6. Copy shared references to ALL agent directories (skills reference these via relative paths):
   ```
   cp /tmp/agent-skills-install/github-ops/references/*.md .claude/references/
   cp /tmp/agent-skills-install/github-ops/references/*.md .agents/references/
   cp /tmp/agent-skills-install/github-ops/references/*.md .gemini/references/
   ```

7. Copy the GitHub Actions review workflow:
   ```
   cp /tmp/agent-skills-install/.github/workflows/agent-pr-review.yml .github/workflows/agent-pr-review.yml
   ```

8. Configure the review identity if it is not already present:
   ```
   REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
   if gh variable list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_ID' && gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_PRIVATE_KEY'; then
     echo "GitHub App review identity configured."
   elif gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_TOKEN'; then
     echo "Bot PAT review identity configured."
   else
     echo "Configure a review identity. Preferred: AGENT_REVIEW_APP_ID variable + AGENT_REVIEW_APP_PRIVATE_KEY secret. See docs/github-app-review-identity.md."
   fi
   ```

   The preferred setup is a GitHub App installed on the target repository with Pull requests read/write access.

9. Clean up:
   ```
   rm -rf /tmp/agent-skills-install
   ```

10. Verify the installation by listing the installed files:
   ```
   echo "=== Claude Code ===" && ls -R .claude/skills/ .claude/references/
   echo "=== Codex ===" && ls -R .agents/skills/ .agents/references/
   echo "=== Gemini CLI ===" && ls -R .gemini/skills/ .gemini/references/
   echo "=== GitHub Actions ===" && ls -R .github/workflows/agent-pr-review.yml
   ```

The final directory structure should look like:

```
.claude/
├── skills/
│   ├── audit/SKILL.md
│   ├── triage/SKILL.md
│   ├── fix-issues/SKILL.md
│   └── review-pr/SKILL.md
└── references/
    ├── commit-and-pr-format.md
    ├── four-dimensions.md
    ├── label-taxonomy.md
    ├── operating-principles.md
    ├── risk-potential.md
    └── severity-definitions.md

.agents/
├── skills/
│   ├── audit/
│   │   ├── SKILL.md
│   │   └── agents/openai.yaml
│   ├── triage/
│   │   ├── SKILL.md
│   │   └── agents/openai.yaml
│   ├── fix-issues/
│   │   ├── SKILL.md
│   │   └── agents/openai.yaml
│   └── review-pr/
│       ├── SKILL.md
│       └── agents/openai.yaml
└── references/
    ├── commit-and-pr-format.md
    ├── four-dimensions.md
    ├── label-taxonomy.md
    ├── operating-principles.md
    ├── risk-potential.md
    └── severity-definitions.md

.gemini/
├── skills/
│   ├── audit/SKILL.md
│   ├── triage/SKILL.md
│   ├── fix-issues/SKILL.md
│   └── review-pr/SKILL.md
└── references/
    ├── commit-and-pr-format.md
    ├── four-dimensions.md
    ├── label-taxonomy.md
    ├── operating-principles.md
    ├── risk-potential.md
    └── severity-definitions.md

.github/
└── workflows/
    └── agent-pr-review.yml
```

Do not modify the skill files. The relative reference paths (../../references/) are already correct for this directory structure.
````

---

### Prompt: Install for Claude Code only

````
Install the github-ops AI agent skills for Claude Code into this project from https://github.com/skylordafk/agent-skills.git

Run these commands:
```
git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-install
mkdir -p .claude/skills .claude/references .github/workflows
cp -r /tmp/agent-skills-install/github-ops/claude/audit .claude/skills/audit
cp -r /tmp/agent-skills-install/github-ops/claude/triage .claude/skills/triage
cp -r /tmp/agent-skills-install/github-ops/claude/fix-issues .claude/skills/fix-issues
cp -r /tmp/agent-skills-install/github-ops/claude/review-pr .claude/skills/review-pr
cp /tmp/agent-skills-install/github-ops/references/*.md .claude/references/
cp /tmp/agent-skills-install/.github/workflows/agent-pr-review.yml .github/workflows/agent-pr-review.yml
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
if gh variable list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_ID' && gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_PRIVATE_KEY'; then
  echo "GitHub App review identity configured."
elif gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_TOKEN'; then
  echo "Bot PAT review identity configured."
else
  echo "Configure a review identity. Preferred: AGENT_REVIEW_APP_ID variable + AGENT_REVIEW_APP_PRIVATE_KEY secret. See docs/github-app-review-identity.md."
fi
rm -rf /tmp/agent-skills-install
```

Verify with: `ls -R .claude/skills/ .claude/references/ .github/workflows/agent-pr-review.yml`

Do not modify the skill files.
````

---

### Prompt: Install for Codex only

````
Install the github-ops AI agent skills for Codex into this project from https://github.com/skylordafk/agent-skills.git

Run these commands:
```
git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-install
mkdir -p .agents/skills .agents/references .github/workflows
cp -r /tmp/agent-skills-install/github-ops/codex/audit .agents/skills/audit
cp -r /tmp/agent-skills-install/github-ops/codex/triage .agents/skills/triage
cp -r /tmp/agent-skills-install/github-ops/codex/fix-issues .agents/skills/fix-issues
cp -r /tmp/agent-skills-install/github-ops/codex/review-pr .agents/skills/review-pr
cp /tmp/agent-skills-install/github-ops/references/*.md .agents/references/
cp /tmp/agent-skills-install/.github/workflows/agent-pr-review.yml .github/workflows/agent-pr-review.yml
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
if gh variable list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_ID' && gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_PRIVATE_KEY'; then
  echo "GitHub App review identity configured."
elif gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_TOKEN'; then
  echo "Bot PAT review identity configured."
else
  echo "Configure a review identity. Preferred: AGENT_REVIEW_APP_ID variable + AGENT_REVIEW_APP_PRIVATE_KEY secret. See docs/github-app-review-identity.md."
fi
rm -rf /tmp/agent-skills-install
```

Verify with: `ls -R .agents/skills/ .agents/references/ .github/workflows/agent-pr-review.yml`

Do not modify the skill files.
````

---

### Prompt: Install for Gemini CLI only

````
Install the github-ops AI agent skills for Gemini CLI into this project from https://github.com/skylordafk/agent-skills.git

Run these commands:
```
git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-install
mkdir -p .gemini/skills .gemini/references .github/workflows
cp -r /tmp/agent-skills-install/github-ops/gemini/audit .gemini/skills/audit
cp -r /tmp/agent-skills-install/github-ops/gemini/triage .gemini/skills/triage
cp -r /tmp/agent-skills-install/github-ops/gemini/fix-issues .gemini/skills/fix-issues
cp -r /tmp/agent-skills-install/github-ops/gemini/review-pr .gemini/skills/review-pr
cp /tmp/agent-skills-install/github-ops/references/*.md .gemini/references/
cp /tmp/agent-skills-install/.github/workflows/agent-pr-review.yml .github/workflows/agent-pr-review.yml
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
if gh variable list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_ID' && gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_PRIVATE_KEY'; then
  echo "GitHub App review identity configured."
elif gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_TOKEN'; then
  echo "Bot PAT review identity configured."
else
  echo "Configure a review identity. Preferred: AGENT_REVIEW_APP_ID variable + AGENT_REVIEW_APP_PRIVATE_KEY secret. See docs/github-app-review-identity.md."
fi
rm -rf /tmp/agent-skills-install
```

Verify with: `ls -R .gemini/skills/ .gemini/references/ .github/workflows/agent-pr-review.yml`

Do not modify the skill files.
````

## Manual Installation

If you prefer to install manually without an agent:

```bash
# Clone the skill repository
git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-install

# Choose your agent(s) and run the appropriate commands above

# Clean up
rm -rf /tmp/agent-skills-install
```

## How the Reference Paths Work

Skills contain instructions like "Read `../../references/label-taxonomy.md`". The agent resolves this relative to the skill file's location:

```
.claude/skills/triage/SKILL.md
       ↑      ↑
       │      └─ first ../
       └─ second ../

Result: .claude/references/label-taxonomy.md
```

This is why references must be at `.claude/references/` (or `.agents/references/`), not inside the skills directory. The directory structure in this repo is designed so the relative paths work correctly after copying.

## Updating Skills

### Prompt: Check for updates and upgrade (all agents)

Paste this into your coding agent to see what's changed and apply updates:

````
Check for updates to the github-ops AI agent skills from https://github.com/skylordafk/agent-skills.git and apply them.

Follow these steps exactly:

1. Clone the latest version:
   ```
   git clone --depth 1 https://github.com/skylordafk/agent-skills.git /tmp/agent-skills-update
   ```

2. Show what's changed by diffing against installed files. For each skill and reference file, compare the installed version to the latest version and summarize the differences:
   ```
   echo "=== Checking for changes ==="
   for dir in audit triage fix-issues review-pr; do
     for agent_dir in ".claude/skills:github-ops/claude" ".agents/skills:github-ops/codex" ".gemini/skills:github-ops/gemini"; do
       local_dir="${agent_dir%%:*}"
       source_dir="${agent_dir##*:}"
       if [ -f "$local_dir/$dir/SKILL.md" ]; then
         if ! diff -q "$local_dir/$dir/SKILL.md" "/tmp/agent-skills-update/$source_dir/$dir/SKILL.md" > /dev/null 2>&1; then
           echo "CHANGED: $local_dir/$dir"
           diff --unified=3 "$local_dir/$dir/SKILL.md" "/tmp/agent-skills-update/$source_dir/$dir/SKILL.md" | head -40
           echo "..."
         fi
       fi
     done
   done
   echo "=== Checking references ==="
   for ref in /tmp/agent-skills-update/github-ops/references/*.md; do
     name=$(basename "$ref")
     for ref_dir in .claude/references .agents/references .gemini/references; do
       if [ -f "$ref_dir/$name" ]; then
         if ! diff -q "$ref_dir/$name" "$ref" > /dev/null 2>&1; then
           echo "CHANGED: $ref_dir/$name"
         fi
       else
         echo "NEW: $ref_dir/$name (not currently installed)"
       fi
     done
   done
   if [ -f .github/workflows/agent-pr-review.yml ]; then
     if ! diff -q .github/workflows/agent-pr-review.yml /tmp/agent-skills-update/.github/workflows/agent-pr-review.yml > /dev/null 2>&1; then
       echo "CHANGED: .github/workflows/agent-pr-review.yml"
       diff --unified=3 .github/workflows/agent-pr-review.yml /tmp/agent-skills-update/.github/workflows/agent-pr-review.yml | head -40
       echo "..."
     fi
   else
     echo "NEW: .github/workflows/agent-pr-review.yml (not currently installed)"
   fi
   ```

3. Show me a summary of what changed and what's new before proceeding. Ask for confirmation.

4. After confirmation, copy the updated files:
   ```
   cp -r /tmp/agent-skills-update/github-ops/claude/audit .claude/skills/audit
   cp -r /tmp/agent-skills-update/github-ops/claude/triage .claude/skills/triage
   cp -r /tmp/agent-skills-update/github-ops/claude/fix-issues .claude/skills/fix-issues
   cp -r /tmp/agent-skills-update/github-ops/claude/review-pr .claude/skills/review-pr
   cp /tmp/agent-skills-update/github-ops/references/*.md .claude/references/

   cp -r /tmp/agent-skills-update/github-ops/codex/audit .agents/skills/audit
   cp -r /tmp/agent-skills-update/github-ops/codex/triage .agents/skills/triage
   cp -r /tmp/agent-skills-update/github-ops/codex/fix-issues .agents/skills/fix-issues
   cp -r /tmp/agent-skills-update/github-ops/codex/review-pr .agents/skills/review-pr
   cp /tmp/agent-skills-update/github-ops/references/*.md .agents/references/

   cp -r /tmp/agent-skills-update/github-ops/gemini/audit .gemini/skills/audit
   cp -r /tmp/agent-skills-update/github-ops/gemini/triage .gemini/skills/triage
   cp -r /tmp/agent-skills-update/github-ops/gemini/fix-issues .gemini/skills/fix-issues
   cp -r /tmp/agent-skills-update/github-ops/gemini/review-pr .gemini/skills/review-pr
   cp /tmp/agent-skills-update/github-ops/references/*.md .gemini/references/
   mkdir -p .github/workflows
   cp /tmp/agent-skills-update/.github/workflows/agent-pr-review.yml .github/workflows/agent-pr-review.yml
   REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
   if gh variable list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_ID' && gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_APP_PRIVATE_KEY'; then
     echo "GitHub App review identity configured."
   elif gh secret list --repo "$REPO" | grep -q '^AGENT_REVIEW_TOKEN'; then
     echo "Bot PAT review identity configured."
   else
     echo "Configure a review identity. Preferred: AGENT_REVIEW_APP_ID variable + AGENT_REVIEW_APP_PRIVATE_KEY secret. See docs/github-app-review-identity.md."
   fi
   echo ".claude/" >> .geminiignore
   echo ".agents/" >> .geminiignore
   ```

5. Clean up:
   ```
   rm -rf /tmp/agent-skills-update
   ```

6. Verify:
   ```
   echo "=== Claude Code ===" && ls -R .claude/skills/ .claude/references/
   echo "=== Codex ===" && ls -R .agents/skills/ .agents/references/
   echo "=== Gemini CLI ===" && ls -R .gemini/skills/ .gemini/references/
   echo "=== GitHub Actions ===" && ls -R .github/workflows/agent-pr-review.yml
   echo "=== .geminiignore ===" && cat .geminiignore
   ```

Do not modify the skill files after copying.
````

### Manual update

If you prefer to update without the prompt:

```bash
rm -rf .claude/skills/{audit,triage,fix-issues,review-pr} .claude/references/
rm -rf .agents/skills/{audit,triage,fix-issues,review-pr} .agents/references/
rm -rf .gemini/skills/{audit,triage,fix-issues,review-pr} .gemini/references/
# Then re-run the installation prompt
```

## Using the Skills

Once installed, invoke skills by name in your agent:

**Claude Code:**
- `/audit` — run a full audit cycle
- `/triage` — classify open issues
- `/fix-issues` — fix ready-to-fix issues
- `/review-pr 123` — review a specific PR

**Codex:**
- Skills appear in the Codex skill picker
- Invoke by name when relevant to your task

**Gemini CLI:**
- Skills are activated automatically when Gemini identifies a matching task
- You can also ask Gemini directly: "use the audit skill" or "triage the open issues"
- Use `/skills list` to see installed skills

All skills auto-detect the current repository via `gh repo view`. No per-project configuration is needed beyond the file installation.
