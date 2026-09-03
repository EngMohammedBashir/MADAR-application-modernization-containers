# 🎯 Repository Scope — MADAR Phase 05

This repository is the implementation and evidence record for **MADAR Phase 05 — Application Modernization with Containers**.

## 🚀 In Scope

- 🔎 recover the existing MADAR Flask application from the retained Phase 03 AMI,
- 🧹 remove VM-specific runtime assumptions,
- 🐳 containerize with Docker + Gunicorn + non-root execution,
- 📦 publish versioned application/restore images to Amazon ECR,
- 🚀 run the application using Amazon ECS on AWS Fargate,
- ⚖️ place an Application Load Balancer in front of the ECS service,
- 🐘 use a private Amazon RDS PostgreSQL data layer,
- 🗃️ recover the legacy PostgreSQL dump from retained Phase 03 artifacts when required for continuity,
- ♻️ restore that dump through a controlled one-off Fargate task,
- 🔐 externalize database credentials with Secrets Manager,
- 🛡️ enforce `Internet → ALB → ECS → RDS` security-group boundaries,
- 📊 capture useful CloudWatch/ECS/ALB/RDS observability,
- ♻️ validate task replacement/self-healing,
- 📈 validate controlled target-tracking scale-out,
- 🔌 validate database dependency failure and recovery,
- 💰 capture cost evidence,
- 🧹 remove temporary AWS resources and prove residual cleanup.

## 🚫 Explicitly Out of Scope

- ☸️ EKS / Kubernetes,
- 🐘 production-grade Multi-AZ RDS,
- 💸 NAT Gateway for the short-lived lab,
- 🔒 full private Fargate + VPC endpoint production networking,
- 🧱 WAF,
- 🌐 Route 53/custom domain purchase,
- 🔏 mandatory HTTPS for this temporary lab,
- 🏗️ Terraform/IaC,
- 🔁 CI/CD/GitHub Actions deployment automation,
- 🧩 application rewrite or microservice decomposition.

These exclusions are deliberate: **Phase 05 demonstrates runtime modernization and container operations.** Delivery automation and broader platform engineering belong in later dedicated phases.

## 🧠 Lab vs Production

```text
🧪 LAB
Public Fargate ENIs + strict SG ingress
HTTP ALB listener
Single-AZ temporary RDS
No NAT Gateway
Temporary restore container/task

🏭 PRODUCTION EXTENSION
Private Fargate tasks
Controlled egress / VPC endpoints as appropriate
HTTPS + ACM
Encrypted / highly available database design
Least-privilege application DB user
IaC + CI/CD
```

## 📚 Source-of-Truth Hierarchy

1. 📍 `CURRENT-STATE.md` — what is actually true now.
2. 🚀 `README.md` — portfolio-facing architecture/story/progress.
3. 📋 `docs/PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md` — approved architecture plus execution status.
4. 🧰 `runbooks/` — operational procedures.
5. 📸 `evidence/` — proof supporting project claims.
6. 🧠 `decisions/` — architecture trade-offs.

## 🛡️ Claim Rule

No document may claim a validation gate is complete until execution actually occurred. Planned production improvements remain recommendations, not deployed controls. Observed results are recorded exactly: automatic scale-out is proven; separate automatic scale-in is not claimed, and the RDS failure test records the observed ALB-facing `502` rather than rewriting it to the application's intended internal `503`.