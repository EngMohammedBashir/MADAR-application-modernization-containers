# 🚦 START HERE — Phase 05 Day One

Do **not** create a VPC, RDS, ALB or ECS service first.

Tomorrow starts with one goal:

> Prove the account and retained Phase 03 artifacts are ready before spending anything on Phase 05.

## Step 1 — Open AWS CloudShell in `us-east-1`

Confirm identity/region:

```bash
aws sts get-caller-identity
aws configure get region
```

If no default region is set, commands in this runbook explicitly use `us-east-1`.

## Step 2 — Run the retained-asset audit

### AMI

```bash
AWS_PAGER="" aws ec2 describe-images \
  --region us-east-1 \
  --image-ids ami-0cbd2e9ec0d6f9168 \
  --query 'Images[0].{ImageId:ImageId,Name:Name,State:State,Architecture:Architecture,RootDeviceName:RootDeviceName}' \
  --output table
```

### Snapshot

```bash
AWS_PAGER="" aws ec2 describe-snapshots \
  --region us-east-1 \
  --snapshot-ids snap-0920a020c47fb6447 \
  --query 'Snapshots[0].{SnapshotId:SnapshotId,State:State,VolumeSize:VolumeSize,Encrypted:Encrypted}' \
  --output table
```

### S3

```bash
AWS_PAGER="" aws s3 ls \
  s3://madar-operational-files-197821101770/operational-data/ \
  --recursive
```

Expected checkpoint:

```text
AMI       available
Snapshot  completed
S3        reachable / expected objects visible
```

## Step 3 — Reconfirm Phase 04 cleanup

```bash
AWS_PAGER="" aws workspaces describe-workspaces \
  --region us-east-1 \
  --query 'Workspaces[*].[WorkspaceId,UserName,State]' \
  --output table

AWS_PAGER="" aws ds describe-directories \
  --region us-east-1 \
  --query 'DirectoryDescriptions[*].[DirectoryId,Name,Type,Stage]' \
  --output table
```

Expected: no Phase 04 WorkSpace or directory.

## Step 4 — Read-only service access smoke test

```bash
AWS_PAGER="" aws ecs list-clusters --region us-east-1 --output table
AWS_PAGER="" aws ecr describe-repositories --region us-east-1 --output table
AWS_PAGER="" aws elbv2 describe-load-balancers --region us-east-1 --output table
AWS_PAGER="" aws rds describe-db-instances --region us-east-1 --output table
AWS_PAGER="" aws secretsmanager list-secrets --region us-east-1 --max-results 5 --output table
```

We are looking for successful API access — not for existing Phase 05 resources.

## Step 5 — Billing / credit checkpoint

From Billing & Cost Management verify:

```text
Current account plan
Free Plan remaining time (if applicable)
Credit balance
Credit expiration
```

Then capture current Cost Explorer baseline.

Do not assume promotional credits automatically prove a service is available to the account.

## Step 6 — Budget

Before RDS/ALB/Fargate creation:

```text
Warning budget     $3
Secondary warning  $5
```

Confirm notification delivery.

## Step 7 — GO / NO-GO

Record:

```text
AMI                PASS / FAIL
Snapshot           PASS / FAIL
S3                 PASS / FAIL
Phase04 cleanup    PASS / FAIL
ECS API            PASS / FAIL
ECR API            PASS / FAIL
ELB API            PASS / FAIL
RDS API            PASS / FAIL
Secrets API        PASS / FAIL
Credits checked    PASS / FAIL
Budget ready       PASS / FAIL

GATE 0             GO / NO-GO
```

### If GO

The first cost-bearing engineering action is **not RDS**.

Next:

```text
Launch temporary EC2 from retained AMI
      ↓
Find existing Flask application
      ↓
Inspect runtime/config/dependencies
      ↓
Extract safe source
      ↓
Terminate temporary EC2
```

Then containerization happens locally before the ECS/RDS/ALB environment exists.

### If NO-GO

Stop. Document the exact account/service restriction before changing architecture.

Do not improvise an upgrade or substitute service in the middle of the build.
