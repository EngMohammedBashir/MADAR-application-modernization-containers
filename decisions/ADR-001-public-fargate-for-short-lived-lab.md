# 🧠 ADR-001 — Public Fargate for the Short-Lived Lab

**Status:** ✅ Accepted → Executed → Cleaned up  
**Decision date:** 2026-08-27 · **Outcome recorded:** 2026-09-03

## 🎯 Context

Fargate needed outbound access to ECR, Secrets Manager and CloudWatch. I compared: private subnets + NAT, private subnets + VPC endpoints, or public subnets + public IPv4 with strict SG ingress.

## ✅ Decision

For this short-lived cost-controlled lab, I used public Fargate ENIs with `assignPublicIp=ENABLED` while allowing application ingress only from `ALB-SG` to `ECS-SG:8080`.

```text
🌍 Internet → ⚖️ ALB → 🛡️ ECS-SG:8080
🌍 Internet ─X→ ECS-SG:8080
```

## 💡 Why I Chose It

- 💰 avoided NAT Gateway hourly/data-processing cost;
- 🧩 avoided adding multiple interface endpoints for a short validation window;
- 🎯 kept focus on container modernization, ALB, Fargate, `awsvpc` and SG-to-SG controls;
- 🧪 synthetic-data lab was destroyed after testing.

## 🧪 Observed Outcome

A manual Fargate task received a public IPv4, but direct Internet access to `:8080` timed out because ECS-SG allowed only ALB-SG. I treated that timeout as a successful security-boundary test, not an application failure. End-to-end access through the ALB worked.

## ⚖️ Consequences

- active tasks could incur public IPv4 cost;
- task ENIs were Internet-routable at the network layer;
- SGs carried more responsibility for ingress isolation;
- this design is **not** my production recommendation.

## 🏭 Production Recommendation

Use private Fargate tasks with controlled egress, selecting NAT and/or VPC endpoints based on traffic, cost and security requirements.

> 🧠 **What I learned:** a cheaper lab architecture can still be intentional and defensible when its risk boundary is explicit, tested and removed after use.