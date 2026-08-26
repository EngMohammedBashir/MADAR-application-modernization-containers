# Phase 05 — Current State

**Status:** 🟡 PREPARED / PREFLIGHT REQUIRED / BUILD NOT STARTED  
**Region target:** `us-east-1`  
**Prepared:** 2026-08-27

## Executive state

Phase 05 has been architecturally reviewed before deployment. No new Phase 05 AWS infrastructure is claimed as implemented yet.

```text
Architecture                 FROZEN FOR LAB
Account/cost preflight       REQUIRED NEXT
Legacy AMI recovery          NOT STARTED
Containerization             NOT STARTED
ECR                          NOT STARTED
ECS/Fargate                  NOT STARTED
ALB                          NOT STARTED
RDS PostgreSQL               NOT STARTED
Failure/scaling tests        NOT STARTED
Cleanup                      PRE-DESIGNED
```

## Continuity assets from Phase 03

Expected retained assets to verify during Gate 0:

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

These identifiers are historical records until live AWS inventory confirms they still exist.

## Frozen lab architecture

```text
Internet
   ↓
Internet Gateway
   ↓
ALB across 2 public subnets
   ↓ target type = ip
ECS Fargate tasks
   ├── public subnet
   ├── assignPublicIp = ENABLED
   ├── initial 0.25 vCPU / 512 MiB
   └── inbound only from ALB-SG
   ↓ TCP 5432
Private RDS PostgreSQL
   ├── Single-AZ
   ├── db.t4g.micro subject to live availability/account check
   └── 20 GB gp3
```

NAT Gateway, Multi-AZ RDS, Route 53/custom domain, EKS, Terraform and CI/CD are intentionally outside the current lab scope.

## Next action

Run [`checklists/00-account-cost-preflight.md`](checklists/00-account-cost-preflight.md) before creating **any** Phase 05 cost-bearing resource.

If Gate 0 passes, proceed to legacy application recovery from the retained AMI.
