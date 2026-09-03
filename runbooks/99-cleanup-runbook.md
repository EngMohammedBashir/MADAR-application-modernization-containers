# 🧹 Phase 05 — Cleanup Runbook

> ⚠️ All required build/failure/scaling/observability/cost evidence has been captured.  
> 🎯 **This is now the active runbook.** Goal: remove Phase 05 cost-bearing infrastructure while preserving Git evidence and intentionally retained Phase 03 artifacts.

## 🛡️ Never Delete as Part of Phase 05

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

The S3 bucket now also contains the recovered database dump. Retention/deletion of these Phase 03 assets requires a separate decision.

## 📍 Known Phase 05 Cleanup Inventory

```text
VPC             vpc-011a441b0a790c458
IGW             igw-0df8ed399478fa879
Public RT       rtb-076fdacedac35cd66
Private RT      rtb-0ed3daeca13e9987e
ALB-SG          sg-00b9b70e13293ff46
ECS-SG          sg-0d13f6af551e284c8
RDS-SG          sg-00ae439cb916d164b
RDS             madar-p05-postgres
DB subnet group madar-p05-db-subnet-group
Secret          MADAR/Phase05/Postgres
Log group       /ecs/madar-phase05-app
ECR app         madar-phase05-app
ECR restore     madar-p05-restore
ECS cluster     MADAR-P05-Cluster
ECS service     MADAR-P05-App-Service
ALB             MADAR-P05-ALB
Target group    MADAR-P05-TG
Task defs       madar-phase05-app:1
                madar-p05-db-restore:1
                madar-p05-db-verify:1
IAM execution   MADAR-P05-ECS-ExecutionRole
IAM restore     MADAR-P05-Restore-TaskRole
Scaling policy  MADAR-P05-CPU-TargetTracking
Scalable target service/MADAR-P05-Cluster/MADAR-P05-App-Service
```

## 1️⃣ Remove Application Auto Scaling

Delete the target-tracking policy, then deregister the scalable target. Verify no scaling policy remains for the ECS service.

## 2️⃣ Scale ECS Service to Zero

Set `MADAR-P05-App-Service` desired count to `0`. Wait for `runningCount=0` and `pendingCount=0`.

## 3️⃣ Delete ECS Service

Delete the service after tasks drain. Verify no running service tasks remain.

## 4️⃣ Delete ALB Listener / Rules

Discover the listener ARN rather than guessing. Delete the HTTP listener/rules associated with `MADAR-P05-ALB`.

## 5️⃣ Delete ALB

Delete `MADAR-P05-ALB` and wait until `describe-load-balancers` no longer returns it. Allow ENI cleanup to finish before network deletion.

## 6️⃣ Delete Target Group

Delete `MADAR-P05-TG` only after listener/load-balancer references are gone.

## 7️⃣ Deregister Task Definitions

Deregister:

```text
madar-phase05-app:1
madar-p05-db-restore:1
madar-p05-db-verify:1
```

AWS may retain inactive task-definition metadata; document that accurately.

## 8️⃣ Delete ECS Cluster

Delete `MADAR-P05-Cluster` only after service/tasks are gone.

## 9️⃣ Delete RDS

Delete `madar-p05-postgres` with **skip final snapshot** as planned for this disposable Phase 05 database. Wait until the instance disappears before deleting its subnet group.

## 🔟 Delete DB Subnet Group

Delete `madar-p05-db-subnet-group` after RDS is fully gone.

## 1️⃣1️⃣ Delete Phase 05 Secret

Delete `MADAR/Phase05/Postgres` using an appropriate disposable-lab deletion method. Never retrieve or print the secret value during verification.

## 1️⃣2️⃣ Delete CloudWatch Log Group

Delete `/ecs/madar-phase05-app` only after all evidence/log requirements are satisfied.

## 1️⃣3️⃣ Delete Both Phase 05 ECR Repositories

Delete images/repositories:

```text
madar-phase05-app
madar-p05-restore
```

The restore image is temporary and should not be forgotten.

## 1️⃣4️⃣ Remove IAM Roles / Policies

For `MADAR-P05-ECS-ExecutionRole`:
- detach `AmazonECSTaskExecutionRolePolicy`,
- delete the Phase 05 secret-access inline policy,
- delete the role.

For `MADAR-P05-Restore-TaskRole`:
- delete the exact-dump S3 read inline policy,
- delete the role.

Delete any other Phase 05 task role only if actually present and verified by name/tag.

## 1️⃣5️⃣ Delete Security Groups

After ALB/ECS/RDS ENIs are gone, delete:

```text
sg-00b9b70e13293ff46  ALB-SG
sg-0d13f6af551e284c8  ECS-SG
sg-00ae439cb916d164b  RDS-SG
```

Do not manually delete the VPC default security group.

## 1️⃣6️⃣ Delete Subnets / Custom Route Tables

Delete all four Phase 05 subnets after dependent ENIs disappear, then delete the custom public/private route tables. The VPC main route table is removed with the VPC.

## 1️⃣7️⃣ Detach / Delete IGW

Detach `igw-0df8ed399478fa879` from `vpc-011a441b0a790c458`, then delete it.

## 1️⃣8️⃣ Delete Phase 05 VPC

Delete `vpc-011a441b0a790c458` only after dependencies are gone.

## 🔍 19 — Residual Resource Audit

Check at minimum:

```text
🚀 ECS clusters/services/running tasks
📈 Application Auto Scaling policies/targets
⚖️ ALB/listeners/target groups
🐘 RDS instances/subnet groups
🌐 public IPv4 / Elastic IPs / ENIs
🚫 NAT gateways — should be none for Phase 05
🏠 Phase 05 VPC/subnets/routes/IGW
📦 both ECR repositories
🔐 Phase 05 secret
📊 Phase 05 log group
🔑 both Phase 05 IAM roles/policies
```

Capture `phase05-residual-audit.png` only after the audit is clean.

## 💰 20 — Final Cost Closeout

Cost Explorer can lag. Capture the final available month-to-date checkpoint and describe it as a billing checkpoint, not an absolute real-time zero-cost guarantee.

## 📸 Cleanup Evidence

Recommended:

```text
phase05-final-cleanup.png
phase05-residual-audit.png
```

## 🏁 Definition of Done

```text
No running Phase 05 Fargate tasks       ✅ required
No Phase 05 scaling target/policy       ✅ required
No Phase 05 ALB/TG                      ✅ required
No Phase 05 RDS/subnet group            ✅ required
No Phase 05 public IPv4/ENIs surprise   ✅ required
No Phase 05 ECR repos                   ✅ required
No Phase 05 secret/log group            ✅ required
No Phase 05 IAM roles                   ✅ required
No Phase 05 VPC                         ✅ required
Residual audit clean                    ✅ required
Final cost checkpoint recorded          ✅ required
Phase 03 retained assets untouched      ✅ required
```
