# Gate 0 — AWS Account, Cost & Retained-Asset Preflight

> **Rule:** do not create Phase 05 infrastructure until this checklist is completed with live account evidence.

## 1. Account / plan

- [ ] Confirm current AWS account plan.
- [ ] Confirm Free Plan remaining days if applicable.
- [ ] Confirm current promotional credit balance.
- [ ] Confirm credit expiry date.
- [ ] Confirm no account-plan upgrade is required for ECS/Fargate, ECR, ELB/ALB, RDS PostgreSQL, Secrets Manager and CloudWatch.
- [ ] If any service is unavailable, stop and redesign before creating dependent resources.

## 2. Current billing baseline

Capture month-to-date cost before Phase 05:

```bash
START=$(date +%Y-%m-01)
END=$(date -d tomorrow +%Y-%m-%d)

AWS_PAGER="" aws ce get-cost-and-usage \
  --time-period Start=$START,End=$END \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table
```

- [ ] Capture gross usage/fees.
- [ ] Capture credits separately.
- [ ] Save a screenshot as the Phase 05 starting cost baseline.

## 3. Verify Phase 04 remains cleaned up

Expected: no active Phase 04 cloud infrastructure.

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

- [ ] No Phase 04 WorkSpace.
- [ ] No Phase 04 AD Connector.
- [ ] No forgotten Phase 04 Elastic IP/VPC resources.

## 4. Verify Phase 03 retained assets

### AMI

```bash
AWS_PAGER="" aws ec2 describe-images \
  --region us-east-1 \
  --image-ids ami-0cbd2e9ec0d6f9168 \
  --query 'Images[0].{ImageId:ImageId,Name:Name,State:State,Architecture:Architecture,RootDeviceName:RootDeviceName}' \
  --output table
```

Expected: `State = available`.

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
AWS_PAGER="" aws s3api head-bucket \
  --bucket madar-operational-files-197821101770

AWS_PAGER="" aws s3 ls \
  s3://madar-operational-files-197821101770/operational-data/ \
  --recursive
```

- [ ] AMI exists and is available.
- [ ] backing snapshot exists.
- [ ] S3 bucket exists.
- [ ] expected operational data is visible.

## 5. Service/API access smoke checks

Read-only calls only:

```bash
AWS_PAGER="" aws ecs list-clusters --region us-east-1 --output table
AWS_PAGER="" aws ecr describe-repositories --region us-east-1 --output table
AWS_PAGER="" aws elbv2 describe-load-balancers --region us-east-1 --output table
AWS_PAGER="" aws rds describe-db-instances --region us-east-1 --output table
AWS_PAGER="" aws secretsmanager list-secrets --region us-east-1 --max-results 5 --output table
AWS_PAGER="" aws logs describe-log-groups --region us-east-1 --limit 5 --output table
```

- [ ] Commands succeed without account-plan/service-access errors.

## 6. Regional capability checks

- [ ] Confirm `db.t4g.micro` is orderable for PostgreSQL in `us-east-1` at execution time.
- [ ] Confirm intended PostgreSQL engine version.
- [ ] Confirm two AZs available for ALB public subnets and DB subnet group.
- [ ] Confirm Fargate platform supports initial `0.25 vCPU / 512 MiB` combination.

## 7. Budget guardrail

- [ ] Create/verify AWS Budget before expensive resources.
- [ ] Recommended warning threshold: `$3`.
- [ ] Recommended second threshold: `$5`.
- [ ] Confirm contact/notification destination works.

## 8. Planned cost clocks acknowledged

- [ ] RDS — billed while running.
- [ ] ALB — hourly + LCU while provisioned.
- [ ] Fargate — billed while tasks run.
- [ ] Public IPv4 — billed while allocated/in use.
- [ ] Secrets Manager — storage/API cost.
- [ ] ECR — image storage.
- [ ] CloudWatch — log ingestion/storage.
- [ ] No NAT Gateway planned.

## 9. GO / NO-GO record

```text
Account plan        :
Credits remaining   :
Credits expiry      :
AMI available       :
Snapshot available  :
S3 available        :
ECS/Fargate access  :
ALB access          :
RDS access          :
Secrets access      :
Budget configured   :

FINAL GATE 0: GO / NO-GO
```

Do not continue until every blocking item is resolved.
