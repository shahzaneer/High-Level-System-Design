# Threat Modeling

Threat Modeling is the systematic process of identifying and prioritizing potential threats to a system. It answers: "What are we building? What can go wrong? What are we going to do about it?" It's a design-time activity, not a penetration test (which is runtime). The most widely used methodology is STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege), developed at Microsoft.

Threat modeling should happen during architecture design, not after implementation. Fixing a design flaw found during threat modeling costs engineer-hours. Fixing the same flaw found during a penetration test in production costs engineer-weeks plus potential breach impact.

## STRIDE Methodology

| Threat | Definition | Example | Mitigation |
|--------|-----------|---------|------------|
| **S**poofing | Impersonating someone/something | Attacker uses stolen JWT | Strong authentication, MFA |
| **T**ampering | Modifying data or code | Modifying order total in transit | TLS, digital signatures |
| **R**epudiation | Denying an action | "I didn't place that order" | Audit logs, digital signatures |
| **I**nformation Disclosure | Exposing information | API returns other users' data | Authorization checks, encryption |
| **D**enial of Service | Overwhelming resources | API DDoS attack | Rate limiting, WAF, CDN |
| **E**levation of Privilege | Gaining unauthorized access | Normal user becomes admin | Least privilege, RBAC |

## Practical Threat Modeling

```
DATA FLOW DIAGRAM → THREATS → MITIGATIONS

Step 1: Draw the system
  [User Browser] → [CloudFront] → [ALB] → [Order Service] → [PostgreSQL]
                                   ↓
                              [Payment Service] → [Stripe API]

Step 2: Identify trust boundaries
  Between: Browser and CloudFront (public internet)
  Between: ALB and Order Service (internal AWS)
  Between: Order Service and Payment Service (internal)
  Between: Payment Service and Stripe API (external)

Step 3: Apply STRIDE at each boundary
  Browser→CloudFront: Tampering (MITM) → Mitigation: TLS, HSTS
  Order Service→PostgreSQL: Information Disclosure (SQL injection) → Mitigation: parameterized queries
  Payment Service→Stripe: Spoofing (fake Stripe) → Mitigation: TLS, API key verification
```

## Threat Modeling Table

```markdown
| # | Threat | Component | STRIDE | Impact | Likelihood | Risk | Mitigation |
|---|--------|-----------|--------|--------|------------|------|------------|
| 1 | SQL injection | Order API | Tampering | High | Medium | High | Parameterized queries, WAF |
| 2 | JWT token replay | Auth Service | Spoofing | Critical | Low | Medium | Short token expiry, nonce |
| 3 | Unencrypted DB backups | Backup System | Information Disclosure | Critical | Low | Medium | Encrypt backups, restrict access |
| 4 | Mass assignment | User API | Elevation of Privilege | High | Medium | High | Explicit allowlist of updatable fields |
| 5 | DDoS on checkout | Order API | Denial of Service | High | Medium | High | Rate limiting, Shield, WAF |
```

## When to Threat Model

```
DO threat model:
  ✓ New system architecture design
  ✓ Major feature that changes trust boundaries
  ✓ Integration with third-party service
  ✓ Handling of new sensitive data type (PII, PCI, PHI)
  ✓ After a security incident (what else is vulnerable?)

DON'T threat model (alone):
  ✗ Instead of penetration testing (complementary, not replacement)
  ✗ Instead of code review (different layers of defense)
  ✗ Once and forget (threat landscape evolves)
```

## Tools

- **Microsoft Threat Modeling Tool**: Free, STRIDE-based, diagram-driven
- **OWASP Threat Dragon**: Open-source, web-based, STRIDE
- **Threatspec**: Threat modeling as code (annotations in source)
- **IriusRisk**: Enterprise, integrates with Jira, CI/CD

## Summary

Threat modeling is the cheapest and most effective security activity an architect can perform. It finds design flaws before a single line of code is written. The STRIDE framework provides a systematic way to identify threats; data flow diagrams reveal trust boundaries where threats concentrate. Threat modeling should be a standard part of architecture review, not an optional security activity.
