# Zennvoi — Threat Model (STRIDE)

This document is a companion to [`report.md`](./report.md). It records a
STRIDE matrix per component, with one short paragraph per non-empty cell.

## Components analyzed

1. Public web (unauthenticated routes — landing, authentication)
2. Authenticated application (dashboard, document and client CRUD)
3. API routes (privileged operations)
4. Database (Postgres with Row-Level Security)
5. Stripe webhook ingress
6. Email send pipeline
7. PDF render pipeline
8. Scheduled jobs (cron)

## Matrix

Cell values: **M** = mitigated, **P** = partial mitigation, **O** = open,
**N/A** = not credible at this layer.

| Component | Spoofing | Tampering | Repudiation | Info Disclosure | DoS | Elevation |
|---|---|---|---|---|---|---|
| Public web | M | M | N/A | M | P | N/A |
| Authenticated application | M | M | P | M | P | M |
| API routes | M | M | P | M | P | M |
| Database | N/A | M | P | M | O | M |
| Stripe webhook | M | M | M | N/A | O | M |
| Email pipeline | M | M | P | M | P | N/A |
| PDF render | N/A | M | N/A | M | M | M |
| Scheduled jobs | M | O | P | N/A | N/A | M |

## Per-cell notes

### Public web — Spoofing (M)
The managed authentication provider handles password storage, session token
issuance, and brute-force throttling. Disposable email rejection at signup
is enforced client-side; a server-side equivalent is on the roadmap (see
ZNV-2026-006).

### Public web — Tampering (M)
HTTPS only, HSTS enforced at one year with `includeSubDomains`. Strong
response-header baseline prevents UI-layer tampering: `X-Frame-Options:
DENY`, `X-Content-Type-Options: nosniff`, `Permissions-Policy` denying
camera, microphone, and geolocation.

### Public web — Information Disclosure (M)
The authentication provider returns generic "invalid credentials" responses
without distinguishing between unknown user and incorrect password. Source
maps are not exposed in production builds.

### Public web — Denial of Service (P)
Authentication-layer rate limits are applied at the provider's level.
Application-layer rate limits use process-local state and are inaccurate
under horizontal scaling on the serverless platform; see ZNV-2026-001.

### Authenticated application — Spoofing (M)
Sessions are HTTP-only secure cookies set by the authentication provider.
Same-origin cookie defaults plus exact-origin CSRF validation on all
mutating endpoints prevent session-riding from third-party origins.

### Authenticated application — Tampering (M)
All mutating endpoints verify the user's session server-side before
proceeding. Row-Level Security scopes every read and write to
`auth.uid() = user_id`. The `protect_profile_update` trigger blocks
modification of `role` and `plan` columns regardless of the access path.

### Authenticated application — Repudiation (P)
Stripe and the authentication provider maintain their own audit trails.
Application-side audit logging is currently limited to console logs on
privileged operations; a structured audit log on document state
transitions and plan changes is on the roadmap.

### Authenticated application — Information Disclosure (M)
Error responses are generic and do not include stack traces. Row-Level
Security enforces cross-tenant isolation at the database layer; cross-tenant
read attempts in Phase 2 testing returned empty result sets.

### Authenticated application — Denial of Service (P)
Per-user rate limits cap per-window calls on each privileged endpoint;
caveat ZNV-2026-001 applies. The PDF render path is the heaviest endpoint
and is rate-limited per user.

### Authenticated application — Elevation of Privilege (M)
The `protect_profile_update` trigger blocks self-promotion to administrator
and self-upgrade between plan tiers, regardless of whether the request
arrives via the application UI or via a direct database call.

### API routes — Spoofing (M)
Every privileged route runs origin validation, then session credential
verification, then rate limiting before reaching business logic. Admin
endpoints additionally verify the caller's `role` field against the
database.

### API routes — Tampering (M)
Inputs are validated by schema validators before use. Service-role mutations
carry an explicit `.eq("user_id", user.id)` scope to prevent cross-tenant
writes even though the service role bypasses Row-Level Security.

### API routes — Repudiation (P)
See Authenticated application — Repudiation. Same gap.

### API routes — Information Disclosure (M)
Error responses are generic. The administrative user-list endpoint returns
high-cardinality data to administrators only and would benefit from
pagination at scale.

### API routes — Denial of Service (P)
See Authenticated application — Denial of Service. Same caveat.

### API routes — Elevation of Privilege (M)
Administrator role is verified at every administrative endpoint; the
database-level trigger prevents self-promotion regardless of the access
path.

### Database — Tampering (M)
Row-Level Security enforces tenant scope. Service-role usage is confined
to API routes that re-scope by `user_id` in code. Triggers prevent
forbidden field mutations.

### Database — Repudiation (P)
The managed Postgres platform retains audit logging at its default level.
Application-level audit logs cover privileged Stripe state changes but not
all user mutations.

### Database — Information Disclosure (M)
Row-Level Security is the primary mechanism. Phase 3 confirmed every
public-schema table has RLS enabled and policy-covered. Service-role
escapes are deliberate and reviewed.

### Database — Denial of Service (O)
Query patterns are simple and indexed where it matters
(`idx_documents_user_id`, `idx_documents_type`, `idx_documents_status`,
`idx_clients_user_id`). Connection pool exhaustion is bounded by the
managed platform's default pool. A formal slow-query review has not yet
been performed.

### Database — Elevation of Privilege (M)
`SECURITY DEFINER` functions are scoped to the service role for intentional
operations. Finding ZNV-2026-007 documents a gap in the implementation of
this scoping due to Postgres `PUBLIC` role inheritance; remediation is
scheduled.

### Stripe webhook — Spoofing (M)
The Stripe SDK's signature-construction primitive rejects payloads without
a valid signature. Replay is bounded by the signature's timestamp window.

### Stripe webhook — Tampering (M)
Signature covers the full body; tampered payloads fail verification.

### Stripe webhook — Repudiation (M)
Stripe Dashboard preserves the full event log; the application's webhook
handler logs each event type for cross-reference.

### Stripe webhook — Denial of Service (O)
Webhook handlers do not currently deduplicate by `event.id`. If Stripe
retries on a 5xx response, idempotency relies on the underlying SQL update
being naturally idempotent, which it is for the implemented event types.
A dedicated idempotency table is on the roadmap.

### Stripe webhook — Elevation of Privilege (M)
Even with a valid signature, the handler additionally requires a `userId`
in metadata and double-scopes the database update by document and owner.
Forged metadata inside a forged-but-signed event would still target zero
rows.

### Email pipeline — Spoofing (M)
SPF, DKIM, and DMARC are configured on the sender domain. The sender
address is fixed and not user-controllable.

### Email pipeline — Tampering (M)
The recipient field is validated as a well-formed email. Subject lines are
stripped of `\r` and `\n` to prevent header injection. All user-supplied
body fields are HTML-escaped consistently.

### Email pipeline — Repudiation (P)
The email provider maintains delivery logs. Per-user send tracking inside
the application is on the roadmap.

### Email pipeline — Information Disclosure (M)
Recipient is the document's intended client; PDF attachments are generated
per-request from authenticated data. No customer data leaks between sends.

### Email pipeline — Denial of Service (P)
Per-user send rate limits cap abuse. The application-instance bound
described in ZNV-2026-001 applies here as well; the email provider's quota
provides an additional ceiling.

### PDF render — Tampering (M)
Templates are server-controlled React components. User input is treated as
data, not as template syntax.

### PDF render — Information Disclosure (M)
Render is in-process with no external resource fetch (no SSRF surface).
User-controlled fields are constrained to the document's own data.

### PDF render — Denial of Service (M)
Document size is bounded by schema constraints (numeric precision,
`line_items` JSONB size). PDF generation is rate-limited per user via the
email-send endpoint.

### PDF render — Elevation of Privilege (M)
Plan tier is verified server-side at render time. Free-plan users cannot
select premium themes; only studio-plan users receive the no-watermark
render.

### Scheduled jobs — Spoofing (M)
Every cron handler verifies `Authorization: Bearer <secret>` against an
environment-resident secret. Routes return HTTP 401 otherwise.

### Scheduled jobs — Tampering (O)
The recurring-invoice and client-reminder cron handlers perform multi-step
operations without transactional wrapping; see ZNV-2026-003.

### Scheduled jobs — Repudiation (P)
Cron runs are logged via the platform's runner. The application logs
record counts of records processed per run. A per-record audit trail is
implicit in the resulting documents and notifications.

### Scheduled jobs — Elevation of Privilege (M)
Cron handlers run as service role but operate on bounded query sets
(WHERE clauses tied to scheduled state, not user-supplied input).

## Top remediations from the matrix

The three highest-priority items surfaced by the STRIDE pass are also
documented as findings in `report.md`:

1. ZNV-2026-001 — In-process rate limiting on serverless platform.
2. ZNV-2026-002 — Email recipient not constrained to known clients.
3. ZNV-2026-003 — Recurring job handlers not transactional.
