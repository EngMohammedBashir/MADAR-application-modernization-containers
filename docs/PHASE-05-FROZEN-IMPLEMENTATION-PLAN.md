# 📋 Phase 05 — Frozen Implementation Plan

> 🔒 **Architecture baseline remains frozen.**  
> 🟠 **Execution checkpoint: 2026-09-03 — build/validation complete; cleanup next.**  
> This document preserves the approved architecture while recording actual execution. Live truth is in `../CURRENT-STATE.md`.

## 🎯 Objective

Modernize MADAR's existing Flask workload from a VM-origin runtime into a portable containerized ECS/Fargate service while preserving PostgreSQL as an external managed data layer, then prove reliability, scaling, dependency behavior, observability and cleanup discipline.

## 🔗 Executed Continuity

```text
Phase 03 retained AMI                 ✅
  → temporary recovery EC2            ✅
  → Flask source extraction           ✅
  → Docker + Gunicorn + non-root      ✅
  → application ECR v1                ✅
  → Phase 05 VPC / SG / private RDS   ✅
  → retained snapshot DB recovery     ✅
  → S3-backed one-off restore task    ✅
  → ECS Fargate service               ✅
  → ALB                               ✅
  → self-healing                      ✅
  → target-tracking scale-out         ✅
  → RDS dependency failure/recovery   ✅
  → observability + cost evidence     ✅
  → destructive cleanup              ⏳ NEXT
```

## 🏗️ Frozen Lab Architecture

```text
🌍 Internet
  ↓ HTTP :80
⚖️ ALB — two public subnets
  ↓ TCP :8080
🚀 ECS Service / Fargate
  ├── awsvpc
  ├── public subnets + public IPv4
  ├── 0.25 vCPU / 512 MiB
  ├── desired baseline 1
  └── ingress only from ALB-SG
  ↓ TCP :5432
🐘 RDS PostgreSQL
  ├── private subnets
  ├── Single-AZ
  ├── db.t4g.micro
  ├── 20 GB gp3
  └── PubliclyAccessible=false
```

Supporting services: ECR, Secrets Manager, CloudWatch Logs/Metrics and least-privilege IAM.

## 🗃️ Database Recovery Addition

The implementation discovered that the retained S3 operational bucket did not already contain a full PostgreSQL dump. The retained Phase 03 EBS snapshot was therefore inspected read-only using temporary resources. The newer authoritative custom dump was recovered, copied to the retained S3 bucket under `database-backups/`, and restored through a dedicated one-off Fargate task.

The restore task role was restricted to `s3:GetObject` for the single dump object. The restore task exited `0` and logged successful completion. Temporary inspection EC2/EBS/SG resources were removed.

## 🐳 Container Design — Implemented

- ✅ `python:3.12-slim`
- ✅ Gunicorn / 2 workers
- ✅ non-root `madar` user
- ✅ pinned dependencies
- ✅ stdout/stderr logging
- ✅ external `MADAR_DB_*` configuration
- ✅ `/api/health` liveness
- ✅ `/api/ready` DB readiness
- ✅ `.dockerignore`

## 🌐 Networking / Security — Implemented

```text
0.0.0.0/0 → ALB-SG :80       ✅
ALB-SG     → ECS-SG :8080     ✅
ECS-SG     → RDS-SG :5432     ✅
0.0.0.0/0 → ECS-SG            🚫
0.0.0.0/0 → RDS-SG            🚫
```

No NAT Gateway was used. Public Fargate ENIs are a deliberate short-lived lab trade-off; production guidance remains private tasks with controlled egress/VPC endpoints as appropriate.

## 🔑 IAM — Implemented

`MADAR-P05-ECS-ExecutionRole` supplies ECR pull, CloudWatch logging and narrowly scoped Secrets Manager retrieval for task startup.

`MADAR-P05-Restore-TaskRole` supplies only the S3 object read needed by the one-off restore task.

Application code is not given broad AWS API permissions.

## ⚖️ ECS / ALB — Implemented

```text
Cluster       MADAR-P05-Cluster
Service       MADAR-P05-App-Service
App task def  madar-phase05-app:1
Target group  MADAR-P05-TG / ip / :8080
Health path   /api/health
ALB            MADAR-P05-ALB / HTTP :80
```

Validated through the ALB: health `200`, readiness `200`, and the MADAR dashboard rendered successfully.

## 🧪 Validation Gates

| Gate | Requirement | State |
|---|---|---|
| 🔎 A | AMI recovery / source extraction / recovery cleanup | ✅ PASS |
| 🐳 B | Docker / health / non-root / external config | ✅ PASS |
| 📦 C | ECR application image publication | ✅ PASS |
| 🗃️ D | Legacy DB dump recovery + controlled restore | ✅ PASS |
| 🚀 E | Fargate runtime + logs + DB-backed application | ✅ PASS |
| ⚖️ F | ALB healthy target + public functional path | ✅ PASS |
| ♻️ G | desiredCount=2 + intentional task replacement | ✅ PASS |
| 📈 H | target-tracking automatic scale-out | ✅ PASS |
| 🔌 I | RDS dependency failure + recovery | ✅ PASS |
| 📊 J | observability + cost checkpoint | ✅ PASS |
| 🧹 K | destructive cleanup + residual audit | ⏳ NEXT |

### Claim boundary

Automatic scale-out is proven. A separate automatic scale-in event is not claimed. During RDS failure injection, `/api/health` remained `200` and `/api/ready` was observed as `502` through the ALB; after restoring TCP/5432 both returned `200`.

## 📈 Auto Scaling Final Configuration

```text
MinCapacity        1
MaxCapacity        2
Metric             ECSServiceAverageCPUUtilization
TargetValue        40%
ScaleOutCooldown   60s
ScaleInCooldown    60s
```

For the controlled test only, TargetValue was temporarily set to 5%. Three datapoints crossed the test threshold, the high alarm entered `ALARM`, the policy triggered, and desired count moved automatically from 1 to 2. TargetValue was restored to 40% immediately afterward.

## 📊 Observability — Captured

Evidence includes ECS CPU/memory, service/task state, ALB target health/request volume, RDS state/connections and CloudWatch log streams.

## 💰 Cost Controls

No NAT Gateway, EKS, Multi-AZ RDS, WAF, custom domain, Terraform or CI/CD was added. The pre-cleanup Cost Explorer screenshot showed negligible/~$0.00 visible usage for the captured period; billing data may lag.

## 🧭 Implementation Checklist

```text
01–21  foundation / Docker / ECR / VPC / RDS       ✅
22     legacy DB recovery + restore                  ✅
23–30  Secrets / IAM / logs / ECS runtime / DB       ✅
31–35  target group / ALB / service / functional     ✅
36–37  two-task test / self-healing                   ✅
38–39  target tracking / controlled scale-out         ✅
40     ECS→RDS dependency failure/recovery            ✅
41     logs/metrics/evidence                           ✅
42     Cost Explorer checkpoint                       ✅
43     cleanup runbook                                ▶️ NEXT
44     residual-resource audit                         ⏳
45     final repository closeout                       ⏳
46     MADAR master repository update                  ⏳
```

## 🧹 Cleanup Order

```text
📈 scaling
→ 🚀 ECS service/tasks
→ ⚖️ ALB/listener/TG
→ 📋 app/restore/verify task definitions + cluster
→ 🐘 RDS + DB subnet group
→ 🔐 secret + 📊 log group
→ 📦 both Phase 05 ECR repositories
→ 🔑 Phase 05 IAM roles/policies
→ 🛡️ SGs
→ 🌐 subnets/routes/IGW/VPC
→ 🔍 residual audit
→ 💰 final cost closeout
```

Do not delete the retained Phase 03 AMI, snapshot or operational S3 bucket as part of this runbook.

## 🏁 Closeout Rule

Phase 05 is complete only after the destructive cleanup and residual audit are proven. Running infrastructure is a validation state, not the final state.