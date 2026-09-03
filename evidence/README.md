# 📸 Phase 05 — Evidence Index

> 🟢 Screenshots are kept in this single `evidence/` directory and are included only when they support an engineering claim. No passwords, access keys, tokens, private keys or secret values belong here.

## 🔎 Legacy Recovery

| File | Proof |
|---|---|
| `recovery-ec2-legacy-app-discovered.png` | Retained Phase 03 AMI recovered and Flask workload identified |
| `recovery-resources-cleaned-up.png` | Temporary recovery EC2/EBS/SG cleanup verified |
| `legacy-database-dump-recovered.png` | Authoritative PostgreSQL custom dump recovered from retained Phase 03 snapshot |

## 🐳 Containerization & ECR

| File | Proof |
|---|---|
| `docker-build-success.png` | Application Docker image built successfully |
| `docker-container-health-success.png` | Local container and health behavior validated |
| `ecr-image-push-success.png` | Application image pushed to private ECR |
| `ecr-console-v1-verified.png` | Application `v1` image verified in ECR |
| `MADAR-P05-Restore-Image-QA.png` | Temporary DB restore image contains required PostgreSQL/AWS tooling |
| `MADAR-P05-ECR-Restore-Repository.png` | Restore ECR repository created |
| `MADAR-P05-Restore-Image-Pushed.png` | Restore image `v1` published successfully |

## 🌐 Networking & Security

| File | Proof |
|---|---|
| `public-network-routing-validated.png` | Public subnet/IGW routing validated |
| `security-group-chain-validated.png` | `Internet :80 → ALB → ECS :8080 → RDS :5432` SG chain validated |

## 🚀 Fargate / Database / ALB

| File | Proof |
|---|---|
| `ecs-fargate-task-running.png` | Fargate runtime reached RUNNING and container execution was established |
| `database-api-validation.png` | DB-backed application path validated through the deployed runtime |
| `alb-healthy-fargate-target.png` | ALB target registered healthy |
| `madar-dashboard-on-fargate.png` | MADAR dashboard rendered through the Fargate/ALB deployment |

The database restore task exited `0` and CloudWatch recorded `RESTORE COMPLETED SUCCESSFULLY`. A separate DB verification task also exited `0`. Exact row-count output is not claimed unless separately evidenced.

## ♻️ Reliability / Self-Healing

| File | Proof |
|---|---|
| `ecs-two-healthy-fargate-targets.png` | Two healthy service targets established for the HA/failure test |
| `ecs-task-self-healing.png` | Intentional task stop caused desired/running mismatch and replacement activity |
| `ecs-self-healing-recovered.png` | ECS returned the service to the desired healthy state |

During the intentional task failure, the application remained available through the surviving target while ECS launched a replacement.

## 📈 Auto Scaling

| File | Proof |
|---|---|
| `ecs-auto-scaling-triggered.png` | CloudWatch high alarm triggered the target-tracking policy and ECS desired count moved automatically from 1 to 2 |

For the controlled test only, the CPU target was temporarily lowered from 40% to 5%. CPU crossed the threshold for the required evaluation periods, the alarm entered `ALARM`, and Application Auto Scaling initiated scale-out. The policy was restored to **40%** afterward.

**Claim boundary:** automatic **scale-out** is validated. This repository does not claim a separately evidenced automatic scale-in event.

## 🔌 RDS Dependency Failure & Recovery

| File | Proof |
|---|---|
| `rds-dependency-failure.png` | After revoking ECS→RDS TCP/5432, `/api/health` remained `200` while `/api/ready` returned `502` through the ALB |
| `rds-dependency-recovery.png` | After restoring the SG rule, `/api/health` and `/api/ready` returned `200` |

The observed `502` is recorded exactly as observed; it is not rewritten as the application's intended internal `503` response.

## 📊 Observability

| File | Proof |
|---|---|
| `observability-infrastructure-health.png` | ECS service, ALB target health, private/available RDS and CloudWatch log streams viewed together |
| `observability-cloudwatch-metrics.png` | ECS CPU/memory, ALB request and RDS connection metrics captured from CloudWatch |

## 💰 Cost

| File | Proof |
|---|---|
| `phase05-cost-checkpoint.png` | Cost Explorer checkpoint showed negligible/~$0.00 visible usage for the captured period |

Cost Explorer can lag. This screenshot is a checkpoint, not a guarantee that final settled cost is exactly zero.

## ⏳ Evidence Still Required

Only final closeout evidence remains:

```text
phase05-final-cleanup.png
phase05-residual-audit.png
```

The residual audit must check ECS, ALB/TG, RDS/subnet group, Phase 05 VPC/network resources, ECR, Secrets Manager, CloudWatch Logs, IAM roles and public IPv4/ENIs as applicable.

## 🧠 Evidence Rule

A screenshot belongs here only if it supports a reviewer-relevant claim:

**architecture • security • functionality • recovery • failure behavior • scaling • observability • cost • cleanup**.