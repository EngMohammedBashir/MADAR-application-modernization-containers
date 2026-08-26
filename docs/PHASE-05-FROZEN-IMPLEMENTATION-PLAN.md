# Phase 05 — Frozen Implementation Plan

**Status:** Architecture frozen; live account preflight required before deployment.

## 1. Objective

Modernize MADAR's existing Flask application runtime from a VM-origin workload into a containerized ECS/Fargate service while preserving PostgreSQL as an external managed data layer.

## 2. Business continuity

```text
Phase 03
Legacy VMware VM
  → EC2 rehost
  → PostgreSQL replatform to RDS
  → operational files to S3
  → temporary compute/database migration resources cleaned up
  → AMI + backing snapshot + S3 intentionally retained

Phase 05
Retained AMI
  → temporary recovery EC2
  → extract existing Flask app
  → remove VM assumptions
  → Dockerize
  → ECR
  → ECS Fargate
  → ALB
  → new temporary private RDS PostgreSQL
  → failure / recovery / scaling proof
  → cleanup
```

## 3. Frozen lab architecture

```text
Internet
  ↓
Internet Gateway
  ↓
ALB in two public subnets / two AZs
  ↓
Target Group — target type IP
  ↓
ECS Service / Fargate
  ├── awsvpc
  ├── public subnets
  ├── assignPublicIp = ENABLED
  ├── desiredCount = 1 baseline
  ├── 0.25 vCPU / 512 MiB initial task size
  └── ECS-SG inbound only from ALB-SG
  ↓ TCP/5432
RDS PostgreSQL
  ├── private subnets
  ├── Single-AZ
  ├── db.t4g.micro subject to live preflight
  ├── 20 GB gp3
  ├── not publicly accessible
  └── RDS-SG inbound only from ECS-SG
```

## 4. Supporting services

Required:

- ECR private repository,
- Secrets Manager for DB credentials,
- CloudWatch Logs for application/container logs,
- IAM execution role,
- task role only if the application calls AWS APIs.

Optional only if recovered source proves a requirement:

- existing Phase 03 S3 operational-data bucket.

Not planned:

- NAT Gateway,
- VPC interface endpoints for the short-lived lab,
- EKS,
- Multi-AZ RDS,
- Route 53/custom domain,
- WAF,
- Terraform,
- CI/CD.

## 5. Networking decision

For a short-lived lab, public Fargate ENIs are the deliberate cost/complexity trade-off. Direct ingress to tasks remains blocked by Security Groups.

```text
0.0.0.0/0 → ALB-SG :80
ALB-SG     → ECS-SG :application-port
ECS-SG     → RDS-SG :5432
0.0.0.0/0 → ECS-SG  DENIED
0.0.0.0/0 → RDS-SG  DENIED
```

Production guidance is documented separately: private tasks + controlled egress.

## 6. Container design

During AMI recovery, inventory:

- application source,
- templates/static assets,
- Python version/dependencies,
- systemd unit/launch assumptions,
- local DB configuration,
- filesystem state,
- environment variables,
- secrets,
- hardcoded addresses/paths,
- logging behavior,
- listening port.

Do **not** copy:

- passwords,
- SSH private keys,
- AWS credentials,
- machine certificates,
- shell history,
- database dumps not intentionally required,
- OS-specific service files as runtime dependencies,
- `.env` secrets.

Target container requirements:

- slim supported Python image,
- Gunicorn,
- non-root user,
- pinned dependencies,
- stdout/stderr logging,
- stateless filesystem,
- configuration through environment/secrets,
- `/api/health` for liveness,
- `/api/ready` or functional DB-backed endpoint for dependency readiness.

## 7. RDS design

```text
PostgreSQL
Single-AZ
Private
20 GB gp3
Deletion protection OFF
Backup retention 0 for short-lived lab
Skip final snapshot during cleanup
```

The database is recreated specifically for Phase 05. The retained Phase 03 EBS snapshot backs the AMI; it is not an RDS snapshot and must not be presented as an RDS restore source.

## 8. IAM

### Task Execution Role

Used by ECS/Fargate infrastructure to:

- authenticate/pull from ECR,
- publish logs,
- retrieve the specific Secrets Manager secret used in task definition secret injection.

Use the standard ECS task execution managed policy plus only the additional secret/KMS permissions actually needed.

### Task Role

Used by application code at runtime.

Default: no AWS API permissions.

If the recovered app genuinely reads S3, grant only the required S3 actions on the required prefix.

PostgreSQL network/database authentication does not require generic RDS IAM permissions unless IAM DB authentication is intentionally introduced, which is not planned for this lab.

## 9. Health model

```text
/api/health
  → process/application liveness
  → ALB health check

/api/ready
  → database/dependency readiness
  → functional validation / dependency failure test
```

This prevents a database outage from causing endless container replacement when containers themselves are healthy.

## 10. Validation gates

### Gate A — recovery

- AMI launches.
- app location/runtime identified.
- source extracted without secrets.
- temporary recovery EC2 terminated.

### Gate B — local container

- Docker image builds.
- container starts.
- liveness passes.
- database configuration is external.
- no secrets in image/history/repo.

### Gate C — ECR

- versioned image pushed.
- immutable evidence of repository/tag/digest captured.

### Gate D — ECS/RDS

- initial task starts.
- ECR pull succeeds.
- logs arrive in CloudWatch.
- task reaches RDS on 5432.
- DB-backed API returns expected MADAR data.

### Gate E — ALB

- target healthy.
- user request through ALB succeeds.
- direct task ingress remains blocked by SG design.

### Gate F — HA/self-healing

- desiredCount set to 2.
- both targets healthy.
- one task intentionally stopped.
- service creates replacement.
- desired count returns to 2.

### Gate G — auto scaling

- baseline minimum 1 task.
- target tracking policy enabled.
- controlled CPU load generated.
- scale-out observed.
- post-load scale-in observed.

### Gate H — dependency failure

- temporarily revoke ECS-SG → RDS-SG 5432.
- DB-backed readiness/request fails predictably.
- restore rule.
- functionality recovers.

## 11. Observability

Minimum useful set:

- CloudWatch Logs,
- ECS CPU/memory service metrics,
- ALB target health / RequestCount / TargetResponseTime / HealthyHostCount,
- RDS basic metrics.

No custom dashboard/SNS/alarms unless execution proves they add real value.

## 12. HTTPS decision

Lab: HTTP on ALB for short-lived validation.

Production recommendation: custom domain + ACM certificate + HTTPS listener + HTTP→HTTPS redirect.

No domain will be purchased solely to make the temporary lab look more production-like.

## 13. Implementation order

1. Gate 0 account/cost preflight.
2. Verify retained AMI/snapshot/S3.
3. Launch temporary EC2 from AMI.
4. Locate and inspect Flask source/runtime.
5. Export safe application source.
6. Terminate temporary EC2.
7. Sanitize source and configuration.
8. Create dependency manifest.
9. Create Dockerfile / `.dockerignore`.
10. Build image locally.
11. Test liveness locally.
12. Test DB-config behavior locally.
13. Create ECR repository.
14. Push versioned image.
15. Create Phase 05 VPC.
16. Create two public ALB/ECS subnets.
17. Create private DB subnets.
18. Create/associate route tables and IGW.
19. Create ALB-SG, ECS-SG, RDS-SG.
20. Create DB subnet group.
21. Create RDS PostgreSQL and wait for `available`.
22. Initialize schema/test data through a controlled path.
23. Create Secrets Manager secret.
24. Create ECS execution role and minimum task role.
25. Create CloudWatch log group.
26. Create ECS cluster.
27. Register task definition.
28. Run initial Fargate task.
29. Verify task logs/ECR pull.
30. Verify task→RDS connectivity and DB-backed behavior.
31. Create target group (`ip`).
32. Create ALB/listener.
33. Create ECS service behind ALB.
34. Validate `/api/health` via ALB.
35. Validate DB-backed endpoint.
36. Set desiredCount=2 and validate both targets.
37. Stop one task intentionally and verify replacement.
38. Configure target tracking auto scaling.
39. Generate controlled load and capture scale-out/in.
40. Perform ECS→RDS SG dependency failure/recovery test.
41. Capture logs/metrics/screenshots.
42. Capture Cost Explorer checkpoint.
43. Execute cleanup runbook.
44. Run residual-resource audit.
45. Update project repository final state.
46. Update MADAR master repository.

## 14. Cost controls

Cost clocks begin only when the corresponding resource exists/runs.

```text
RDS       continuous while DB instance runs
ALB       hourly + LCU while provisioned
Fargate   CPU/RAM while tasks run
IPv4      hourly while public addresses are in use
Secrets   storage/API
ECR       image storage
Logs      ingestion/storage
```

Rules:

- create RDS/ALB as late as practical,
- keep desiredCount=1 except HA/load tests,
- delete ALB/RDS immediately after evidence,
- no NAT Gateway,
- run final inventory checks after cleanup.

## 15. Cleanup order

See `runbooks/99-cleanup-runbook.md` for executable checks.

High-level:

1. disable/delete auto scaling policies/targets,
2. scale ECS service to 0,
3. delete ECS service and wait for drain,
4. delete ALB listener/rules,
5. delete ALB,
6. delete target group,
7. deregister task definition revisions if desired,
8. delete ECS cluster,
9. delete RDS with skip-final-snapshot,
10. delete DB subnet group,
11. schedule/force-delete temporary secret according to cleanup decision,
12. delete CloudWatch log group if evidence already captured,
13. delete ECR images/repository unless deliberately retained,
14. delete security groups,
15. delete subnets/custom route tables,
16. detach/delete IGW,
17. delete VPC,
18. verify no Phase 05 residual resources.

## 16. What should remain after closeout

Keep at zero/near-zero portfolio cost:

- source code,
- Dockerfile,
- tests,
- runbooks,
- ADRs,
- screenshots/evidence,
- measured cost closeout.

Do not automatically retain:

- RDS,
- ALB,
- running tasks,
- public IPv4,
- ECR images.

Phase 03 AMI/snapshot/S3 retention will be reconsidered only after Phase 05 proves the application can be rebuilt from source/container artifacts.

## 17. Change-control rule

This plan is frozen for execution. A change is allowed only when one of these is true:

1. live AWS account/service preflight exposes a real restriction,
2. recovered application source exposes a technical dependency not knowable beforehand,
3. measured container performance proves the initial task size insufficient,
4. an AWS service/API constraint invalidates a planned step.

Any change must be documented as an ADR or execution note rather than silently rewriting the story.
