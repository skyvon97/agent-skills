# GitHub App Review Identity

Use a GitHub App as the preferred review identity for `review-pr`. The app appears in PR reviews as `<app-slug>[bot]`, is separate from the human PR author, and can be limited to one repository or a selected set of repositories.

These instructions are safe to keep public. Do not publish the app private key, `AGENT_REVIEW_APP_PRIVATE_KEY`, or any fallback `AGENT_REVIEW_TOKEN` value.

## Create the App

Open this prefilled app registration URL while signed into GitHub. This URL is set up for `skyvon97/agent-skills`; for another repository, use the same permissions and set the homepage URL to that target repository.

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

## Using This in Another Repository

Each target repository needs two things:

1. The `Agent PR Review` workflow at `.github/workflows/agent-pr-review.yml`.
2. A separate review identity configured through repository Actions secrets and variables.

For personal use across multiple repositories, you can either reuse one private GitHub App or create one App per repository or project group.

Reuse one App when:

- You own or administer all target repositories.
- You are comfortable with one bot identity, such as `agent-skills-review-pr[bot]`, appearing across those repositories.
- You install the App only on the repositories where agent reviews should be allowed.

Create a separate App when:

- A repository belongs to a different user or organization.
- You want a different visible bot name or avatar.
- You want tighter blast-radius isolation between projects.
- You are distributing this toolset to someone else. They should create and install their own App instead of using yours.

For another repository, replace `OWNER/REPO` in the commands below:

```bash
REPO="OWNER/REPO"
gh variable set AGENT_REVIEW_APP_ID --repo "$REPO" --body "<APP_ID>"
gh secret set AGENT_REVIEW_APP_PRIVATE_KEY --repo "$REPO" < path/to/downloaded-private-key.pem
```

Then install the GitHub App on `OWNER/REPO` from the App settings page. The workflow uses the App ID and private key to mint a short-lived installation token for the repository where it is running. If the App is not installed on that repository, approval reviews will fail.

The same App ID and private key can be stored in more than one repository if you want the same App identity across projects. Treat that private key like any other production credential and rotate it from the App settings page if it is exposed.

After setup, verify with a trivial pull request in the target repository:

```text
/review-pr <PR_NUMBER>
```

A successful approval should create a PR review from `<app-slug>[bot]`, not from the human PR author and not from `github-actions[bot]`.
