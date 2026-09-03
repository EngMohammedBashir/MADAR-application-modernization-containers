# 🔐 ADR-002 — HTTP for Temporary Lab, TLS for Production

**Status:** ✅ Accepted for Phase 05 lab  
**Decision date:** 2026-08-27

## 🎯 Context

A production Internet-facing application should normally terminate TLS at the load balancer using a validated certificate and owned domain. Phase 05 is a short-lived portfolio lab that uses synthetic data and will be destroyed after evidence capture.

## ✅ Decision

Use an ALB HTTP listener on port `80` for temporary validation.

🚫 Do not purchase/configure Route 53 and a custom domain solely to make the temporary lab look more production-like.

## 🏭 Production Requirement

A production version would use:

```text
🌐 Route 53 / owned DNS
        ↓
🔏 ACM certificate
        ↓
⚖️ ALB HTTPS :443
        ↓
↪️ HTTP :80 → redirect to HTTPS
```

## 💡 Why

- 🐳 application modernization is the Phase 05 learning objective,
- 🧪 the lab handles synthetic data only,
- ⏱️ resources are intentionally short lived,
- 📚 documentation explicitly records the security difference rather than presenting HTTP as production-ready.

## ⚠️ Consequence

Screenshots/evidence must not imply the temporary HTTP endpoint represents a production security posture.

> 🧠 **Portfolio takeaway:** security trade-offs are acceptable only when they are intentional, bounded, documented and paired with the correct production recommendation.
