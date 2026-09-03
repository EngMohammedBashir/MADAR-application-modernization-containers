# 📋 Phase 05 — Frozen Implementation Plan

> 🔒 **Architecture baseline remains frozen.**  
> 🟡 **Execution checkpoint: 2026-09-03 — implementation is actively in progress.**  
> This document preserves the approved plan while marking what has actually been completed. Live truth is summarized in `../CURRENT-STATE.md`.

## 🎯 Objective

Modernize MADAR's existing Flask application runtime from a VM-origin workload into a containerized ECS/Fargate service while preserving PostgreSQL as an external managed data layer.

## 🔗 Business Continuity

```text
Phase 03
Legacy VMware VM
  → EC2 rehost
  → PostgreSQL replatform to RDS
  → operational files to S3
  → temporary migration resources cleaned up
  → AMI + backing snapshot + S3 retained

Phase 05
Retained AMI                         ✅
  → temporary recovery EC2           ✅
  → extract existing Flask app       ✅
  → remove VM assumptions            ✅
  → Dockerize                        ✅
  → ECR v1                           ✅
  → VPC / security / private RDS     ✅
  → Secrets Manager                  ✅
  → IAM execution role               🔄 current
  → ECS Fargate                      ⏳
  → ALB                              ⏳
  → failure / recovery / scaling     ⏳
  → cleanup                          ⏳
```

## 🏗️ Frozen Lab Architecture

```text
🌍 Internet
  ↓
🚪 Internet Gateway
  ↓
⚖️ ALB in two public subnets / two AZs
  ↓
🎯 Target Group — target type IP
  ↓
🚀 ECS Service / Fargate
  ├── awsvpc
  ├── public subnets
  ├── assignPublicIp = ENABLED
  ├── desiredCount = 1 baseline
  ├── 0.25 vCPU / 512 MiB
  └── inbound only from ALB-SG
  ↓ TCP/5432
🐘 RDS PostgreSQL
  ├── private subnets
  ├── Single-AZ
  ├── db.t4g.micro
  ├── 20 GB gp3
  ├── not publicly accessible
  └── inbound only from ECS-SG
```

## 🧩 Supporting Services

Required:

- 📦 ECR private repository — ✅ created/published,
- 🔐 Secrets Manager — ✅ created,
- 📊 CloudWatch Logs — ⏳ pending ECS runtime,
- 🔑 IAM execution role — 🔄 role created; permissions being completed,
- 🧑‍💻 Task Role — only if runtime AWS API access is required.

Not planned:

- 🚫 NAT Gateway,
- 🚫 VPC interface endpoints for this short-lived lab,
- 🚫 EKS,
- 🚫 Multi-AZ RDS,
- 🚫 Route 53/custom domain,
- 🚫 WAF,
- 🚫 Terraform,
- 🚫 CI/CD.

## 🌐 Networking Decision

For this short-lived lab, public Fargate ENIs are a deliberate cost/complexity trade-off. Direct ingress to tasks remains blocked by Security Groups.

```text
0.0.0.0/0 → ALB-SG :80       ✅ configured
ALB-SG     → ECS-SG :8080     ✅ configured
ECS-SG     → RDS-SG :5432     ✅ configured
0.0.0.0/0 → ECS-SG            🚫 denied
0.0.0.0/0 → RDS-SG            🚫 denied
```

Production guidance remains private tasks + controlled egress.

## 🐳 Container Design

Implemented requirements:

- ✅ `python:3.12-slim`,
- ✅ Gunicorn,
- ✅ non-root user,
- ✅ pinned dependencies,
- ✅ stdout/stderr logging,
- ✅ external database configuration,
- ✅ `/api/health` liveness,
- ✅ `/api/ready` dependency readiness,
- ✅ `.dockerignore` for local/secret/unnecessary artifacts.

The local container passed liveness and non-root validation. Readiness correctly returned `503` without a local PostgreSQL dependency.

## 🐘 RDS Design

Live implementation:

```text
PostgreSQL 18.3
Single-AZ
Private / PubliclyAccessible=false
20 GB gp3
db.t4g.micro
Deletion protection OFF
Backup retention 0
```

⚠️ Execution note: the current temporary lab instance reports storage encryption disabled. Production hardening should enable encryption at rest. This does not change the frozen lab's short-lived validation objective.

## 🔑 IAM

### 🏗️ Task Execution Role

Used by ECS/Fargate infrastructure for:

- ECR authentication/image pull,
- CloudWatch log publication,
- retrieval/injection of the specific Secrets Manager values referenced by the task definition.

Current state:

```text
MADAR-P05-ECS-ExecutionRole  ✅ role created
Trust: ecs-tasks.amazonaws.com
Managed execution policy     ⏳ next
Secret-specific access       ⏳ next
```

### 🧑‍💻 Task Role

Used by application code at runtime. Default remains no AWS API permissions unless the recovered workload proves a requirement.

## ❤️ Health Model

```text
/api/health
  → process/application liveness
  → ALB health check

/api/ready
  → database/dependency readiness
  → functional/dependency failure validation
```

## 🚦 Validation Gates

| Gate | Requirement | State |
|---|---|---|
| 🔎 A | AMI recovery / safe extraction / recovery cleanup | ✅ PASS |
| 🐳 B | Docker build / local health / non-root / external config | ✅ PASS |
| 📦 C | ECR versioned image publication | ✅ PASS |
| 🚀 D | ECS task + logs + RDS-backed API | 🔄 NEXT |
| ⚖️ E | ALB healthy target + public functional path | ⏳ |
| ♻️ F | desiredCount=2 + task replacement | ⏳ |
| 📈 G | controlled auto scaling | ⏳ |
| 🔌 H | RDS dependency failure + recovery | ⏳ |

## 📊 Observability

Minimum useful set remains:

- CloudWatch Logs,
- ECS CPU/memory,
- ALB target health / RequestCount / TargetResponseTime / HealthyHostCount,
- RDS basic metrics.

No custom dashboard/SNS/alarms unless they add genuine validation value.

## 🔓 HTTPS Decision

Lab: HTTP on ALB for short-lived validation.  
Production: ACM certificate + HTTPS listener + HTTP→HTTPS redirect.

## 🧭 Implementation Checklist

```text
01 ✅ Gate 0 account/cost preflight
02 ✅ Verify retained AMI/snapshot/S3
03 ✅ Launch temporary recovery EC2
04 ✅ Locate/inspect Flask workload
05 ✅ Export safe source
06 ✅ Terminate recovery EC2 + EBS/SG cleanup
07 ✅ Sanitize application/configuration
08 ✅ Dependency manifest
09 ✅ Dockerfile / .dockerignore
10 ✅ Local image build
11 ✅ Local liveness test
12 ✅ Local readiness/config behavior
13 ✅ ECR repository
14 ✅ Push v1 image
15 ✅ Phase 05 VPC
16 ✅ Two public subnets
17 ✅ Two private DB subnets
18 ✅ IGW + route tables + associations
19 ✅ ALB-SG / ECS-SG / RDS-SG
20 ✅ DB subnet group
21 ✅ RDS PostgreSQL available
22 ⏳ Initialize schema/test data through controlled ECS path
23 ✅ Secrets Manager secret
24 🔄 ECS execution role / minimum task role
25 ⏳ CloudWatch log group
26 ⏳ ECS cluster
27 ⏳ Task definition
28 ⏳ Initial Fargate task
29 ⏳ Verify logs/ECR pull
30 ⏳ Verify task→RDS + DB-backed behavior
31 ⏳ Target group (ip)
32 ⏳ ALB/listener
33 ⏳ ECS service behind ALB
34 ⏳ Validate /api/health via ALB
35 ⏳ Validate DB-backed endpoint
36 ⏳ desiredCount=2
37 ⏳ task failure/self-healing
38 ⏳ target tracking auto scaling
39 ⏳ controlled load / scale-out/in
40 ⏳ ECS→RDS dependency failure/recovery
41 ⏳ logs/metrics/evidence
42 ⏳ Cost Explorer checkpoint
43 ⏳ cleanup runbook
44 ⏳ residual-resource audit
45 ⏳ final project repository closeout
46 ⏳ MADAR master repository update
```

## 💰 Cost Controls

```text
🐘 RDS       continuous while DB exists
⚖️ ALB       hourly + LCU once created
🚀 Fargate   CPU/RAM while tasks run
🌐 IPv4      while public addresses are in use
🔐 Secrets   storage/API
📦 ECR       image storage
📊 Logs      ingestion/storage
```

Rules remain: create ALB late, keep desired count at 1 except tests, no NAT Gateway, and delete cost-bearing resources promptly after evidence.

## 🧹 Cleanup Order

High level:

```text
📈 remove scaling
→ 🚀 ECS service/tasks
→ ⚖️ ALB/listener/TG
→ 🐘 RDS
→ 🔐 secret/log decisions
→ 📦 ECR cleanup decision
→ 🛡️ SGs
→ 🌐 subnets/routes/IGW/VPC
→ 🔍 residual audit
→ 💰 final cost closeout
```

See `../runbooks/99-cleanup-runbook.md` for the operational sequence.

## 🏁 Closeout Rule

Phase 05 is complete only after functionality, HA/self-healing, scaling, dependency failure/recovery, evidence, cost and destructive cleanup are all proven. Running resources alone are not success.
