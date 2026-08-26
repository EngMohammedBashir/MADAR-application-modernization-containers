# ADR-002 — HTTP for Temporary Lab, TLS for Production

**Status:** Accepted for Phase 05 lab  
**Decision date:** 2026-08-27

## Context

A production Internet-facing application should normally terminate TLS at the load balancer using a validated certificate and domain. The Phase 05 environment is a short-lived portfolio lab that will be destroyed after evidence capture.

## Decision

Use an ALB HTTP listener on port 80 for the temporary lab.

Do not purchase/configure Route 53 and a custom domain solely for this test.

## Production requirement

A production version would use:

```text
Route 53 / owned DNS
      ↓
ACM certificate
      ↓
ALB HTTPS :443
      ↓
HTTP :80 → redirect to HTTPS
```

## Why

- application modernization, not certificate lifecycle, is the Phase 05 learning objective,
- the lab handles synthetic data only,
- resources are short lived,
- documentation explicitly records the security difference rather than pretending HTTP is a production recommendation.

## Consequence

Screenshots or evidence must not imply the HTTP endpoint is production-ready security posture.
