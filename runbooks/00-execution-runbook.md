# 🧰 Phase 05 — Execution Runbook

> 📍 **Checkpoint: 2026-09-03 — execution/validation complete; cleanup next.**  
> This runbook records what was actually executed. Secret values must never be printed or committed.

## 🟢 Gate 0 — Preflight

**COMPLETE / GO** ✅ — account/credits and retained Phase 03 artifacts verified. No AWS account upgrade performed.

## 🟢 Gate 1 — Recover Legacy Application

**COMPLETE** ✅

Source AMI: `ami-0cbd2e9ec0d6f9168`. Flask workload recovered from `/home/madaradmin/madar-legacy-app`, extracted safely, and temporary recovery EC2/EBS/SG removed.

## 🟢 Gate 2 — Containerize Locally

**COMPLETE** ✅

```text
Base       python:3.12-slim
Runtime    gunicorn / 2 workers
Bind       0.0.0.0:8080
User       madar / UID 1000
health     200 locally
ready      503 locally without PostgreSQL — expected
```

Database configuration externalized through `MADAR_DB_HOST`, `MADAR_DB_PORT`, `MADAR_DB_NAME`, `MADAR_DB_USER`, `MADAR_DB_PASSWORD`.

## 🟢 Gate 3 — Application ECR

**COMPLETE** ✅

```text
Repository  madar-phase05-app
Tag         v1
Digest      sha256:2564714f2668c95ab89c81e95e438a63d14c9d66194ea7eda6a34df59ab99346
```

## 🟢 Gate 4 — Network / RDS / Secrets / IAM

**COMPLETE** ✅

VPC `10.60.0.0/16`, two public and two private subnets, IGW/public routing, local-only private routing, no NAT Gateway.

Security chain:

```text
0.0.0.0/0 → ALB-SG :80
ALB-SG     → ECS-SG :8080
ECS-SG     → RDS-SG :5432
```

RDS `madar-p05-postgres`: PostgreSQL 18.3, db.t4g.micro, 20 GB gp3, Single-AZ, private. Secrets Manager name: `MADAR/Phase05/Postgres`.

Execution role `MADAR-P05-ECS-ExecutionRole` configured for ECS execution plus narrowly scoped secret access. CloudWatch log group `/ecs/madar-phase05-app` created with 7-day retention.

## 🟢 Gate 5 — Recover / Restore Legacy Database

**COMPLETE** ✅

S3 did not already contain a full database dump. The retained Phase 03 snapshot was attached to temporary inspection infrastructure and mounted read-only. The newer authoritative PostgreSQL custom dump was recovered and copied to:

`s3://madar-operational-files-197821101770/database-backups/madar_legacy_final.dump`

Temporary inspection resources were then removed:

```text
EC2  i-008ab8e6f83b405c8     terminated
EBS  vol-0717e3317fe26639e   deleted
SG   sg-07a6ab691556e32a5    deleted
```

A dedicated restore image `madar-p05-restore:v1` was built/published. `MADAR-P05-Restore-TaskRole` received only `s3:GetObject` for the exact dump object.

Restore task `5614c8240a514ba8a25b6ed6c281cd36` exited `0`; logs recorded `RESTORE COMPLETED SUCCESSFULLY`. Verification task `fe9e80d5004d4f4a8f9e0638037bda55` also exited `0`.

## 🟢 Gate 6 — ECS / Fargate / ALB

**COMPLETE** ✅

```text
Cluster       MADAR-P05-Cluster
Task def      madar-phase05-app:1
Service       MADAR-P05-App-Service
ALB           MADAR-P05-ALB
Target group  MADAR-P05-TG / type ip / port 8080
Health path   /api/health
```

Validated through ALB:

```text
/api/health  200
/api/ready   200 / database connected
Dashboard    rendered successfully
```

## 🟢 Gate 7 — Reliability / Scaling / Failure

**COMPLETE** ✅

### ♻️ Self-healing

Desired count was raised to 2. Two targets became healthy. One service task was intentionally stopped; ECS started a replacement, ALB drained the old target, and the application remained available. Service returned to desired healthy capacity.

### 📈 Auto Scaling

Application Auto Scaling configured `Min=1`, `Max=2`, CPU target tracking. For controlled proof the target was temporarily lowered from 40% to 5%. Load produced threshold-crossing datapoints; the high alarm entered `ALARM`, target tracking triggered, and ECS desired count changed automatically from 1 to 2. Target was restored to 40% after the test.

Only automatic scale-out is claimed as proven.

### 🔌 RDS dependency failure/recovery

The exact RDS SG ingress rule allowing ECS-SG TCP/5432 was revoked temporarily.

Observed through ALB:

```text
failure: /api/health = 200
failure: /api/ready  = 502
recovery: /api/health = 200
recovery: /api/ready  = 200
```

The rule was restored immediately. Current replacement rule ID is `sgr-0eb29e1e3032db334`.

## 🟢 Gate 8 — Observability / Cost

**COMPLETE** ✅

Captured ECS CPU/memory, ECS service state, ALB target health/request count, RDS state/connections and CloudWatch log streams.

Cost Explorer checkpoint showed negligible/~$0.00 visible usage for the captured 2026-09-01 through 2026-09-03 window. Billing data can lag.

## 🟠 Gate 9 — Cleanup

**NEXT** ▶️

Execute `99-cleanup-runbook.md`. Required final proof:

```text
phase05-final-cleanup.png
phase05-residual-audit.png
```

Preserve Phase 03 retained AMI `ami-0cbd2e9ec0d6f9168`, snapshot `snap-0920a020c47fb6447`, and S3 bucket `madar-operational-files-197821101770`.