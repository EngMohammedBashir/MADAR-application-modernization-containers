# 📍 Phase 05 — Current State

**Status:** 🟠 VALIDATION COMPLETE / CLEANUP NEXT  
**Region:** `us-east-1`  
**Last synchronized:** 2026-09-03

## 🚦 Executive State

```text
🧾 Gate 0 / account & cost preflight       ✅ PASS / GO
🔎 Legacy AMI recovery                     ✅ COMPLETE
🐳 Containerization + local validation     ✅ COMPLETE
📦 Application ECR image                   ✅ v1 PUBLISHED
🌐 VPC / routing / security groups         ✅ COMPLETE
🐘 Private RDS PostgreSQL                  ✅ AVAILABLE
🗃️ Legacy PostgreSQL dump recovery         ✅ COMPLETE
♻️ Controlled DB restore on Fargate        ✅ COMPLETE
🔐 Secrets Manager + IAM                   ✅ COMPLETE
🚀 ECS / Fargate service                   ✅ RUNNING
⚖️ ALB + healthy target                    ✅ COMPLETE
🩺 End-to-end app → DB validation          ✅ PASS
♻️ Task self-healing                       ✅ PASS
📈 Target-tracking scale-out               ✅ PASS
🔌 RDS dependency failure/recovery         ✅ PASS
📊 Observability evidence                  ✅ CAPTURED
💰 Cost checkpoint                         ✅ CAPTURED (~$0.00 visible)
🧹 Final cleanup                           ⏳ NEXT
🔍 Residual audit                          ⏳ AFTER CLEANUP
```

## 🔗 Phase 03 Continuity

Intentionally retained artifacts:

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

The retained AMI supplied the application source. The retained snapshot later supplied the authoritative PostgreSQL custom dump after S3 was found to contain operational exports/reports but not the database backup needed for full restoration.

The recovered dump was copied to:

```text
s3://madar-operational-files-197821101770/database-backups/madar_legacy_final.dump
```

Temporary snapshot-inspection EC2/EBS/SG resources were removed after recovery.

## 🐳 Application Runtime

```text
App image       madar-phase05-app:v1
Base            python:3.12-slim
Runtime         Gunicorn / 2 workers
Port            8080
User            madar / non-root
Liveness        /api/health
Readiness       /api/ready
ECS cluster     MADAR-P05-Cluster
ECS service     MADAR-P05-App-Service
Task definition madar-phase05-app:1
CPU / memory    256 / 512 MiB
```

The service is reachable only through the ALB on the application path; direct inbound TCP/8080 from the Internet is not permitted by the ECS security group.

## ⚖️ ALB / Service Path

```text
ALB             MADAR-P05-ALB
DNS             MADAR-P05-ALB-727499775.us-east-1.elb.amazonaws.com
Listener        HTTP :80
Target group    MADAR-P05-TG
Target type     ip
Target port     8080
Health path     /api/health
```

Validated through the ALB:

```text
/api/health  → HTTP 200
/api/ready   → HTTP 200 with PostgreSQL connected
Dashboard    → rendered successfully
```

## 🐘 Database / Restore

```text
RDS ID               madar-p05-postgres
Engine               PostgreSQL 18.3
Database             madar_legacy
Class                db.t4g.micro
Storage              20 GB gp3
Single-AZ            yes
PubliclyAccessible   false
DB subnet group      madar-p05-db-subnet-group
```

Restore path:

```text
Retained Phase 03 snapshot
  → read-only inspection
  → madar_legacy_final.dump
  → S3 database-backups/
  → temporary restore container
  → one-off Fargate restore task
  → RDS PostgreSQL
```

Restore task `5614c8240a514ba8a25b6ed6c281cd36` exited `0` and CloudWatch recorded `RESTORE COMPLETED SUCCESSFULLY`. A separate DB verification task also exited `0`.

Temporary restore artifacts currently include:

```text
ECR repo       madar-p05-restore:v1
Task role      MADAR-P05-Restore-TaskRole
Task def       madar-p05-db-restore:1
Verify taskdef madar-p05-db-verify:1
```

These are Phase 05 cleanup targets.

## 🔐 Security / IAM

```text
Internet :80
   ↓
ALB-SG  sg-00b9b70e13293ff46
   ↓ :8080
ECS-SG  sg-0d13f6af551e284c8
   ↓ :5432
RDS-SG  sg-00ae439cb916d164b
```

The RDS ingress rule was deliberately revoked during failure injection and then restored. Current restored rule ID: `sgr-0eb29e1e3032db334`.

Execution role: `MADAR-P05-ECS-ExecutionRole` with standard ECS execution permissions plus narrowly scoped secret retrieval. Restore task role: `MADAR-P05-Restore-TaskRole` with `s3:GetObject` limited to the single recovered dump object.

## ♻️ Reliability Tests

### Task self-healing — PASS

At desired count 2, one service task was intentionally stopped. ECS detected the mismatch, started a replacement, ALB drained the old target, and `/api/ready` remained HTTP 200 during the event. The service returned to two running tasks and healthy targets.

### Target-tracking scale-out — PASS

Application Auto Scaling:

```text
Min capacity       1
Max capacity       2
Metric             ECSServiceAverageCPUUtilization
Final target       40%
Cooldowns          60s / 60s
```

For controlled validation only, the target was temporarily lowered to 5%. Load drove average CPU above the test threshold, the CloudWatch high alarm entered `ALARM`, the target-tracking policy triggered, and ECS changed desired count from 1 to 2 automatically. The policy was then restored to the intended 40% target.

Only scale-out is claimed as validated; no unsupported scale-in claim is made.

### RDS dependency failure/recovery — PASS

The ECS→RDS TCP/5432 SG rule was revoked intentionally:

```text
During failure: /api/health = 200
During failure: /api/ready  = 502 observed through ALB
After restore:  /api/health = 200
After restore:  /api/ready  = 200
```

This demonstrated that application liveness remained healthy while the DB-backed readiness path failed, then recovered after the exact dependency rule was restored.

## 📊 Observability

Captured evidence includes:

- ECS service/task state,
- ALB target health,
- RDS availability/private state,
- CloudWatch log streams,
- ECS CPU and memory,
- ALB request volume,
- RDS database connections.

## 💰 Cost Checkpoint

Cost Explorer checkpoint for 2026-09-01 through 2026-09-03 showed only negligible values (`~$0.00` visible at capture time). Cost Explorer can lag, so this is a checkpoint rather than a claim of final zero cost.

## ▶️ NEXT — Destructive Cleanup

1. Remove Application Auto Scaling policy/scalable target.
2. Scale service to zero and delete it.
3. Delete ALB/listener/target group.
4. Deregister app/restore/verify task definitions and delete ECS cluster.
5. Delete RDS, DB subnet group, secret and CloudWatch log group.
6. Delete both Phase 05 ECR repositories.
7. Remove Phase 05 IAM roles/policies.
8. Delete SGs, subnets, route tables, IGW and Phase 05 VPC.
9. Run residual-resource audit and final cost closeout.
10. Preserve the Phase 03 AMI/snapshot/S3 unless a separate retention decision is made.

📸 Evidence is indexed in `evidence/README.md`.