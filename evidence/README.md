# Phase 05 Evidence Index

> No execution screenshots are claimed yet. This file defines the evidence we must capture during the build.

## Gate 0 — account / cost baseline

Planned evidence:

- current account plan / credit balance,
- cost baseline before Phase 05,
- retained AMI available,
- retained snapshot available,
- retained S3 operational data visible,
- Phase 04 resources still absent.

## Legacy recovery

Capture:

- temporary EC2 launched from retained AMI,
- Flask application path/runtime identified,
- service/start command identified,
- safe source extracted,
- temporary recovery EC2 terminated.

Do not screenshot or commit passwords/credentials.

## Containerization

Capture:

- successful Docker build,
- final image size,
- non-root runtime identity,
- local `/api/health` response,
- local functional application response,
- absence of secrets from committed files.

## ECR

Capture:

- repository,
- image tag,
- image digest,
- push success.

## RDS / security

Capture:

- PostgreSQL engine/class,
- Single-AZ,
- `PubliclyAccessible = false`,
- private DB subnet group,
- RDS-SG inbound 5432 from ECS-SG only,
- deletion protection disabled for temporary lab.

## ECS / Fargate

Capture:

- task definition CPU/memory,
- `awsvpc`,
- image URI/digest,
- execution role/task role distinction,
- `awslogs`,
- task running,
- public-IP-enabled lab networking without direct Internet ingress.

## ALB

Capture:

- ALB spans two subnets/AZs,
- target group type `ip`,
- healthy Fargate task target(s),
- `/api/health` via ALB DNS.

## Database functionality

Capture a DB-backed API response showing deterministic MADAR data from PostgreSQL.

## Self-healing

Capture sequence:

```text
2 healthy tasks
→ stop one task intentionally
→ service sees desired-count mismatch
→ replacement task starts
→ target becomes healthy
→ desired count restored
```

Prefer screenshots with timestamps/task IDs.

## Auto Scaling

Capture:

- baseline task count/CPU,
- target tracking policy,
- load generation,
- CPU threshold crossing,
- scale-out,
- new target healthy,
- scale-in after load ends.

## Dependency failure / recovery

Capture:

```text
ECS→RDS 5432 allowed   → DB endpoint works
rule removed           → DB readiness/function fails
rule restored          → functionality recovers
```

The application liveness endpoint should remain healthy if the application process itself is running.

## Cleanup / cost

Final evidence must prove:

- ECS service/tasks removed,
- ALB/target group removed,
- RDS removed,
- public IPv4/NAT residuals absent,
- Phase 05 VPC removed,
- ECR/secret/log-group cleanup decision recorded,
- final Cost Explorer checkpoint captured.

## Naming convention

Use readable names such as:

```text
phase05-account-cost-preflight.png
phase05-legacy-ami-recovery.png
phase05-docker-local-validation.png
phase05-ecr-image-published.png
phase05-private-rds-security.png
phase05-ecs-fargate-running.png
phase05-alb-healthy-targets.png
phase05-database-api-validation.png
phase05-task-self-healing.png
phase05-auto-scaling.png
phase05-rds-dependency-failure-recovery.png
phase05-final-cleanup-cost-closeout.png
```
