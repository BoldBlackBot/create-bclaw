# Concepts: skills

The three skills that run a dispatch agent's lifecycle, and what each one
does.

## The lifecycle

```text
setup-dispatch  →  manage-dispatch  →  teardown-dispatch
   (deploy)         (day-2 ops)          (decommission)
```

All three ship inside the generated repo at `.agents/skills/`. Your harness
discovers them automatically; each is invoked by name — `/setup-dispatch`,
`/manage-dispatch`, `/teardown-dispatch` — or simply by asking the agent in
natural language ("roll the image to hermes-1.9.11").

## setup-dispatch

Bootstraps the agent from nothing to a running gateway. Follows a gated
sequence — each phase must succeed before the next begins:

| Phase | What happens |
|---|---|
| 0 | Create the CloudFormation service role (`dispatch-cfn-exec`) |
| 1 | Collect configuration, probe one ARM64 availability zone |
| 2 | Deploy the CloudFormation stack (first deploy at `DesiredCount 0`) |
| 3 | Write the SSM secrets |
| 4 | Scale the service to 1 and verify boot |
| 5 | Overlay `agent_home/`, install the `aws_ssm` plugin, merge secrets config |
| 6 | Shell in (ECS Exec) |
| 7 | Final report |

Permissions are not pre-checked: if the deployer principal is missing an
action, CloudFormation surfaces the exact `is not authorized to perform` error
at deploy time — fix the policy and re-run.

## manage-dispatch

Five modes covering everything after the first deploy:

1. **Overlay** (default) — push the repo's `agent_home/` onto the agent's
   `~/.hermes` to update skills, memories, system prompt, or personas without
   a redeploy. Transferred over ECS Exec; `config.yaml` is excluded (use mode 3).
2. **Run** — execute arbitrary commands on the live agent for inspection,
   debugging, or one-off operations, over ECS Exec.
3. **Merge-config** — key-level merge of `agent_home/config.yaml` into the
   live `config.yaml`, with conflict resolution.
4. **Upgrade image** — roll the running agent onto a new
   `ghcr.io/boldblackai/harness` tag by bumping the `HarnessImageTag` stack
   parameter and redeploying. No image rebuild.
5. **Host** — retrieve a stuck container instance's console output or force a
   wedged instance to replace itself, scoped to the agent's instances via
   `aws:ResourceTag/ClawName`.

## teardown-dispatch

Removes everything, in the reverse order of setup so dependencies delete
cleanly without orphans:

| Phase | What happens |
|---|---|
| 0 | Pre-flight — confirm AWS access |
| 1 | Scale the service to 0 |
| 2 | Delete the CloudFormation stack (EBS volume is retained) |
| 3 | Delete the retained EBS volume |
| 4 | Delete the orphaned VPC and networking |
| 5 | Delete the SSM secrets |
| 6 | Delete the CloudFormation service role |
| 7 | Final verification — no stacks, volumes, params, or service role remain |

## Where curated state lives

`agent_home/` in the generated repo is the source of truth for the agent's
skills, memories, system prompt, and personas. Changes to it reach the live
agent through the overlay (manage mode 1) — commit to the repo, push the
overlay, restart. `config.yaml` changes go through merge-config (mode 3).

The agent's own runtime state (sessions, SQLite databases) lives on the EBS
volume and is never touched by the overlay.
