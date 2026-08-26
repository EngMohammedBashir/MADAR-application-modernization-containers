# Repository Scope

This repository contains the implementation record for **MADAR Phase 05 — Application Modernization with Containers**.

## In scope

- recover the existing MADAR Flask application from the retained Phase 03 AMI,
- remove VM-specific runtime assumptions,
- containerize with Docker and Gunicorn,
- publish the image to Amazon ECR,
- run the application using Amazon ECS on AWS Fargate,
- place an Application Load Balancer in front of the ECS service,
- use a private Amazon RDS PostgreSQL database,
- externalize credentials,
- implement minimum useful logging/metrics,
- validate load balancing, task replacement, dependency failure and auto scaling,
- capture evidence and cost,
- remove temporary AWS resources and prove cleanup.

## Explicitly out of scope

- EKS / Kubernetes,
- production-grade Multi-AZ RDS,
- NAT Gateway for the short-lived lab,
- full private Fargate + VPC endpoint production networking,
- WAF,
- Route 53/custom domain purchase,
- mandatory HTTPS for the temporary lab,
- Terraform/IaC,
- CI/CD/GitHub Actions deployment automation,
- application rewrite or microservice decomposition.

These exclusions are deliberate. Phase 05 focuses on **runtime modernization and container operations**. Delivery automation and broader platform engineering can be addressed by later MADAR phases.

## Source-of-truth hierarchy

1. `CURRENT-STATE.md` — what is actually true now.
2. `docs/PHASE-05-FROZEN-IMPLEMENTATION-PLAN.md` — approved plan before deployment.
3. `runbooks/` — commands/procedures used during execution.
4. `evidence/` — proof supporting final claims.
5. `decisions/` — why key architecture trade-offs were made.

No document may claim a resource or validation step is complete until execution evidence exists.
