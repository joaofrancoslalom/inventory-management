# Setting up Claude GitHub Actions with Amazon Bedrock

## The problem

Claude Code's built-in `/install-github-app` command only supports direct
Anthropic API key accounts. It assumes you'll authorize the Anthropic-hosted
[Claude GitHub App](https://github.com/apps/claude) and provisions an
`ANTHROPIC_API_KEY` repo secret.

When Claude Code is configured for Amazon Bedrock
(`CLAUDE_CODE_USE_BEDROCK=1`), that command isn't available at all, and
visiting the App's install page redirects to an Anthropic Console/Claude.ai
login screen instead of a normal GitHub App installation — because that flow
is tied to Anthropic's own API-key provisioning, which doesn't apply to
Bedrock users.

## The workaround

Skip the Anthropic-hosted GitHub App entirely. It isn't required — the
`claude-code-action` workflow only needs standard GitHub Actions triggers and
a way to authenticate to AWS Bedrock.

1. **GitHub auth**: use the default `GITHUB_TOKEN` that GitHub Actions
   already injects per run (no App, no PAT). This is enough for the action to
   read the triggering event and post comments back.

2. **AWS auth**: create an IAM role trusted via GitHub's OIDC provider,
   scoped to this repo only, with Bedrock-invoke-only permissions —
   no GitHub App, no long-lived AWS keys stored as secrets.

   - Reuse the account's existing `token.actions.githubusercontent.com` OIDC
     provider if one is already registered (check with
     `aws iam list-open-id-connect-providers`).
   - Trust policy restricted to
     `token.actions.githubusercontent.com:sub` = `repo:<owner>/<repo>:*`.
   - Permissions policy limited to `bedrock:InvokeModel` and
     `bedrock:InvokeModelWithResponseStream` in the Bedrock region you use
     (e.g. `eu-west-1`) — not `AdministratorAccess`.

   ```bash
   aws iam create-role \
     --role-name github-actions-<repo-name> \
     --assume-role-policy-document file://trust-policy.json

   aws iam put-role-policy \
     --role-name github-actions-<repo-name> \
     --policy-name BedrockInvokeOnly \
     --policy-document file://bedrock-permissions.json
   ```

3. **Workflow file** (`.github/workflows/claude.yml`): trigger on `@claude`
   mentions, assume the role via OIDC, then run the action with
   `use_bedrock: "true"`:

   ```yaml
   permissions:
     contents: read
     pull-requests: write
     issues: write
     id-token: write   # required for OIDC

   jobs:
     claude:
       if: contains(github.event.comment.body, '@claude')  # or issue/review body
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4

         - uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::<account-id>:role/github-actions-<repo-name>
             aws-region: eu-west-1

         - uses: anthropics/claude-code-action@v1
           with:
             github_token: ${{ secrets.GITHUB_TOKEN }}
             use_bedrock: "true"
             model: "eu.anthropic.claude-sonnet-5"
   ```

## Why this works

- GitHub App installation is a one-time, user-consent, browser-only action —
  there's no CLI/API way around it. Since Bedrock auth doesn't need the
  Anthropic App at all, that whole step is simply skipped.
- `id-token: write` in the workflow's `permissions` block lets the job
  request a short-lived OIDC token from GitHub, which
  `aws-actions/configure-aws-credentials` exchanges for temporary AWS
  credentials via `sts:AssumeRoleWithWebIdentity` — no static AWS keys ever
  touch GitHub Actions secrets.
- Scoping the trust policy's `sub` condition to this repo (and the
  permissions policy to Bedrock-invoke-only) means the role can't be used
  from another repo or for anything beyond calling Bedrock models, even if
  the OIDC provider is shared across multiple projects in the same AWS
  account.

## Caveats

- Workflow files that react to `issue_comment` / `pull_request_review` events
  are only picked up from the repository's **default branch**. A workflow
  added on a feature branch won't fire until it's merged.
- If the AWS account already has an OIDC provider for
  `token.actions.githubusercontent.com` (common in shared/multi-repo AWS
  accounts), reuse it — don't create a duplicate. Check for pre-existing
  GitHub Actions IAM roles first; avoid reusing an existing role that has
  broader trust (e.g. `repo:*`) or broader permissions (e.g.
  `AdministratorAccess`) than this workflow needs.
