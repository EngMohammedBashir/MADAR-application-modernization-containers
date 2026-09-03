# 📸 Phase 05 — Evidence Index

> 🟢 **Evidence set COMPLETE.** Screenshots are included only when they support an engineering claim. No passwords, keys, tokens, private keys or secret values belong here.

## 🔎 Recovery
| Evidence | Claim |
|---|---|
| `recovery-ec2-legacy-app-discovered.png` | Legacy Flask workload recovered from retained AMI |
| `recovery-resources-cleaned-up.png` | Temporary recovery EC2/EBS/SG removed |
| `legacy-database-dump-recovered.png` | Authoritative PostgreSQL dump recovered from retained snapshot |

## 🐳 Containers / ECR
| Evidence | Claim |
|---|---|
| `docker-build-success.png` | Application image built |
| `docker-container-health-success.png` | Local container health/non-root behavior validated |
| `ecr-image-push-success.png` | Application image pushed |
| `ecr-console-v1-verified.png` | `v1` present in ECR during lab |
| `MADAR-P05-Restore-Image-QA.png` | Restore image tooling validated |
| `MADAR-P05-ECR-Restore-Repository.png` | Restore repository created |
| `MADAR-P05-Restore-Image-Pushed.png` | Restore `v1` published |

## 🌐 Networking / Security
| Evidence | Claim |
|---|---|
| `public-network-routing-validated.png` | Public routing/IGW path validated |
| `security-group-chain-validated.png` | Internet→ALB→ECS→RDS SG chain validated |

## 🚀 Runtime / Database / ALB
| Evidence | Claim |
|---|---|
| `ecs-fargate-task-running.png` | Fargate runtime reached RUNNING |
| `database-api-validation.png` | DB-backed application path worked |
| `alb-healthy-fargate-target.png` | ALB target healthy |
| `madar-dashboard-on-fargate.png` | Dashboard rendered through ALB/Fargate |

Restore task exited `0` and logged `RESTORE COMPLETED SUCCESSFULLY`; verification task exited `0`. Exact row counts are not claimed without separate evidence.

## ♻️ Reliability
| Evidence | Claim |
|---|---|
| `ecs-two-healthy-fargate-targets.png` | Two healthy targets established |
| `ecs-task-self-healing.png` | Intentional task failure/replacement activity |
| `ecs-self-healing-recovered.png` | Service recovered desired healthy capacity |
| `ecs-auto-scaling-triggered.png` | Target tracking automatically scaled desired count `1→2` |

⚠️ Automatic scale-out is proven. Separate scale-in is not claimed.

## 🔌 Dependency Failure
| Evidence | Claim |
|---|---|
| `rds-dependency-failure.png` | With `5432` revoked: health `200`, ready `502` through ALB |
| `rds-dependency-recovery.png` | After rule restore: health/ready `200` |

The externally observed `502` is preserved exactly.

## 📊 Observability
| Evidence | Claim |
|---|---|
| `observability-infrastructure-health.png` | ECS/ALB/RDS/log-stream state |
| `observability-cloudwatch-metrics.png` | ECS CPU/memory, ALB requests, RDS connections |

## 💰 Cost & Closeout
| Evidence | Claim |
|---|---|
| `phase05-cost-checkpoint.png` | Pre-cleanup Cost Explorer checkpoint |
| `Cost-Closeout-Evidence.png` | Final available cost closeout checkpoint after teardown |
| `phase05-residual-audit.png` | Phase 05 runtime resources reported deleted while retained assets remained available |

> 💰 Cost Explorer can lag; cost images are billing checkpoints, not promises of final settled zero cost.

## 🧠 Reviewer Reading Order

For the fastest proof trail: `madar-dashboard-on-fargate.png` → `ecs-task-self-healing.png` → `ecs-self-healing-recovered.png` → `ecs-auto-scaling-triggered.png` → `rds-dependency-failure.png` → `rds-dependency-recovery.png` → `observability-cloudwatch-metrics.png` → `phase05-residual-audit.png` → `Cost-Closeout-Evidence.png`.

## 🛡️ Evidence Rule

A screenshot belongs here only when it proves **architecture • security • functionality • recovery • failure behavior • scaling • observability • cost • cleanup**. Decorative screenshots and duplicate proof should be omitted.