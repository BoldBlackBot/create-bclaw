# Concepts: security model

How the two-role IAM model keeps a leaked deployer key from being
root-equivalent.

## The two roles

dispatch uses a **two-role model** so the deployer's long-lived access key is
never root-equivalent if it leaks:

- **`dispatch-deployer`** — the human identity. Its attached policy
  (`dispatch-deploy-policy.json`) carries narrow powers: manage the
  CloudFormation stack, write/read SSM secrets, shell in, scale the service,
  debug the container instance, recover orphans during teardown, and manage
  **one** literal role (`dispatch-cfn-exec`).
- **`dispatch-cfn-exec`** — the CloudFormation service role. Its inline policy
  (`dispatch-cfn-exec-policy.json`, trust `dispatch-cfn-exec-trust.json`)
  carries the broad infrastructure-create lifecycle (EC2/ASG/ECS/IAM/KMS/logs)
  that CloudFormation assumes during every deploy and the stack delete.

The deployer identity only ever passes `dispatch-cfn-exec` to CloudFormation
(`iam:PassRole` conditioned to `cloudformation.amazonaws.com`); it never
touches infrastructure resources directly.

Because `dispatch-cfn-exec` is assumable **only** by
`cloudformation.amazonaws.com` (its trust policy) and the deployer's only
`iam:PassRole` for it is conditioned to that same service, none of the broad
powers are reachable by the human-held key — closing the privilege-escalation
chains a leaked deployer key otherwise opens.

## Why the service role exists at all

The stack cannot create the role it assumes to create itself. It is therefore
created out-of-band in setup Phase 0 and deleted last in teardown (after the
stack is gone).

## Scoping rules in the deployer policy

Resources whose ARNs use AWS-assigned IDs cannot be pinned by ARN prefix. The
policy pins what it can and uses tag conditions (ABAC) or read-only
`Resource: "*"` where AWS forces it:

| Statement type | Examples |
|---|---|
| ARN-pinned | `CloudFormation` (`stack/dispatch/*`), `ManageCfnExecRole` (`role/dispatch-cfn-exec`), `PassRoleToCfn`, `ECSExec` (`cluster/dispatch`, `task/dispatch/*`), `ECSServiceManage`, `LogsRead`, `SSMSecrets` (`parameter/dispatch/*`) |
| Tag-conditioned (ABAC) | `EC2NetworkingManage` (`Name = dispatch*`), `EC2InstanceOps` (`ClawName = dispatch`, ARN-scoped to `instance/*`), `EC2DataVolumeManage` (`Name = dispatch-data`), `KMSUseKey` (`ResourceAliases = alias/dispatch-ssm`) |
| Read-only `*` (AWS-forced) | `ReadOnlyDescribe`, `ECSRead`, `SSMMessages`, `CloudFormationGlobalMeta` |

Two deliberate omissions:

- **No `iam:SimulatePrincipalPolicy`** — a leaked key should not be able to
  probe its own scope. Permission gaps surface as the exact
  `is not authorized to perform` error at deploy time.
- **No `sts:DecodeAuthorizationMessage`** — the deployer key deliberately
  cannot decode encoded denial messages; use a separate admin identity.

## ECS Exec and session logging

Shell-in (`ecs:ExecuteCommand`) permissions are on the deployer, scoped to the
agent's cluster and tasks. AWS additionally recommends **denying**
`ssm:StartSession` on ECS tasks (`DenyDirectSSMSession`): sessions via
`ecs:ExecuteCommand` are logged; direct SSM sessions bypass ECS Exec logging
and consume the session quota.

## The secrets boundary

Secrets are SSM SecureStrings under `/dispatch/`, encrypted with the stack's
own KMS key. The deployer policy pins `kms:Decrypt`/`kms:Encrypt` to
`alias/dispatch-ssm` via `kms:ResourceAliases`, so the deployer can only
decrypt parameters this key encrypted. See
[Concepts: secrets](secrets.md).
