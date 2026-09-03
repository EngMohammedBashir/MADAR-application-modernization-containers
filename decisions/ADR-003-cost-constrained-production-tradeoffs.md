# 💰 ADR-003 — Cost-Constrained Lab vs Production Controls

**Status:** ✅ Accepted / retrospective formalization  
**Recorded:** 2026-09-03

## 🎯 Context

Phase 05 ran under strict cost/account constraints and without an AWS account upgrade. The goal was to prove container modernization and operational behavior without leaving expensive infrastructure running.

## ✅ Decision

Prefer the smallest architecture that still proves the intended engineering claims, and explicitly document production controls that were deferred rather than pretending they were implemented.

## 🧪 Implemented Lab Choices

```text
Fargate desired baseline 1
Auto Scaling max 2
Single-AZ db.t4g.micro RDS
No NAT Gateway
No EKS
No WAF
No custom domain
HTTP-only temporary ALB
Short validation window
Aggressive post-test cleanup
```

## 🏭 Deferred Production Controls

Private Fargate networking, NAT/VPC endpoints, HTTPS/ACM/DNS, Multi-AZ encrypted RDS/backups, least-privilege DB user, WAF, IaC, CI/CD, alert routing and richer HA/DR testing.

## 💡 Why

A portfolio lab should demonstrate judgment, not spend. Adding every production service would have increased cost and distracted from the Phase 05 learning objective. The stronger engineering choice was to state the limitations, test the selected architecture deeply, preserve evidence and tear it down cleanly.

## ⚖️ Consequence

This repository is evidence of **runtime modernization and operational reasoning**, not a claim that the temporary lab topology is a finished production platform.

> 🧠 **My takeaway:** constraints are part of architecture. I should be able to explain what I omitted, why I omitted it, the risk introduced, and what I would deploy in production.