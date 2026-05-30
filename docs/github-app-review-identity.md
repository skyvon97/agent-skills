# GitHub App Review Identity

Use a GitHub App as the preferred review identity for `review-pr`. The app appears in PR reviews as `<app-slug>[bot]`, is separate from the human PR author, and can be limited to this repository.

## Create the App

Open this prefilled app registration URL while signed into GitHub:

```text
https://github.com/settings/apps/new?name=agent-skills-review-pr&description=Posts%20agent-authored%20pull%20request%20reviews%20from%20the%20review-pr%20skill&url=https%3A%2F%2Fgithub.com%2Fskyvon97%2Fagent-skills&public=false&webhook_active=false&pull_requests=write
```

Settings to verify before creating:

- **Name:** `agent-skills-review-pr` or another unique name
- **Homepage URL:** `https://github.com/skyvon97/agent-skills`
- **Webhook:** inactive
- **Repository permissions:** Pull requests: Read and write
- **Installable by:** Only this account

After creating the app:

1. Install it on `skyvon97/agent-skills`.
2. Generate a private key and download the `.pem` file.
3. Copy the app ID from the app settings page.

## Configure Repository Secrets

Store the app ID as a repository variable:

```bash
gh variable set AGENT_REVIEW_APP_ID --repo skyvon97/agent-skills --body "<APP_ID>"
```

Store the private key as a repository secret:

```bash
gh secret set AGENT_REVIEW_APP_PRIVATE_KEY --repo skyvon97/agent-skills < path/to/downloaded-private-key.pem
```

## Verify

After the workflow with GitHub App support is on `main`, run `review-pr` against a trivial PR. A successful approval should create a PR review from `<app-slug>[bot]`, not from `skyvon97` and not from `github-actions[bot]`.

`AGENT_REVIEW_TOKEN` remains supported as a fallback for a separate bot account PAT, but the GitHub App path is preferred because it uses short-lived installation tokens.
