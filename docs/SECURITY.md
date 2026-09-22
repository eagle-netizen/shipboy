# ShipBoy Security Model

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md)

**Status:** Security requirements and proposed controls for ParcelOS-first build.

**Compliance note:** This document does **not** claim ISO, SOC 2, PCI, GDPR, or India DPDP certification or legal compliance. Those require separate legal/process work. Treat the following as engineering security baseline.

---

## 1. Security goals

1. Protect tenant data with **strict organization isolation** by default.
2. Authenticate every user; authorize every protected operation **server-side**.
3. Treat **portals as UX only**—frontend route access is never authorization.
4. Keep secrets out of Git, logs, and clients.
5. Make damaging operations **idempotent** and **auditable**.
6. Verify external webhooks; treat provider payloads as untrusted input.
7. Minimize PII exposure; control document/label access.
8. Separate **employee/admin** privileged access from customer membership; never grant standing unrestricted customer-data access.
9. Make **authorized cross-organization** (FreightOS) and privileged access **explicit and auditable**.
10. Prefer simple, enforceable controls over theatrical complexity.

---

## 2. Authentication

### 2.1 User authentication (Proposed)

- Email + password for MVP web app.
- Passwords hashed with **Argon2id** (preferred) or bcrypt.
- Sessions for browser clients: **HTTP-only, Secure, SameSite** cookies.
- CSRF protection for cookie-based session mutations (**Confirmed need** if cookie auth).
- Optional API bearer tokens for integration clients—**Future consideration** / **Decision required** timing.
- Email verification before sensitive actions—**Proposed**.
- Password reset via time-limited single-use tokens—**Proposed**.
- **One identity** across Customer, Forwarder, Ops, and Admin portals—no per-portal user accounts.

### 2.2 Session / token handling (Proposed)

| Control | Rule |
|---------|------|
| Session TTL | Absolute + idle timeouts (**Decision required** exact values) |
| Rotation | Rotate session id on login |
| Revocation | Membership revoke / password change / staff privilege revoke invalidates sessions **Proposed** |
| Storage | Server-side session store or signed session with revocation list—**Decision required** |
| JWT | If used, short-lived access + refresh rotation; prefer not storing long-lived JWT in localStorage |
| Portal hint | May store last portal / active org as UX preference only—**never** sole authz input |

### 2.3 MFA and step-up (Confirmed requirement; implementation Decision required)

- **Platform staff:** MFA and/or **step-up authentication** is **required** for privileged actions (Admin Portal sensitive operations and other privileged staff actions as defined in permission policy).
- **MFA implementation details** (provider, factors, step-up triggers, recovery) remain **Decision required** until the technical architecture phase.
- Org-owner MFA for customer accounts remains **Future consideration** / optional later unless elevated risk demands earlier.

### 2.4 Platform admin / employee authentication

- Staff use the same identity system with a **separate platform staff membership model** ([Database](./DATABASE.md)).
- Admin Portal and privileged staff actions must be strongly gated (staff role + **MFA/step-up**).
- Additional controls (IP allowlist, etc.) remain **Proposed** / **Decision required**.
- Break-glass production infrastructure access remains separate from app Admin Portal ([§18](#18-production-access-controls-confirmed-intent)).

---

## 3. Authorization, RBAC, and portals

### 3.1 Principles (Confirmed)

- Authentication ≠ authorization.
- **Portal ≠ authorization.** Frontend portal restrictions are UX only.
- A user must **not** gain access merely because they can reach a frontend route (e.g., `/admin`, `/ops`).
- Authorization **MUST** be enforced server-side on every protected API/operation.
- Deny by default.

### 3.2 Authorization inputs (Confirmed)

Server checks should consider, as applicable:

- user identity
- organization (active membership context)
- membership
- role
- permissions
- resource ownership
- portal/context (telemetry / wrong-portal misuse signals—**Proposed**; not sufficient alone)
- organization relationships (FreightOS participants)
- platform-level privileges (employee/admin)

### 3.3 Portal vs role vs permission (Confirmed)

| Concept | Security treatment |
|---------|-------------------|
| Portal | Navigation/layout boundary only |
| Role | Maps to permissions in a context |
| Permission | Enforced allow/deny on the backend |

Do **not** implement authorization as simple frontend role checks.

### 3.4 Customer and forwarder RBAC (Confirmed)

- Organization roles (**Confirmed**): `owner`, `admin`, `ops`, `viewer` (same names for exporter and forwarder orgs; permissions differ by assignment/org type).
- Platform staff roles (**Confirmed**): `platform_admin`, `platform_ops`, `platform_support`.
- Customer Portal actions require `exporter` or `both` org membership + permission.
- Forwarder Portal actions require `forwarder` or `both` org membership + permission.
- Roles are **org-scoped**; not globally fixed.
- Permission storage is **hybrid**: role/permission assignments in DB; permission definitions in code ([BR-R10](./BUSINESS_RULES.md)).

### 3.5 Employee / Operations access (Confirmed)

- Employee Portal is separate from customer UX; sequenced **later** than Customer + thin Admin ([BR-R11](./BUSINESS_RULES.md)).
- Employees must **not** automatically have unrestricted access to customer data.
- Each support/investigation capability requires an **explicit** permission.
- **Sensitive/privileged reads** must be audited per [BR-A3](./BUSINESS_RULES.md)—not ordinary dashboard/list/detail reads.
- Prefer ticket-/case-scoped access patterns when practical—**Proposed** / **Future consideration**.

### 3.6 Admin Portal access (Confirmed — thin Admin MVP)

Thin Admin MVP includes ([BR-R12](./BUSINESS_RULES.md)):

- organization / user management
- platform staff management
- carrier / integration configuration
- basic system health
- audit log access
- basic support / operational lookup

**Out of initial Admin MVP:** advanced analytics, billing administration, complex operational tooling.

Admin privileges are distinct from customer-org roles. Strongly protect Admin Portal: authn hardening + MFA/step-up for privileged actions (**implementation Decision required**) + permission checks + audit.

### 3.7 Cross-organization FreightOS access (Confirmed)

Exporter RFQ ↔ forwarder claim/quote/booking is intentional authorized cross-org access—not a tenancy bug—provided:

- **RFQ discovery:** eligible marketplace visibility with controlled eligibility ([BR-F3](./BUSINESS_RULES.md)).
- Quotes/bookings enforce participant org ids.
- Neither party gains unrestricted access to the other’s tenant.
- Mutating cross-org actions are audited; sensitive/privileged reads audited per BR-A1 (not every read).
- **Documents:** default deny; cross-org access only via **explicit resource-based grants** ([BR-D4](./BUSINESS_RULES.md)).

See [Business Rules §14](./BUSINESS_RULES.md).

---

## 4. Tenant isolation

### 4.1 Application-layer isolation (Confirmed)

- Tenant context derived from authenticated session + selected organization—not from client-supplied org id alone without membership check.
- Repositories/services require ownership predicates for default access; “get by id” alone is insufficient.
- Automated tests for cross-tenant IDOR and cross-portal privilege escalation.

### 4.2 Defense in depth (**Decision required**)

- PostgreSQL **Row Level Security (RLS)** remains **Decision required** for the technical architecture phase (including any staff bypass design).
- Until decided, application-layer isolation and tests are mandatory.

### 4.3 Worker / job isolation (Confirmed)

Jobs carry `organization_id` (or relationship ids); workers re-validate resource ownership / relationship authorization before side effects.
---

## 5. API authentication

| Client | Mechanism (Proposed) |
|--------|----------------------|
| All web portals (Customer, Forwarder, Ops, Admin) | Session cookie (shared identity) |
| Future public API | API keys or OAuth2—**Decision required** |
| Inbound webhooks | Provider signatures, not user sessions |
| Internal worker | Not exposed publicly; private network / process |

Rate limit authenticated and unauthenticated endpoints ([§11](#11-rate-limiting-and-abuse-prevention)). Apply stricter limits to Admin/Ops mutation endpoints (**Proposed**).

Portal path or `X-Portal` header may be recorded for audit/telemetry—**must not** authorize by itself.
---

## 6. Password and account security requirements (Proposed)

- Minimum password length ≥ 10 (**Decision required** exact policy).
- Block breached/common passwords if feasible (e.g., list check)—**Proposed**.
- Lockout / backoff after repeated failures—**Proposed**.
- Do not reveal whether email exists on login (**Proposed** uniform errors)—balance with UX **Decision required**.

---

## 7. Secrets management (Confirmed)

- **No secrets in Git** (API keys, DB URLs with passwords, marketplace tokens, carrier credentials).
- Environment variables or secret manager (AWS SM, GCP SM, Doppler, etc.)—**Decision required** vendor.
- Separate secrets per environment (`staging` / `production`).
- Organization-level integration credentials encrypted at rest with envelope encryption (**Proposed**: KMS/app master key).
- Rotation procedures documented operationally before production launch.
- `.env` files local-only; provide `.env.example` without secrets when tooling is added.

---

## 8. Webhook verification (Confirmed)

- Verify signatures/HMAC per provider documentation before processing.
- Reject invalid signatures; log security event without trusting body.
- Use raw request body for signature validation.
- Idempotent processing via `webhook_events` unique keys ([Database](./DATABASE.md)).
- Do not expose webhook URLs that perform state changes without verification.
- Time-tolerance for timestamps where providers support it—**Proposed**.

---

## 9. Encryption

| Data | Requirement |
|------|-------------|
| In transit | TLS 1.2+ for all public endpoints (**Confirmed**) |
| At rest (DB/disks) | Rely on managed Postgres/disk encryption where available (**Proposed**) |
| Integration secrets | Field-level encryption before DB store (**Proposed**) |
| Files in object storage | SSE (S3/R2 default) + private buckets (**Confirmed** intent) |
| Backups | Encrypted backups (**Proposed**) |

Application-level encryption of all PII columns is **Future consideration** unless regulation demands earlier.

---

## 10. File and document access control (Confirmed)

- Documents and labels stored in **private** object storage.
- Download/view only after authz: membership + permission + ownership **or** explicit participant/share rule **or** scoped staff permission.
- Prefer **short-lived signed URLs**; do not use permanent public URLs for private docs.
- Validate content types/sizes on upload; scan for malware **Proposed** / **Future consideration**.
- DocsOS links do not bypass tenant checks ([BR-D1](./BUSINESS_RULES.md)).
- Staff document access is audited.
---

## 11. Rate limiting and abuse prevention

### Confirmed need

- Rate limit login, password reset, and public APIs.
- Rate limit marketplace/carrier proxy endpoints to protect credentials and cost.

### Proposed controls

- Per-IP and per-user/org limits.
- Quotas on bulk shipment size and import batch size.
- FreightOS: rate limit claim attempts; still enforce DB concurrency cap of 10.
- Stricter rate limits on Admin/Ops sensitive endpoints.
- Bot protection on auth endpoints (Turnstile/hCaptcha)—**Decision required**.
- Anti-circumvention telemetry for FreightOS **must not** hostage user data exports ([BR-F11](./BUSINESS_RULES.md)).

---

## 12. Input validation and output safety (Confirmed)

- Validate and sanitize all external input (HTTP, webhooks, imports).
- Parameterized queries / ORM only—no string-built SQL.
- Strict file metadata validation.
- Avoid SSRF in adapters: deny arbitrary URL fetch from user input; allowlist provider endpoints.
- Standard security headers on web responses (**Proposed**: CSP, etc.).

---

## 13. Secure external integrations (Confirmed)

- Carrier and marketplace adapters use server-side credentials only.
- Least-privilege OAuth scopes; Amazon features gated by **actual SP-API permissions**.
- Timeouts on outbound calls; bounded retries; no infinite retry storms.
- Treat provider data as untrusted; map through DTOs.
- Do not invent capabilities; capability flags drive UI/API ([Business Rules](./BUSINESS_RULES.md)).

---

## 14. Payment security principles

Subscription / automated platform billing is **deferred** from the initial MVP ([BR-BILL1](./BUSINESS_RULES.md)). Architecture should remain billing-ready without implementing card/subscription flows yet.

When FreightOS booking protection / subscriptions appear later:

- Prefer PCI-compliant PSP (Razorpay/Stripe/etc.)—**Decision required** at that time.
- No card PAN/CVV storage on ShipBoy servers.
- Idempotent payment intents and webhooks ([BR-I1](./BUSINESS_RULES.md)).
- Reconcile payment state via provider webhooks + ledger—**Proposed**.

Do not claim PCI compliance until validated.

---

## 15. Logging, PII, and sensitive data

### Confirmed

- Never log passwords, session tokens, API keys, access tokens, or raw credential JSON.
- Redact authorization headers.
- Audit logs record actions, not secrets.
- Privileged employee/admin access to tenant resources must be auditable.
- **Sensitive/privileged reads** must be audited per [BR-A3](./BUSINESS_RULES.md) (PII, private documents, financial/billing, security/auth settings, cross-org resources, platform/admin data, support/impersonation)—**not** ordinary dashboard/list/detail reads.
- Material cross-organization mutating actions must be auditable.

### Proposed

- Structured logs with `request_id`, `organization_id`, `user_id`, `portal`, `actor_type` (`member`/`staff`).
- Minimize buyer phone/address in application logs; prefer ids.
- `raw_snapshot` / webhook payloads may contain PII—restrict access (including Ops Portal), retention **Decision required**.

---

## 16. Data retention and privacy considerations

- Retention schedules for orders, tracking, documents, webhooks, audit—**Decision required** (legal input).
- Support org data export / deletion workflows—**Proposed** for offboarding (align BR-T4).
- Do not assert GDPR/DPDP compliance in marketing or docs until counsel reviews.
- Privacy policy / terms are legal artifacts outside this engineering doc.

---

## 17. Backup and recovery considerations (Proposed)

- Automated PostgreSQL backups with tested restore.
- Object storage versioning/replication as appropriate.
- Document RPO/RTO targets—**Decision required**.
- Backup access restricted like production data.

---

## 18. Production access controls (Confirmed intent)

- Least-privilege cloud IAM.
- No direct production DB mutation outside migrations/controlled jobs ([AGENTS.md](../AGENTS.md)).
- Secrets rotation without committing values.
- Separate staging and production credentials.
- Deploy via reviewed pipeline—not ad-hoc prod SSH as normal workflow (**Proposed** hardening).
- App Admin Portal ≠ infrastructure root access; both must be controlled and logged.

---

## 19. Dependency and security update practices (Proposed)

- Lockfiles committed when app exists.
- Dependabot/Renovate for vulnerable dependencies.
- Periodic `npm audit` / equivalent in CI.
- Review adapter SDK releases for breaking security changes.
- Do not run untrusted scripts from marketplace payloads.

---

## 20. Threat-oriented control map (summary)

| Threat | Primary controls |
|--------|------------------|
| Cross-tenant data leak | Tenant predicates, RBAC, tests, optional RLS |
| Portal URL privilege escalation | Server-side authz; portal is UX only |
| Insider / employee overreach | Explicit staff permissions; audit; no auto full access |
| Admin misuse | Strong admin gating; audit; separate from customer roles |
| Unauthorized FreightOS peeking | Relationship-based authz; no unrestricted peer access |
| Credential theft from repo | Secret manager; no Git secrets |
| Replay webhooks | Signature + idempotency keys |
| Duplicate shipments | Idempotency records + carrier constraints |
| RFQ claim overrun | DB transaction + unique constraints |
| SSRF via integrations | Allowlisted provider endpoints |
| Document leakage | Private buckets + signed URLs + authz |
| Brute-force login | Rate limits + lockout |

---

## 21. Security acceptance criteria (engineering)

- Cross-tenant API tests fail closed.
- Navigating to `/admin` or `/ops` without privileges cannot perform admin/ops API actions.
- Customer-org `owner` cannot call platform admin APIs.
- Employee without explicit permission cannot retrieve arbitrary tenant orders/shipments.
- Privileged staff access to tenant resources produces audit events.
- Forwarder cannot read unrelated exporter tenant data outside authorized RFQ/claim/quote/booking scope.
- Webhook without valid signature is rejected and not processed.
- Replayed webhook does not duplicate shipment/order side effects.
- Document URL is not guessably public without authz.
- Creating shipment with Idempotency-Key replay returns original result without second carrier create (**Proposed** API behavior).
- RFQ claim race cannot exceed 10 rows.
- Secrets scanning in CI—**Proposed** when toolchain exists.

---

## 22. Open security decisions

| Topic | Status |
|-------|--------|
| Exact auth library / session store | Decision required |
| Org-owner MFA timing | Future consideration |
| **MFA implementation details** | **Decision required** (requirement Confirmed) |
| Admin IP allowlist | Proposed / Decision required |
| **PostgreSQL RLS** (incl. staff bypass design) | **Decision required** |
| Password policy specifics | Decision required |
| API keys for third parties | Decision required |
| Bot protection vendor | Decision required |
| Retention periods | Decision required |
| Secret manager vendor | Decision required |
| Case-scoped support access model | Future consideration / Decision required |

### Founder-confirmed (cumulative)

| Topic | Decision |
|-------|----------|
| Role catalogs | Org: `owner`/`admin`/`ops`/`viewer`; Staff: `platform_admin`/`platform_ops`/`platform_support` |
| Permission storage | Hybrid: DB assignments, code definitions |
| Portal sequencing | Customer + thin Admin → Ops later → Forwarder with FreightOS |
| Thin Admin MVP | Org/user, staff, carrier/integration config, basic health, audit logs, basic support lookup |
| Routing | Path prefixes initially |
| Web architecture | Single web app initially |
| Organization type | `exporter`, `forwarder`, `both` |
| Platform staff | Separate staff membership + MFA/step-up (impl details Decision required) |
| RFQ discovery | Eligible marketplace visibility with controlled eligibility |
| Privileged reads | BR-A3 allowlist; not ordinary list/detail reads |
| Cross-org documents | Explicit resource-based access, default deny |
| International MVP | Core workflow; verified integrations only |
| Billing | Deferred from initial MVP; architecture billing-ready |
| Notifications | Basic abstraction + essential transactional only |---

*Security-relevant product changes must update this document alongside Business Rules and Architecture.*
