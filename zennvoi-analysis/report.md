# Zennvoi: Security and Architecture Assessment

## Document Information

| Field | Value |
|---|---|
| Project | Zennvoi |
| Application | Production invoicing SaaS — [https://www.zennvoi.com](https://www.zennvoi.com) |
| Assessment type | Self-conducted security and architecture review |
| Author | Luis Alvero |
| Author role | Founder and sole engineer |
| Assessment dates | 2026-04-29 to 2026-05-01 |
| Document version | 1.0 |
| Document status | Final |
| Distribution | Public |
| Companion documents | [`threat-model.md`](./threat-model.md), [`./diagrams/`](./diagrams/) |

---

## 1. Executive Summary

Zennvoi is a freelancer-focused invoicing and quoting application running on a serverless platform with a managed Postgres backend. This document records a self-conducted security and architecture review performed across three phases: static source review, dynamic verification against a running instance, and direct database-layer auditing.

Eight findings are reported, ranging in severity from Informational to Medium-High. None expose customer data or confer privilege escalation under the conditions tested. Each finding is accompanied by a recommended remediation and a target completion date.

The review confirms that the system's central authorization invariant — that all multi-tenant access is enforced at the database via Row-Level Security — holds across all sixteen public-schema tables. Three database-level safety triggers, including a DDL event trigger that automatically enables Row-Level Security on newly-created tables, mechanically enforce the discipline cost of this architectural choice. Webhook signature verification, request-origin validation, response-header baseline, and session handling each operate as designed under live testing.

The most significant areas for hardening are infrastructure-aware rate limiting on the application's serverless deployment (Finding ZNV-2026-001) and tighter recipient validation in the email-send pipeline (Finding ZNV-2026-002). Both are scheduled for remediation in the next development iteration.

---

## 2. Engagement Overview

### 2.1 Scope

| Area | In scope | Out of scope |
|---|---|---|
| Application source | Full TypeScript codebase (`/src`) | Marketing site, mobile application (in development) |
| Running endpoints | All `/api/*` route handlers, tested against a local instance | Production endpoints |
| Database | All public-schema tables, all Row-Level Security policies, storage bucket policies | Internal platform implementation details |
| Integrations | Stripe webhook handling, email sender configuration | Third-party platform internals |

### 2.2 Methodology

The review was conducted in three sequential phases.

**Phase 1: Static review.** Every source file in the repository was read. For each entry point, the trust boundary it crosses, input validation, authentication and authorization paths, side effects, and trust assumptions were recorded. A STRIDE matrix per component was produced; the full matrix is published as a companion document at [`threat-model.md`](./threat-model.md).

**Phase 2: Dynamic verification.** A local instance of the application was stood up with payments configured in test mode. Twenty-five tests were executed in two layers. The unauthenticated layer covered fifteen probes against privileged endpoints, including specific tests for the CSRF subdomain-suffix pattern, missing and malformed Stripe webhook signatures, and missing scheduled-job authorization. The authenticated layer covered ten probes covering cross-tenant read and write attempts via direct database queries, privilege-escalation attempts on the user profile, mass-assignment attempts on protected fields, and abuse-surface tests on the email-send and file-upload pipelines.

**Phase 3: Database-layer audit.** Six SQL queries were executed against the production database via the managed-platform's SQL editor. The queries verified that every public-schema table has Row-Level Security enabled, that policy verb coverage matches schema declarations, that no tables are bricked or have orphaned policies, that storage buckets have correct visibility and policy coverage, and that `SECURITY DEFINER` functions have appropriate execute privileges.

### 2.3 Tools

| Tool | Use |
|---|---|
| Browser DevTools | Network inspection, cookie examination, session decoding |
| `curl` (Git Bash for Windows) | HTTP endpoint probing, header and method testing |
| Supabase SQL Editor | Direct query execution for Phase 3 audit |
| Stripe CLI (`stripe listen`) | Test-mode webhook forwarding to local instance |
| Local source review | Read-only checkout of the repository |

### 2.4 Severity definitions

| Severity | Criteria |
|---|---|
| Critical | Direct unauthenticated access to customer data or full account takeover. |
| High | Significant data exposure or privilege escalation requiring some preconditions. |
| Medium | Abuse surface enabling reputational or operational impact, or material correctness defects on user-facing financial paths. |
| Low | Hardening or hygiene issues with no direct exploitation path, or minor information disclosure. |
| Informational | Architectural observations or process recommendations with no immediate exploitation context. |

Severity in this report combines impact and likelihood. Where reasonable, the rationale for the rating is explained in the finding description.

---

## 3. Findings Summary

| ID | Title | Severity | Component | Status |
|---|---|---|---|---|
| ZNV-2026-001 | In-process rate limiting on serverless platform | Medium-High | API | Open |
| ZNV-2026-002 | Email recipient not constrained to known clients | Medium | Email pipeline | Open |
| ZNV-2026-003 | Recurring job handlers not transactional across multi-step operations | Medium | Background jobs | Open |
| ZNV-2026-004 | Mass-assignment surface on direct profile updates | Low-Medium | Database | Open |
| ZNV-2026-005 | Account-deletion cascade not transactional | Low-Medium | API | Open |
| ZNV-2026-006 | Disposable-email rejection enforced client-side only | Low | Authentication | Open |
| ZNV-2026-007 | `SECURITY DEFINER` functions retain `PUBLIC` execute grant | Low | Database | Open |
| ZNV-2026-008 | Schema drift between repository and production database | Informational | Process | Open |

All findings are scheduled for remediation in the next development iteration; see Section 5 for per-finding target dates.

---

## 4. Architecture Overview

![Figure 1: System architecture](./diagrams/architecture.svg)

*Figure 1. Application components, data plane, external services, and trust boundaries.*

Zennvoi runs as a TypeScript Next.js application on a serverless hosting platform. The data plane is a managed Postgres database with Row-Level Security policies enforcing multi-tenant isolation on every table. Object storage holds user-uploaded logos in a public bucket and contract attachments in a private bucket. Stripe handles all payment flows; the system uses Stripe Connect to issue payouts directly to per-user Stripe accounts, ensuring Zennvoi never receives or stores card data. A transactional email API handles outbound delivery for invoices, reminders, and payment notifications. Scheduled tasks run on the platform's native cron with handlers gated by a shared secret.

### 4.1 Trust boundaries

The system has five distinct trust boundaries.

**Browser to application.** Transport security uses TLS with HTTP Strict Transport Security enforced at one year and `includeSubDomains` set. Cross-site request forgery is prevented through exact-origin matching that explicitly rejects subdomain-suffix patterns of the form `https://zennvoi.com.attacker.com`. The response-header baseline includes `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, a `Permissions-Policy` denying camera, microphone, and geolocation, and `Referrer-Policy: strict-origin-when-cross-origin`. All headers were confirmed present and correctly valued during Phase 2.

**Application to database.** Client-side reads and writes flow through Postgres with the user's session JWT bound to `auth.uid()`. Row-Level Security policies use this binding to scope every query. Service-role keys are used only inside API routes for intentionally privileged operations and are never exposed to browser code.

**Application to Stripe.** Outbound calls use server-side keys. Inbound webhooks are verified with the Stripe SDK's signature-construction primitive, which validates both signature and timestamp. Webhook handlers additionally re-verify metadata fields against database state before mutating records, providing defense in depth on top of signature verification.

**Application to email provider.** Outbound only. The sender domain is configured with SPF, DKIM, and DMARC aligned. User-supplied content is HTML-escaped at every embed point. Subject lines are stripped of carriage-return and line-feed characters to prevent header injection.

**Scheduled runner to application.** Cron handlers require an `Authorization: Bearer` header verified against an environment-resident secret. Tests confirmed that unauthenticated and wrong-secret invocations return HTTP 401.

### 4.2 Authentication and sessions

![Figure 2: Authentication flow](./diagrams/auth-flow.svg)

*Figure 2. Signup, login, and authenticated database access flows.*

Email-and-password authentication is handled by the managed authentication provider. Sessions are stored in HTTP-only secure cookies with token rotation managed by the provider's SDK. OAuth callback and password-reset routes exist for standard recovery paths.

A database trigger on `auth.users INSERT` automatically creates the application-level `profiles` row at signup time, ensuring atomic creation of the auth user and the application profile. A user cannot exist in one table without the corresponding row in the other.

API route handlers enforce a fixed pre-business-logic sequence: origin validation against an explicit allowlist, session credential verification, per-user rate limit application, and only then proceed to handler logic. The sequence is enforced by a shared utility module to prevent omission on new endpoints.

### 4.3 Authorization model and database-level defenses

![Figure 3: Core data model](./diagrams/data-model.svg)

*Figure 3. Core entities and their Row-Level Security tenant keys.*

Authorization is enforced exclusively at the database via Row-Level Security. The application layer does not maintain a separate authorization model. Every read and write — whether issued from browser code or from server-side API routes acting on behalf of the user — passes through Postgres with the user's session bound to `auth.uid()`. Row-Level Security policies on every tenant table reduce to `auth.uid() = user_id`.

This is a deliberate architectural choice. The advantage is locality: there is one place to get authorization correct, and that place is the same component that physically holds the data. The cost is the requirement that every new table opt into Row-Level Security at creation time and that every policy be deny-by-default.

Three database-level safety mechanisms address this discipline cost.

**`rls_auto_enable` event trigger.** A DDL event trigger fires on every `CREATE TABLE`, `CREATE TABLE AS`, and `SELECT INTO` in the public schema and automatically issues `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` against the new table. New tables are Row-Level Security enabled at creation time; no migration window exists in which an unprotected table could be reachable.

**`protect_profile_update` BEFORE UPDATE trigger.** The trigger raises an exception if any non-service-role connection attempts to modify the user's `role` or `plan` column. Even with a misconfigured Row-Level Security policy on `profiles`, a user cannot self-promote to administrator or self-upgrade to a paid tier. Phase 2 testing confirmed that both attempts raise `P0001` exceptions.

**`check_document_limit` BEFORE INSERT trigger.** The free plan caps document creation at three per calendar month. The trigger counts existing documents and raises an exception on the fourth. The cap is enforced at the database, not in the user interface.

The schema covers sixteen public-schema tables. Phase 3 confirmed that every table has Row-Level Security enabled, that every policy verb count matches schema declarations, and that no tables are in a bricked or orphaned-policy state.

---

## 5. Findings

### ZNV-2026-001 — In-process rate limiting on serverless platform

| | |
|---|---|
| Severity | Medium-High |
| Component | API (shared utilities) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-799: Improper Control of Interaction Frequency |

**Description.** The application implements per-user rate limits using a JavaScript `Map` resident in process memory. The hosting platform scales serverless function instances horizontally to meet demand; each cold-started instance has fresh, independent rate-limit state.

**Impact.** An adversary distributing requests across multiple function instances can exceed the intended per-window cap by a factor proportional to the number of active instances. While no tested endpoint can produce direct customer-data exposure even under unbounded request rates, increased traffic to email-sending endpoints could be used to abuse the platform's outbound channel beyond its intended limits.

**Recommendation.** Migrate rate-limit state to a request-coordinated store. Two options integrate cleanly with the current platform: a managed Redis service with native sliding-window primitives, or a Postgres counter table with row-level locking. Either preserves current per-endpoint limit definitions while making limits accurate under horizontal scaling.

---

### ZNV-2026-002 — Email recipient not constrained to known clients

| | |
|---|---|
| Severity | Medium |
| Component | Email pipeline (`/api/email/send-invoice`) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-285: Improper Authorization |

**Description.** The invoice-send endpoint validates that the recipient address is well-formed but does not verify that the address corresponds to the document's stored `client_email` field or to any row in the requesting user's `clients` table.

**Impact.** A user can dispatch invoice-styled email through the platform's outbound channel to arbitrary addresses. Per-user rate limits cap the total volume but do not eliminate the abuse surface; the pattern enables misuse of the platform's deliverable sender domain for unrelated communications.

**Recommendation.** Add a server-side check that the recipient address matches either the document's `client_email` value or any `email` value in the requesting user's `clients` table. Reject mismatches with HTTP 400.

---

### ZNV-2026-003 — Recurring job handlers not transactional

| | |
|---|---|
| Severity | Medium |
| Component | Background jobs (`/api/recurring/process`, `/api/cron/client-reminders`) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-662: Improper Synchronization |

**Description.** The recurring-invoice cron handler and the client-reminder cron handler each perform multi-step operations: insert a new row, update a counter on the user's profile, advance the schedule's next-fire date. Each step issues a separate database query. If the insert succeeds and a subsequent step fails, the platform's retry semantics will reprocess the same scheduled entry on the next run, producing duplicate inserts.

**Impact.** A user could be billed for or notified about a duplicate invoice in the event of partial failure during scheduled processing. The defect is correctness rather than security, but operates on the user-facing financial surface.

**Recommendation.** Wrap each multi-step handler in a `SECURITY DEFINER` Postgres function invoked via the SDK's RPC primitive. The function executes atomically: either every step commits or the entire operation rolls back.

---

### ZNV-2026-004 — Mass-assignment surface on direct profile updates

| | |
|---|---|
| Severity | Low-Medium |
| Component | Database (`profiles` table) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes |

**Description.** The application reads and writes the `profiles` table directly via the client SDK from browser code. The `protect_profile_update` trigger guards the `role` and `plan` columns against user mutation. Other columns — including Stripe identifier fields, onboarding flags, and formatting preferences — are mutable by the user via direct database PATCH operations.

**Impact.** No mutable field today confers security-meaningful privilege. The Stripe identifier fields require valid downstream state at Stripe to be useful, and the formatting fields affect only the user's own interface. The pattern, however, invites future bugs as the schema evolves: any new column added to `profiles` is mutable by default unless individually protected.

**Recommendation.** Introduce a server-side endpoint for profile updates with an explicit allowlist of user-mutable fields. Migrate user-interface write paths to the new endpoint. This converts the default behavior from "allowed unless explicitly protected" to "denied unless explicitly allowed."

---

### ZNV-2026-005 — Account-deletion cascade not transactional

| | |
|---|---|
| Severity | Low-Medium |
| Component | API (`/api/account/delete`) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-460: Improper Cleanup on Thrown Exception |

**Description.** Account deletion executes eight separate `DELETE` statements followed by an authentication-row deletion. The application-side deletes are not wrapped in a transaction. A failure during any intermediate delete leaves the user in a partially-deleted state.

**Impact.** A user requesting deletion may end up with some data persisted after the request returns success, depending on the failure point. The auth-row deletion is sequenced last, so re-running the endpoint typically completes the cleanup, but a hard mid-sequence failure is not cleanly recoverable without administrator intervention.

**Recommendation.** Consolidate the application-side deletes into a single Postgres function invoked via RPC. The function executes atomically; only the auth-row deletion remains as a separate cross-schema operation.

---

### ZNV-2026-006 — Disposable-email rejection enforced client-side only

| | |
|---|---|
| Severity | Low |
| Component | Authentication (signup form) |
| Status | Open |
| Target | Next development iteration |
| CWE | CWE-602: Client-Side Enforcement of Server-Side Security |

**Description.** The signup form rejects disposable email domains before submitting to the authentication provider. The provider's signup endpoint can be invoked directly without going through the form, bypassing the check.

**Impact.** An adversary can sign up with disposable email addresses, increasing the pool of throwaway accounts available for abuse of the platform's free tier and other unauthenticated-but-rate-limited surfaces.

**Recommendation.** Move the disposable-domain check to a server-side authentication hook or to a database trigger on `auth.users INSERT`. The client-side check may remain as a UX improvement but should not be the sole enforcement.

---

### ZNV-2026-007 — `SECURITY DEFINER` functions retain `PUBLIC` execute grant

| | |
|---|---|
| Severity | Low |
| Component | Database (privileged utility functions) |
| Status | Open |
| Target | Next migration |
| CWE | CWE-732: Incorrect Permission Assignment for Critical Resource |

**Description.** Postgres maintains a `PUBLIC` pseudo-role from which all roles inherit. The schema's `REVOKE EXECUTE ... FROM anon, authenticated` directives on privileged utility functions do not remove the implicit `PUBLIC` grant. As a result, the affected functions remain callable by any authenticated or unauthenticated client. The condition was identified by `has_function_privilege` queries during Phase 3.

**Impact.** Practical impact is low. The most exposed function manipulates a decorative counter that is not consulted by the actual free-plan enforcement trigger. A second function returns an aggregate database size, enabling minor information disclosure. Other affected functions are event-trigger handlers that produce no useful effect when invoked outside the trigger context.

**Recommendation.** Add explicit `REVOKE EXECUTE ... FROM PUBLIC` directives to each privileged function, then grant execute only to the `service_role`. Update the committed schema to follow this pattern by default.

```sql
REVOKE EXECUTE ON FUNCTION public.<fn>() FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.<fn>() FROM anon, authenticated;
GRANT  EXECUTE ON FUNCTION public.<fn>() TO service_role;
```

---

### ZNV-2026-008 — Schema drift between repository and production database

| | |
|---|---|
| Severity | Informational |
| Component | Process (schema management) |
| Status | Open |
| Target | Next development iteration |

**Description.** Phase 3 surfaced two pieces of evidence that the checked-in SQL migrations do not match the production schema. A column referenced by application code is absent from the committed schema files, and a database-level event trigger function exists in production without a corresponding entry in the repository.

**Impact.** The drift is in the safer direction (production has additional state not documented in the repository, rather than the repository expecting state that is missing in production). The condition is a process risk: a new environment provisioned from the committed schema would lack functionality that the production environment depends on, and review of changes is harder when the repository is not authoritative.

**Recommendation.** Adopt the managed platform's CLI for schema migrations. Treat the repository's migrations directory as authoritative. Establish a periodic diff between committed migrations and production schema state, and review any drift.

---

## 6. Verified Defenses

The Phase 2 dynamic test suite confirmed the following defensive controls operate as designed under live testing.

| Control | Test | Outcome |
|---|---|---|
| Origin validation, including subdomain-suffix attack | Request bearing `Origin: https://zennvoi.com.attacker.com` against a privileged endpoint | Rejected (HTTP 403) |
| Cron handler authorization | Scheduled-job endpoint invoked without and with incorrect `Authorization` header | Rejected (HTTP 401) |
| Stripe webhook signature verification | Webhook invoked without signature header and with malformed signature | Rejected (HTTP 400) |
| API authentication gate | Eight privileged endpoints invoked without credentials | All returned HTTP 401 prior to business logic |
| Cross-tenant database read | Authenticated user attempted to read another user's `documents` and `clients` rows via direct database queries | Empty result sets returned (Row-Level Security filtered) |
| Cross-tenant database write | Authenticated user attempted to update another user's `profiles` row | Empty result set returned (zero rows updated) |
| `protect_profile_update` trigger | Authenticated user attempted to modify own `role` and `plan` via direct database update | Both raised `P0001` exceptions (`Cannot modify role`, `Cannot modify plan`) |
| Payment-link URL validator | Email-send endpoint invoked with payment link set to non-`https://` and `javascript:` URI values | Both rejected by validator and stripped from outbound email |
| Response-header baseline | `curl -I` against the application root | Five expected headers present and correctly valued |

The trigger-based defenses and the Row-Level Security layer were independently confirmed by Phase 3 to apply across all sixteen public-schema tables.

---

## 7. Architectural Observations

The following observations are not findings. They inform recommendations and future architectural decisions.

**Database-level invariants are mechanically guaranteed.** The use of `BEFORE UPDATE`, `BEFORE INSERT`, and DDL event triggers to enforce authorization invariants at the database layer — rather than at the application layer or via continuous integration checks — converts security-critical conditions from socially enforced (the engineer remembers to add the check) to mechanically enforced (the database refuses transactions that would violate the invariant). This pattern is recommended for any invariant whose violation would have outsized impact relative to its enforcement cost.

**Webhook ownership double-checking.** The Stripe webhook handler validates the signature via the SDK and additionally re-verifies ownership metadata before mutating database state. The additional check is small (a single additional `eq` predicate) and provides defense in depth against compromise of upstream signature verification. The pattern generalizes to any incoming integration where event metadata is meant to authorize a state change.

**Postgres role inheritance behavior.** The handling of `SECURITY DEFINER` execute privileges (Finding ZNV-2026-007) demonstrates that `REVOKE EXECUTE FROM <named role>` is not equivalent to `REVOKE EXECUTE FROM PUBLIC`. Security controls that depend on role-based primitives should be empirically validated rather than inferred from `REVOKE` syntax. The Phase 3 audit query that surfaced this condition (`has_function_privilege`) is recommended as a periodic check.

**Production data exposure during local development.** The development environment for the system points at the production database by default. This is not a finding against the production system itself, but it represents a single-point-of-failure in operational practice: any local mutation reaches production data unless the engineer remembers not to write. Establishing a separate development project is on the operational roadmap.

---

## 8. Disclosure and Scope Notes

This document is a self-conducted review, not an independent third-party assessment. Findings reported above are reductions in abuse surface, architectural hygiene improvements, and process recommendations. None expose customer data or confer privilege escalation under the conditions tested.

The document deliberately omits exact numeric thresholds (rate-limit windows, account-lockout counts), specific column names from the schema where they would aid fingerprinting, and any current finding for which detailed exploitation guidance would meaningfully aid an attacker prior to remediation. Diagrams in [`./diagrams/`](./diagrams/) describe component relationships at the level of role rather than specific implementation. The companion threat model is published as [`threat-model.md`](./threat-model.md).

Independent review is welcomed. Comments may be addressed to the author via the contact information at [https://luisalvero.com](https://luisalvero.com).

---

## 9. Document History

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-05-01 | Initial publication. |

---

*Author: Luis Alvero. Engineer and operator of Zennvoi. Department of Electrical and Computer Engineering, Florida International University.*
