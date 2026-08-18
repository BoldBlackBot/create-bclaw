# dispatch

dispatch is an opinionated, one-command deployment of
[hermes-agent](https://hermes-agent.nousresearch.com/) — a long-running coding
agent that lives in your Slack workspace and runs on your own AWS account.

One `npx` command generates a repository for your own dispatch agent. The
generated repo is a set of skills you open in your favorite
[harness](https://github.com/boldblackai/harness); the skills deploy the agent
to AWS ECS and manage it for the rest of its life.

This documentation covers what a dispatch agent is, how to generate one, and
how to set up, manage, upgrade, and tear it down.

[Get started](getting-started.md)

## Where to go next

- [Getting started](getting-started.md) — generate a repo and deploy your first agent
- [Architecture](architecture.md) — what the CloudFormation stack builds on AWS
- [Concepts: skills](skills.md) — the three skills that run the lifecycle
- [Concepts: security model](security.md) — the deployer/service-role split
- [Concepts: secrets](secrets.md) — SSM SecureStrings under `/dispatch/`
- [CLI reference](cli.md) — `npx @boldblackai/create-dispatch` flags
- [Upgrading](upgrading.md) — rolling the running agent to a new image tag
- [Tearing down](teardown.md) — removing every AWS resource
