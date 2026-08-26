# 📦 MADAR — Application Modernization with Containers

## Phase 05 of the MADAR Cloud Transformation

> **🟡 STATUS: PREPARED • PREFLIGHT REQUIRED • BUILD NOT STARTED**  
> Legacy Flask workload → Docker → Amazon ECR → Amazon ECS on AWS Fargate → Application Load Balancer → private Amazon RDS PostgreSQL → scaling / self-healing / failure testing → cost-controlled cleanup.

Phase 05 continues the same MADAR workload story from Phase 03. It does **not** invent a new demo application. The retained Phase 03 AMI is the recovery source used to extract the existing Flask application, remove machine-specific assumptions and modernize its runtime into a stateless container architecture.

---

## 🎯 Business problem

Phase 03 successfully migrated the representative legacy workload into AWS and proved database/file migration and application cutover. The application runtime, however, still originated from a traditional VM model.

Phase 05 asks a different engineering question:

> Can MADAR separate the application from the operating system, package it as a portable container, run it on managed compute, preserve the external PostgreSQL data layer, and prove health, scaling and recovery behavior without keeping a VM alive?

---

## 🔗 Continuity from Phase 03

Historical assets intentionally retained after Phase 03:

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

Phase 05 begins with:

```text
Retained Phase 03 AMI
        ↓
Temporary recovery EC2
        ↓
Inspect/extract Flask application
        ↓
Terminate recovery EC2
        ↓
Dockerize application
```

The temporary EC2 is a recovery/extraction step only. It is not the target runtime.

---

## 🏗️ Frozen lab architecture

```text
                         INTERNET
                            │
                            ▼
                     Internet Gateway
                            │
               ┌────────────┴────────────┐
               │                         │
        Public Subnet A           Public Subnet B
               │                         │
               └────────────┬────────────┘
                            │
                       Application
                      Load Balancer
                         HTTP :80
                            │
                    Target Group
                      target = IP
                            │
                    ALB-SG → ECS-SG
                            │
             ┌──────────────┴──────────────┐
             │                             │
      Fargate Task                  Fargate Task
      0.25 vCPU                     temporary during
      512 MiB                       HA/load tests
      public IPv4
             │
             │ TCP 5432
             ▼
       Private DB Subnets
             │
             ▼
      Amazon RDS PostgreSQL
         db.t4g.micro
          Single-AZ
           private
          20 GB gp3
```

Supporting services only where they have a real job:

```text
Amazon ECR          → private Docker image registry
Secrets Manager     → database credentials
CloudWatch Logs     → container stdout/stderr
Existing S3         → only if the recovered application genuinely needs operational files
```

---

## 🧠 Lab vs production decision

For this short-lived portfolio lab, Fargate tasks will use public subnets with `assignPublicIp=ENABLED`. The task security group will allow **no direct Internet ingress**; application ingress is allowed only from the ALB security group.

This avoids NAT Gateway and multiple paid VPC interface endpoints during a short test window.

Production recommendation remains different:

```text
LAB
Public Fargate tasks + strict SGs

PRODUCTION
Private Fargate tasks + controlled egress
(NAT and/or VPC endpoints according to workload requirements)
```

The difference is intentional and documented as an Architecture Decision Record.

---

## 🐘 PostgreSQL design

The Phase 03 RDS instance was deleted during cleanup. Phase 05 therefore creates a **new temporary RDS PostgreSQL instance** for the modernization test.

Lab configuration:

```text
Engine                PostgreSQL
Class                 db.t4g.micro (subject to account preflight)
Deployment            Single-AZ
Storage               20 GB gp3
Public accessibility  No
Inbound               TCP 5432 from ECS-SG only
Deletion protection   Off
Backup retention      0 for temporary lab
Final snapshot         Skip during final cleanup
```

The database must remain external to the application containers so multiple tasks can use one managed data layer.

---

## 🐳 Containerization rules

The recovered Flask application will be modernized without an unnecessary rewrite.

Required changes:

- Docker image built from a slim Python base image.
- Gunicorn instead of Flask's development server.
- non-root container user.
- pinned Python dependencies.
- `.dockerignore` excluding credentials, Git metadata, caches and local artifacts.
- database host/name/user/password externalized from the image.
- application logs written to stdout/stderr.
- no durable application state written to the container filesystem.
- machine-specific paths, systemd assumptions, localhost database settings and hardcoded IPs removed.

Health model:

```text
/api/health  → application process/liveness
/api/ready   → application dependency readiness / database path
```

ALB health checks should use the liveness endpoint so a database outage does not create a container replacement loop for a dependency problem.

---

## 🔐 Security model

```text
Internet
   ↓
ALB-SG
   ↓ application port only
ECS-SG
   ↓ PostgreSQL 5432 only
RDS-SG
```

Controls:

- RDS is private.
- no database secret in GitHub.
- no secret inside the Docker image.
- Secrets Manager secret scoped to Phase 05.
- least-privilege ECS execution role.
- task role receives AWS API permissions only if the application actually needs them.
- no SSH to Fargate.
- non-root container.
- ECR and RDS encryption at rest where supported/default.
- HTTP is accepted only for the short-lived lab; production TLS termination at ALB is documented as a future production requirement.

---

## 🧪 Acceptance tests

Phase 05 is not complete merely because resources are `RUNNING`.

```text
1. ALB → Flask health                         PASS required
2. Flask → PostgreSQL database-backed API     PASS required
3. Two healthy Fargate targets                PASS required
4. Stop one task → ECS replacement            PASS required
5. Controlled CPU load → scale out/in          PASS required
6. Break ECS→RDS SG rule → fail → restore      PASS required
7. Cost / residual-resource audit              PASS required
```

Expected self-healing story:

```text
Task A ✅     Task B ✅
      ↓ stop Task A intentionally
ALB removes unhealthy target
      ↓
ECS detects desired-count mismatch
      ↓
replacement task starts
      ↓
health checks pass
      ↓
desired count restored ✅
```

---

## 💰 Cost guardrail

No cost-bearing infrastructure will be created until **Gate 0** passes.

Primary cost clocks:

```text
1. RDS instance
2. Application Load Balancer / LCU
3. running Fargate tasks
4. public IPv4 addresses
5. Secrets Manager
6. ECR storage / CloudWatch ingestion
```

NAT Gateway is intentionally excluded from the lab design.

AWS promotional credits and Free Plan service availability are account-specific. Public documentation cannot prove the current balance or every account-level restriction, so the live preflight is mandatory.

See [`checklists/00-account-cost-preflight.md`](checklists/00-account-cost-preflight.md).

---

## 🚦 Execution gates

```text
GATE 0  Account / credits / retained-asset preflight
   ↓
GATE 1  Recover and inspect legacy application
   ↓
GATE 2  Local Docker validation
   ↓
GATE 3  ECR image publication
   ↓
GATE 4  VPC / RDS / IAM / ECS foundation
   ↓
GATE 5  ALB + functional validation
   ↓
GATE 6  HA / failure / scaling tests
   ↓
GATE 7  evidence + cost closeout
   ↓
GATE 8  destructive cleanup + residual audit
```

Full sequence: [`docs/PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md`](docs/PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md)

---

## 📂 Repository structure

```text
.
├── README.md
├── CURRENT-STATE.md
├── REPOSITORY-SCOPE.md
├── .gitignore
├── checklists/
│   └── 00-account-cost-preflight.md
├── decisions/
│   ├── ADR-001-public-fargate-for-short-lived-lab.md
│   └── ADR-002-http-lab-production-tls.md
├── docs/
│   └── PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md
├── evidence/
│   └── README.md
├── runbooks/
│   ├── 00-execution-runbook.md
│   └── 99-cleanup-runbook.md
├── src/                       # populated after AMI extraction
├── tests/                     # populated with actual validation commands
└── Dockerfile                 # created after source assessment
```

The repository is deliberately prepared before deployment so design, cost, acceptance and cleanup are known before the first AWS resource is created.

---

## 🏁 Current state

```text
Architecture reviewed          ✅
Cost traps identified          ✅
Lab/production differences     ✅
Cleanup designed               ✅
Repository prepared            ✅
AWS live account preflight     ⏳ NEXT
Infrastructure deployment      ⏸ NOT STARTED
```

Master transformation record: `EngMohammedBashir/MADAR-cloud-transformation`.
