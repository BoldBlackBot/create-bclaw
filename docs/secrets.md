# Concepts: secrets

Where a dispatch agent's secrets live, how they are encrypted, and how to
rotate them.

## SSM, not CloudFormation

Secrets are SSM Parameter Store **SecureStrings** under the `/dispatch/`
namespace (renamed to your agent's name at generation). They are deliberately
**not** CloudFormation resources — they live in SSM so they survive stack
updates and stack deletes, and stay out of template diffs.

Each parameter must be encrypted with the **agent's own KMS key** (alias
`alias/dispatch-ssm`, created by the setup stack) — **not** the default
`alias/aws/ssm`. The deployer IAM policy pins `kms:Decrypt`/`kms:Encrypt` to
`alias/dispatch-ssm` via `kms:ResourceAliases`, so the agent can only decrypt
parameters this key encrypted. A parameter left under the default SSM key
fails to decrypt — the plugin can't resolve it and the gateway runs without
it.

## Resolution at startup

A Hermes secret-source plugin (`aws_ssm`, from
[boldblackai/hermes-aws-ssm-secret-source](https://github.com/boldblackai/hermes-aws-ssm-secret-source),
installed during setup) resolves every `/dispatch/*` parameter into the
gateway's environment at startup. Adding or rotating a key is an SSM write plus
a task restart — no template edit, no redeploy.

## The inventory

### Slack (required)

| SSM key | What it is | Where to find it |
|---|---|---|
| `/dispatch/SLACK_BOT_TOKEN` | Slack bot OAuth token (`xoxb-`) | Slack app → OAuth & Permissions → Bot User OAuth Token |
| `/dispatch/SLACK_APP_TOKEN` | Slack app-level token (`xapp-`, enables socket mode) | Slack app → Basic Information → App-Level Tokens |
| `/dispatch/SLACK_ALLOWED_USERS` | Comma-separated Slack user IDs allowed to use the bot | Slack profile → "Copy member ID" |
| `/dispatch/SLACK_HOME_CHANNEL` | Slack channel ID the bot treats as home | Right-click channel → "Copy link", take the trailing ID |

### Inference-provider key (create at least one)

| SSM key | What it is | Where to find it |
|---|---|---|
| `/dispatch/OPENROUTER_API_KEY` | OpenRouter API key (recommended) | <https://openrouter.ai/keys> |
| `/dispatch/ANTHROPIC_API_KEY` | Anthropic (direct Claude API) | <https://console.anthropic.com/> |
| `/dispatch/ZAI_API_KEY` | Z.AI / Zhipu (GLM) | <https://z.ai/manage-apikey/apikey-list> |

The aws_ssm plugin resolves every provider key present in SSM, so you can
create more than one if the gateway uses multiple providers.

### Optional: GitHub authentication

| SSM key | What it is | Where to find it |
|---|---|---|
| `/dispatch/GH_TOKEN_VAL` | GitHub PAT for the on-boot `gh auth login` | <https://github.com/settings/tokens> |

`/dispatch/GH_TOKEN_VAL` is the ONE secret still injected via CloudFormation
(`secrets[]` + the `EnableGitHubKey` stack parameter), because the on-boot
`gh auth login --with-token` runs before Hermes (and the aws_ssm plugin) start.
Named `*_VAL`, not `GH_TOKEN`, to avoid `gh`'s reserved env var. Enable it only
if the agent should make authenticated `gh`/HTTPS-git calls; when disabled the
login is skipped and no token is injected.

## Rotation

Rotate any SSM-resolved secret by overwriting the parameter (with the same
KMS key) and restarting the task:

```bash
aws ssm put-parameter --name /swe-pal/OPENROUTER_API_KEY \
  --type SecureString --key-id alias/swe-pal-ssm \
  --value "sk-or-..." --overwrite
```

then restart the agent (manage-dispatch mode 2, or scale the service
0 → 1). The `GH_TOKEN_VAL` exception is rotated via its stack parameter
instead.
