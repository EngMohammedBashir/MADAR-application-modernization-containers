# 🧰 Phase 05 — Execution Runbook

> 📍 **Live checkpoint: 2026-09-03.** This runbook now records completed execution gates and the remaining operational path. Secret values must never be printed or committed.

## 🟢 Gate 0 — Preflight

**Status: COMPLETE / GO** ✅

Verified account/credits, Phase 03 retained artifacts and required service access. No AWS account upgrade was performed.

## 🟢 Gate 1 — Recover Legacy Application

**Status: COMPLETE** ✅

Recovery source:

```text
AMI: ami-0cbd2e9ec0d6f9168
```

Recovered workload:

```text
/home/madaradmin/madar-legacy-app
├── app.py
├── templates/
├── static/
├── scripts/
└── seed_data.py
```

The imported VMware-origin AMI retained its historical OS login configuration rather than injecting the selected EC2 key as expected. The application was inspected and extracted safely. No credentials from that host belong in Git.

Temporary recovery resources were subsequently terminated/deleted, including the detached root EBS volume and recovery security group.

## 🟢 Gate 2 — Containerize Locally

**Status: COMPLETE** ✅

Final runtime design:

```text
Base       python:3.12-slim
Runtime    gunicorn
Bind       0.0.0.0:8080
Workers    2
User       madar (non-root)
```

Validated locally:

```text
docker build                         ✅
container starts                     ✅
/api/health                          200 / status ok ✅
/api/ready without local PostgreSQL  503 / expected ✅
container UID                        1000 / non-root ✅
```

Database configuration was externalized through:

```text
MADAR_DB_HOST
MADAR_DB_PORT
MADAR_DB_NAME
MADAR_DB_USER
MADAR_DB_PASSWORD
```

## 🟢 Gate 3 — Amazon ECR

**Status: COMPLETE** ✅

```text
Repository  madar-phase05-app
Tag         v1
Digest      sha256:2564714f2668c95ab89c81e95e438a63d14c9d66194ea7eda6a34df59ab99346
Scan push   enabled
Encryption  AES256
```

## 🟡 Gate 4 — Network / Database / IAM Foundation

**Status: IN PROGRESS** 🔄

### 🌐 Network — COMPLETE

```text
VPC        10.60.0.0/16
Public-A   10.60.1.0/24  us-east-1a
Public-B   10.60.2.0/24  us-east-1b
Private-A  10.60.11.0/24 us-east-1a
Private-B  10.60.12.0/24 us-east-1b
```

Public route table has `0.0.0.0/0 → IGW`. Private route table has local routing only. No NAT Gateway.

### 🛡️ Security Groups — COMPLETE

```text
0.0.0.0/0 → ALB-SG :80
ALB-SG     → ECS-SG :8080
ECS-SG     → RDS-SG :5432
```

### 🐘 RDS — AVAILABLE

```text
Identifier           madar-p05-postgres
Engine               PostgreSQL 18.3
Class                db.t4g.micro
Database             madar_legacy
Storage              20 GB gp3
Single-AZ            yes
PubliclyAccessible   false
```

DB subnet group spans the two private subnets. Current temporary lab storage reports encryption disabled; production hardening should enable encryption at rest.

### 🔐 Secrets Manager — COMPLETE

Secret name:

```text
MADAR/Phase05/Postgres
```

Never retrieve the secret value into screenshots/logs/Git.

### 🔑 IAM — CURRENT STEP

Created:

```text
MADAR-P05-ECS-ExecutionRole
Trust principal = ecs-tasks.amazonaws.com
```

Next actions:

1. attach `AmazonECSTaskExecutionRolePolicy`,
2. add least-privilege permission for the specific Phase 05 secret,
3. create a minimal application Task Role only if needed.

## ⏳ Gate 5 — ECS / Fargate

Remaining:

```text
📊 Create CloudWatch log group
🚀 Create ECS cluster
📋 Register task definition
🏃 Run initial Fargate task
📦 Verify ECR pull
📊 Verify awslogs delivery
🐘 Initialize/validate PostgreSQL schema/data through controlled ECS path
🩺 Verify task → RDS / DB-backed endpoint
```

Initial task resources remain:

```text
CPU     256
Memory  512 MiB
Network awsvpc
Public  ENABLED for short-lived lab
Port    8080
```

## ⏳ Gate 6 — ALB / ECS Service

Create only after initial task/runtime validation:

```text
Target group type  ip
Health path        /api/health
ALB listener       HTTP :80
ALB subnets        Public-A + Public-B
ECS ingress        only from ALB-SG
```

Then create ECS service and validate application traffic through ALB DNS.

## ⏳ Gate 7 — HA / Failure / Scaling

### ♻️ Task Replacement

```text
desiredCount=2
→ two healthy targets
→ intentionally stop one task
→ ECS starts replacement
→ replacement becomes healthy
→ desired count restored
```

### 📈 Auto Scaling

Configure ECS Service Application Auto Scaling target tracking, generate controlled load and capture scale-out plus scale-in.

### 🔌 Database Dependency Failure

Temporarily revoke `ECS-SG → RDS-SG :5432`.

Expected:

```text
/api/health remains healthy
/api/ready or DB-backed endpoint fails
```

Restore the rule and prove recovery.

## ⏳ Gate 8 — Evidence / Cost / Cleanup

Before destructive cleanup capture:

- 📋 task definition/service state,
- ⚖️ ALB healthy targets,
- 🐘 private RDS configuration,
- 📊 logs/metrics,
- ♻️ task replacement,
- 📈 scaling,
- 🔌 dependency failure/recovery,
- 💰 Cost Explorer checkpoint.

Then execute `99-cleanup-runbook.md` and run a residual-resource audit.

## 🧭 Current Resume Point

```text
🔑 Finish ECS Execution Role permissions
      ↓
📊 CloudWatch log group
      ↓
🚀 ECS cluster
      ↓
📋 Task definition
      ↓
🏃 Initial Fargate task
```
