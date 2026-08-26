# ADR-001 — Public Fargate for the Short-Lived Lab

**Status:** Accepted for Phase 05 lab  
**Decision date:** 2026-08-27

## Context

Fargate tasks must pull images from ECR and may need access to CloudWatch Logs and Secrets Manager during startup/runtime.

Three patterns were considered:

1. private subnets + NAT Gateway,
2. private subnets + VPC endpoints,
3. public subnets + `assignPublicIp=ENABLED` with strict Security Groups.

## Decision

Use option 3 for the short-lived Phase 05 lab.

```text
Internet → ALB-SG → ECS-SG

Direct Internet → ECS-SG = denied
```

Tasks can initiate outbound connections through the Internet Gateway while direct application ingress remains restricted to the ALB security group.

## Why

- avoids NAT Gateway hourly/data-processing cost,
- avoids multiple paid interface endpoints for a lab measured in hours,
- reduces setup complexity,
- still demonstrates ALB, ECS service networking, `awsvpc`, SG-to-SG rules and target type `ip`,
- keeps the portfolio focus on application modernization rather than private-service networking plumbing.

## Consequences

- each active Fargate task may incur public IPv4 cost,
- tasks have Internet-routable ENIs, even though Security Groups block direct inbound access,
- this is **not** documented as the preferred production network design.

## Production recommendation

Use private Fargate subnets with controlled egress based on actual requirements, using NAT and/or VPC endpoints where justified.
