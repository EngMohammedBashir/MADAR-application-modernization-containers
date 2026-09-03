# 🎯 Repository Scope — MADAR Phase 05

> 🟢 **Scope delivered and closed.** This repository is both a portfolio artifact and a reproducible engineering record.

## 🚀 Delivered in Scope

- ✅ recovered the legacy Flask application from the retained Phase 03 AMI;
- ✅ removed VM-specific DB assumptions and containerized with Docker/Gunicorn/non-root execution;
- ✅ published versioned images to ECR during the lab;
- ✅ deployed on ECS Fargate behind an ALB;
- ✅ used private RDS PostgreSQL and Secrets Manager;
- ✅ recovered the authoritative DB dump from the retained snapshot and restored it through a one-off Fargate task;
- ✅ enforced `Internet → ALB → ECS → RDS` SG boundaries;
- ✅ captured CloudWatch/ECS/ALB/RDS observability;
- ✅ proved task self-healing and automatic scale-out;
- ✅ injected/recovered an RDS dependency failure;
- ✅ captured cost evidence;
- ✅ destroyed temporary Phase 05 infrastructure and proved residual cleanup.

## 🚫 Deliberately Not Implemented

| Production capability | Why it was not Phase 05 |
|---|---|
| ☸️ EKS/Kubernetes | Runtime modernization was proven with ECS/Fargate; Kubernetes belongs in a dedicated phase. |
| 🐘 Multi-AZ encrypted RDS + production backups | Short-lived synthetic-data lab and strict cost control. |
| 💸 NAT Gateway | Avoided recurring NAT cost for a short validation window. |
| 🔒 Private Fargate + VPC endpoints | More production-like, but adds endpoint/network cost and complexity outside the learning objective. |
| 🔏 HTTPS/ACM + Route 53/domain | No owned production domain was required for the disposable lab. |
| 🧱 WAF | Edge hardening reserved for security-focused work. |
| 🏗️ Terraform/IaC | Intentionally deferred so this phase demonstrated the underlying services and runtime behavior directly. |
| 🔁 CI/CD | Intentionally deferred to a delivery-automation phase. |
| 👤 Least-privilege PostgreSQL app user | Lab used the restored/admin path; production should create a restricted app role. |
| 📣 Production alerting | Metrics/logs were proven; full alarm routing/on-call workflow was outside scope. |

## 🧪 Lab vs Production

```text
🧪 PHASE 05 LAB                       🏭 PRODUCTION EVOLUTION
Public-IP Fargate + strict SG         Private tasks + controlled egress
HTTP :80                              HTTPS :443 + ACM + redirect
Single-AZ temporary RDS               Multi-AZ + encryption + backups
Manual AWS CLI build                  IaC + reviewed change workflow
Manual image/deploy steps             CI/CD + approvals + rollback
Master-user DB path                   Least-privilege app DB role
Evidence-driven testing               Automated tests + alerts/SLOs
```

## 🛡️ Claim Boundaries

- Automatic **scale-out** is proven; separate automatic scale-in is not claimed.
- During DB failure injection, the externally observed readiness response was **502 through ALB**; documentation preserves that observation.
- Cost screenshots are checkpoints because Cost Explorer can lag.
- Cleaned-up AWS resources are historical architecture, not currently running infrastructure.

## 📚 Source-of-Truth Hierarchy

1. 📍 `CURRENT-STATE.md` — final state.
2. 🚀 `README.md` — reviewer-facing story.
3. 📋 `docs/PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md` — design + executed status.
4. 🧰 `runbooks/` — independent rebuild/cleanup instructions.
5. 📸 `evidence/` — proof.
6. 🧠 `decisions/` — architectural reasoning.
7. ✅ `checklists/` — gate completion.

## 🔗 Continuity Rule

Phase 05 cleanup must never be confused with deletion of the retained Phase 03 AMI, snapshot and S3 bucket. Those assets intentionally bridge MADAR into later phases.