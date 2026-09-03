# 📸 Phase 05 — Evidence Index

> 🟢 **Live evidence repository.** Screenshots are captured only at meaningful engineering milestones. Never commit passwords, access keys, tokens, secret values, private keys or sensitive console output.

## 🗂️ Current Evidence

All screenshots currently live in this single `evidence/` directory. They are indexed here by engineering stage so the repository stays easy to review even without nested folders.

### 🔎 01 — Legacy Recovery

| File | Proof |
|---|---|
| `recovery-ec2-legacy-app-discovered.png` | 🔎 Retained Phase 03 AMI successfully recovered and the existing Flask workload identified |
| `recovery-resources-cleaned-up.png` | 🧹 Temporary recovery EC2/EBS/security group cleanup verified |

### 🐳 02 — Containerization

| File | Proof |
|---|---|
| `docker-build-success.png` | 🏗️ Docker image built successfully |
| `docker-container-health-success.png` | ❤️ Local container started and application health endpoint passed |

Additional validated facts recorded in project docs: Gunicorn runtime, port `8080`, non-root `madar` user and separate liveness/readiness behavior.

### 📦 03 — Amazon ECR

| File | Proof |
|---|---|
| `ecr-image-push-success.png` | 🚀 Docker image push to private ECR succeeded |
| `ecr-console-v1-verified.png` | 🏷️ `v1` image index/tag/digest visible in ECR |

### 🌐 04 — Networking

| File | Proof |
|---|---|
| `public-network-routing-validated.png` | 🌍 Phase 05 VPC public routing, two public subnets and IGW path validated |

Private routing is recorded in `CURRENT-STATE.md`: `Private-A` and `Private-B` use the dedicated private route table with local-only routing and no NAT Gateway.

### 🛡️ 05 — Security

| File | Proof |
|---|---|
| `security-group-chain-validated.png` | 🛡️ `Internet :80 → ALB-SG → :8080 ECS-SG → :5432 RDS-SG` validated |

This proves ECS and RDS do not accept direct Internet ingress.

---

## 📷 Evidence Still To Capture

### 🐘 RDS / Secrets

Capture one clean view proving:

```text
PostgreSQL available
PubliclyAccessible = false
DB subnet group = private subnets
Single-AZ / db.t4g.micro / 20 GB gp3
RDS-SG attached
```

Suggested filename:

`rds-private-postgres-validated.png`

Do **not** reveal the Secrets Manager secret value.

### 🔑 IAM / ECS Task Definition

Capture:

- ECS execution role,
- task definition CPU/memory,
- `awsvpc`,
- ECR `v1` image,
- `awslogs`,
- secret injection configuration without values.

Suggested filename:

`ecs-task-definition-security-validated.png`

### 🚀 Initial Fargate Runtime

Capture:

- task `RUNNING`,
- image pull success,
- CloudWatch log stream,
- task networking/security group.

Suggested filename:

`ecs-fargate-task-running.png`

### 🗃️ Database Functionality

Capture a deterministic DB-backed MADAR API response proving Flask → PostgreSQL connectivity.

Suggested filename:

`database-api-validation.png`

### ⚖️ ALB

Capture:

- ALB across both public AZs,
- target group type `ip`,
- healthy Fargate target,
- `/api/health` response through ALB DNS.

Suggested filename:

`alb-healthy-fargate-target.png`

### ♻️ Self-Healing

Capture the sequence:

```text
2 healthy tasks
→ intentionally stop one
→ desired-count mismatch
→ replacement task starts
→ target becomes healthy
→ desired count restored
```

Suggested filename:

`ecs-task-self-healing.png`

### 📈 Auto Scaling

Capture baseline, target tracking, controlled load, CPU threshold crossing, scale-out and scale-in.

Suggested filename:

`ecs-auto-scaling-validated.png`

### 🔌 Dependency Failure / Recovery

Capture:

```text
ECS→RDS :5432 allowed  → DB works
rule revoked           → readiness/DB call fails
rule restored          → functionality recovers
```

Suggested filename:

`rds-dependency-failure-recovery.png`

### 📊 Observability

Capture meaningful CloudWatch/ECS/ALB/RDS metrics and logs used during validation.

Suggested filename:

`observability-validation.png`

### 💰🧹 Cost & Cleanup

Final evidence must prove cost checkpoint plus removal of ECS service/tasks, ALB/TG, RDS, secret/log decision, ECR cleanup decision, SGs, subnets/routes/IGW/VPC and residual-resource audit.

Suggested filenames:

```text
phase05-cost-checkpoint.png
phase05-final-cleanup.png
phase05-residual-audit.png
```

---

## 🧠 Evidence Rule

A screenshot belongs here only if it supports a claim a reviewer would care about:

**architecture • security • functionality • failure behavior • scaling • observability • cost • cleanup**.

Routine command output does not need a screenshot.
