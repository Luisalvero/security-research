# Zennvoi — Security and Architecture Assessment

| | |
|---|---|
| Status | Complete (v1.0, 2026-05-01) |
| Live system | [https://www.zennvoi.com](https://www.zennvoi.com) |
| Assessment type | Self-conducted security and architecture review |

A self-conducted security and architecture review of a production SaaS I
build and operate. Eight findings, ranked by severity, with prioritized
remediation. Three-phase methodology: static source review, dynamic
verification against a running instance, and direct database-layer audit.

## Contents

- [report.md](./report.md) — full assessment document
- [threat-model.md](./threat-model.md) — STRIDE matrix with per-cell rationale
- [diagrams/](./diagrams/) — system architecture, authentication flow, data model

## Skills demonstrated

Backend architecture, authentication and session design, Postgres
Row-Level Security and multi-tenant isolation, Stripe and Stripe Connect
integration security, threat modeling (STRIDE), attack-surface
enumeration, defense in depth at the database layer.

## Disclosure posture

The assessment deliberately omits exact numeric thresholds, vendor names
of managed-service providers (with the exception of Stripe, where the
integration is part of the analysis), specific column names where they
would aid fingerprinting, and any current finding for which detailed
exploitation guidance would meaningfully aid an attacker prior to
remediation. See `report.md` Section 8 for the full disclosure note.
