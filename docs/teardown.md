# Tearing down

Remove a dispatch agent and every associated AWS resource.

## Run the teardown skill

Open the generated repo in your harness and run the `/teardown-dispatch`
skill. It follows the reverse order of setup so dependencies delete cleanly
without orphans:

| Phase | What happens |
|---|---|
| 0 | Pre-flight — confirm AWS access |
| 1 | Scale the service to 0 |
| 2 | Delete the CloudFormation stack (the EBS volume is **retained**, not deleted) |
| 3 | Delete the retained EBS volume |
| 4 | Delete the orphaned VPC and networking (subnets, route tables, IGW, SG) |
| 5 | Delete the SSM secrets (every parameter under `/<name>/`) |
| 6 | Delete the CloudFormation service role (`dispatch-cfn-exec`) |
| 7 | Final verification — no stacks, volumes, params, or service role remain |

## What is deleted last, and why

Two resources are deliberately deleted **after** the stack:

- **The EBS data volume** — `DeletionPolicy: Retain` keeps it alive through the
  stack delete so a teardown run aborting mid-way never destroys your data by
  accident. Deleting it explicitly (Phase 3) is the point of no return.
- **The CloudFormation service role** — the stack cannot create the role it
  assumes to create itself, so it is created out-of-band in setup Phase 0 and
  survives `delete-stack`. Phase 6 removes it once the stack is gone.

## Data warning

Phase 3 permanently destroys the agent's SQLite databases — sessions,
memories, and any runtime state. If you might want the data back, snapshot the
volume before tearing down:

```bash
aws ec2 create-snapshot \
  --description "swe-pal data volume backup" \
  --volume-id <volume-id>
```

## After teardown

The generated repository itself (and its git history) is untouched — delete
the local directory when you no longer want it. To bring the agent back, run
`/setup-dispatch` again on a freshly generated (or the same) repo: setup
re-creates everything from scratch, including the service role and SSM
secrets (you'll re-enter them).
