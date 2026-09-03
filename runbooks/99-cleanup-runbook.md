# 🧹 Phase 05 — Cleanup Runbook

> 🟢 **EXECUTED SUCCESSFULLY — 2026-09-03**  
> Purpose: let me—or another engineer—repeat the dependency-safe teardown without deleting MADAR continuity assets.

## 🛡️ NEVER Delete in Phase 05 Cleanup

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

## 🧠 Dependency-Safe Order

```text
Scaling
→ ECS service/tasks
→ ALB/TG
→ task definitions/cluster
→ RDS/subnet group
→ secret/logs/ECR
→ IAM
→ SGs
→ subnets/custom route tables
→ IGW
→ VPC
→ residual audit
→ cost closeout
```

## 1️⃣ Scaling — ✅ DELETED

```powershell
aws application-autoscaling delete-scaling-policy --service-namespace ecs --resource-id service/MADAR-P05-Cluster/MADAR-P05-App-Service --scalable-dimension ecs:service:DesiredCount --policy-name MADAR-P05-CPU-TargetTracking --region us-east-1
aws application-autoscaling deregister-scalable-target --service-namespace ecs --resource-id service/MADAR-P05-Cluster/MADAR-P05-App-Service --scalable-dimension ecs:service:DesiredCount --region us-east-1
```

Verify both policy/target lists are empty.

## 2️⃣ ECS Service — ✅ DELETED

```powershell
aws ecs update-service --cluster MADAR-P05-Cluster --service MADAR-P05-App-Service --desired-count 0 --region us-east-1
aws ecs delete-service --cluster MADAR-P05-Cluster --service MADAR-P05-App-Service --force --region us-east-1
```

I verified `Desired=0`, `Running=0`, `Pending=0`, then no running tasks/services remained.

## 3️⃣ ALB + Target Group — ✅ DELETED

Discover ARNs rather than guessing when rebuilding the procedure. Delete ALB, wait for disappearance/ENI release, then delete TG. Final verification returned `LoadBalancerNotFound` and `TargetGroupNotFound`.

## 4️⃣ RDS + DB Subnet Group — ✅ DELETED

```powershell
aws rds delete-db-instance --db-instance-identifier madar-p05-postgres --skip-final-snapshot --delete-automated-backups --region us-east-1
aws rds wait db-instance-deleted --db-instance-identifier madar-p05-postgres --region us-east-1
aws rds delete-db-subnet-group --db-subnet-group-name madar-p05-db-subnet-group --region us-east-1
```

⚠️ **Actual incident:** the first waiter failed with `ExpiredToken`. A following PowerShell `Write-Host "RDS DELETED"` still printed because it was separated with `;`. I did **not** accept that text as proof. After refreshing the CLI session, I reran the waiter successfully and then deleted/verified the subnet group.

**Lesson:** verification output from AWS is authoritative; decorative shell output is not.

## 5️⃣ Secret + Logs — ✅ DELETED

```powershell
aws secretsmanager delete-secret --secret-id "MADAR/Phase05/Postgres" --force-delete-without-recovery --region us-east-1
aws logs delete-log-group --log-group-name "/ecs/madar-phase05-app" --region us-east-1
```

The secret returned a `DeletionDate`/`DeletedDate`. Force deletion is asynchronous, so temporary `describe-secret` visibility is not unexpected.

## 6️⃣ ECR — ✅ DELETED

```powershell
aws ecr delete-repository --repository-name madar-phase05-app --force --region us-east-1
aws ecr delete-repository --repository-name madar-p05-restore --force --region us-east-1
```

Both deletion calls returned repository metadata; later lookup returned `RepositoryNotFoundException`.

## 7️⃣ Task Definitions + Cluster — ✅ DELETED/INACTIVE

Deregistered:

```text
madar-phase05-app:1
madar-p05-db-restore:1
madar-p05-db-verify:1
```

Then:

```powershell
aws ecs delete-cluster --cluster MADAR-P05-Cluster --region us-east-1
```

Verification: cluster `INACTIVE`, running tasks `0`, active services `0`. AWS can retain inactive task-definition metadata; that is not a running resource.

## 8️⃣ IAM — ✅ DELETED

Execution role:
- detached AWS-managed `AmazonECSTaskExecutionRolePolicy`;
- deleted inline `MADAR-P05-ReadPostgresSecret`;
- deleted role.

Restore role:
- deleted inline `MADAR-P05-Restore-Dump-ReadOnly`;
- deleted role.

```powershell
aws iam list-roles --query "Roles[?contains(RoleName,'MADAR-P05')].RoleName" --output table
```

Final result: no matching Phase 05 roles.

## 9️⃣ Network — ✅ DELETED

Before deletion I checked VPC ENIs and got none. Then I removed RDS-SG → ECS-SG → ALB-SG, four subnets, custom public/private route tables, detached/deleted IGW and finally deleted VPC `vpc-011a441b0a790c458`.

Expected verification errors such as `InvalidGroup.NotFound`, `InvalidInternetGatewayID.NotFound` and `InvalidVpcID.NotFound` were treated as positive proof of absence.

The automatically created **main route table** was intentionally not deleted manually; it disappeared with the VPC.

## 🔍 Final Residual Audit — ✅ CLEAN

The final colored PowerShell audit checked Phase 05 ECS, RDS, ALB, ECR, IAM and VPC and printed `DELETED` when no matching resources remained. It also separately verified retained AMI/snapshot/S3.

Evidence: `../evidence/phase05-residual-audit.png`.

## 💰 Cost Closeout — ✅ CAPTURED

Final available Cost Explorer output was captured in `../evidence/Cost-Closeout-Evidence.png`. Billing data can lag, so the screenshot is a closeout checkpoint—not a guarantee of final settled `$0.00`.

## 🧯 Common Cleanup Traps

- ❌ Delete VPC first → dependency errors.
- ❌ Delete RDS subnet group while DB still exists → dependency error.
- ❌ Delete SGs while ENIs still exist → dependency error.
- ❌ Delete IAM role before detaching/deleting policies → conflict.
- ❌ Trust `Write-Host` after a failed AWS command → false success.
- ❌ Delete retained Phase 03 AMI/snapshot/S3 → breaks later MADAR continuity.

## 🏁 Definition of Done — ACHIEVED

```text
No Phase 05 running Fargate tasks       ✅
No scaling target/policy                ✅
No Phase 05 ALB/TG                      ✅
No Phase 05 RDS/subnet group            ✅
No Phase 05 ECR repositories            ✅
No Phase 05 IAM roles                   ✅
No Phase 05 VPC                         ✅
Residual audit clean                    ✅
Cost closeout captured                  ✅
Phase 03 retained assets untouched      ✅
```

**Cleanup is part of the engineering deliverable, not an afterthought.**