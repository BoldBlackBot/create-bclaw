# CLI reference

`npx @boldblackai/create-dispatch` — generate a repository for your dispatch
agent.

## Usage

```bash
npx @boldblackai/create-dispatch <name>
# or equivalently
npm init @boldblackai/dispatch <name>
```

If no name is given (and stdin is a TTY), you'll be prompted for one.

## The name

`<name>` must match `^[a-zA-Z]([a-zA-Z0-9-]*[a-zA-Z0-9])?$` and be 1–59
characters. It becomes:

- the CloudFormation stack name,
- the IAM role prefix (`<name>-exec`, `<name>-task`, `<name>-instance`),
- the ECS cluster and service name,
- the log group,
- the SSM namespace (`/<name>/`),
- the KMS alias (`alias/<name>-ssm`),
- and the EBS volume tag (`<name>-data`).

The 59-char ceiling keeps the `-exec`/`-task`/`-instance` role suffixes under
IAM's 64-char role-name limit. A name containing the literal region token
`us-east-1` is rejected (it would be corrupted by region substitution).

## Options

- `--region <region>` — AWS region to bake into the agent (default
  `us-east-1`). Substituted into the deployer IAM policy's `kms:ViaService`
  so the agent works in that region.
- `--force` — generate into a non-empty target directory, merging with
  existing files (default: refuse).
- `--version`, `-V` — print the version.
- `--help`, `-h` — show help.

`--region` must match `^[a-z]{2}(-gov)?-[a-z]+-[0-9]+$` (any AWS region,
including GovCloud/China). If omitted and stdin is a TTY you'll be prompted;
otherwise the default is used silently.

## What generation does

Running the generator produces a `<name>/` directory whose contents match the
bundled `template/` snapshot except every lowercase `dispatch` reference —
file contents **and** file/directory names — is renamed to `<name>`. A second
literal token, `us-east-1`, is substituted with the chosen AWS region so
region-bearing static files match the deploy region.

## The pointer stub

The unscoped npm name `create-dispatch` is a reserved pointer stub: the CLI is
published as `@boldblackai/create-dispatch`. Reaching the unscoped name — by
habit or by guess — prints a pointer to the real package instead of a dead
end.
