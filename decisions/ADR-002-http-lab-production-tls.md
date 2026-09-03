# 🔐 ADR-002 — HTTP for Temporary Lab, TLS for Production

**Status:** ✅ Accepted → Executed → Endpoint destroyed  
**Decision date:** 2026-08-27 · **Outcome recorded:** 2026-09-03

## 🎯 Context

A real Internet-facing production application should terminate TLS with a validated certificate and owned DNS. Phase 05 was a short-lived synthetic-data lab whose goal was container runtime modernization and operational testing.

## ✅ Decision

I used an ALB listener on HTTP `:80` only for temporary validation. I did not purchase/configure a domain solely to make the disposable lab look more production-like.

## 🧪 Outcome

The HTTP endpoint was sufficient to validate ALB→Fargate routing, health/readiness, dashboard behavior, task self-healing, scaling and DB dependency failure/recovery. After evidence capture, the ALB and Phase 05 network were deleted.

## 🏭 Production Requirement

```text
🌐 Owned DNS / Route 53
        ↓
🔏 ACM certificate
        ↓
⚖️ ALB HTTPS :443
        ↑
↪️ HTTP :80 → redirect to HTTPS
```

Production should also add appropriate security headers, edge controls/WAF where justified, monitoring and certificate lifecycle management.

## ⚠️ Claim Boundary

No Phase 05 screenshot or document should describe HTTP `:80` as production-ready security. It was a bounded lab decision, not a recommended steady state.

> 🧠 **What I learned:** a portfolio project is stronger when it states what was deliberately *not* built and explains the production replacement.