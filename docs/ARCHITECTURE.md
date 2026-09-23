# ShipBoy Architecture

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md)

**Status:** ParcelOS-first architecture. Core application/runtime stack choices below are **Confirmed** (Astra). Remaining open items stay marked **Decision required** / **Proposed**.

---

## 1. Goals of the architecture

- Ship **ParcelOS** first with clean boundaries for **FreightOS** and **DocsOS**.
- Prefer a **modular monolith** over microservices initially.
- Deliver a **multi-portal** frontend architecture (Customer, Forwarder, Employee/Ops, Admin)—not one generic dashboard ([Product](./PRODUCT.md)).
- Share **one backend / domain module set** across portaviewls; enforce authz server-side.
- Enforce **strict tenant isolation** by default, plus **explicit authorized cross-organization** paths for FreightOS ([Business Rules](./BUSINESS_RULES.md)).
- Isolate **carrier** and **marketplace** integrations behind adapters.
- Avoid premature infrastructure; keep local development simple.
- Remain maintainable for a small engineering team.

## 2. Application architecture recommendation

### Recommendation: Modular monolith + multi-portal web shell (Confirmed)

ShipBoy should start as a **single deployable backend** (API + workers) with **internal domain modules**, plus a **single web application** that hosts **isolated portal modules** (route groups)—not four unrelated frontend codebases on day one.

```text
┌──────────────────────────────────────────────────────────────────┐
│              Web application (shared shell, Proposed)            │
│  /app/* customer │ /forwarder/* │ /ops/* employee │ /admin/*     │
└───────────────────────────────┬──────────────────────────────────┘
                                │ HTTPS / JSON
┌───────────────────────────────▼──────────────────────────────────┐
│                 ShipBoy API (modular monolith)                   │
│  authn → portal/org context → authz → domain services            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐         │
│  │ Platform │ │ ParcelOS │ │ FreightOS│ │   DocsOS    │         │
│  │ identity │ │ orders   │ │ RFQ/quote│ │  documents  │         │
│  │ tenancy  │ │ ship     │ │ booking  │ │             │         │
│  │ RBAC     │ │ track    │ │ (later)  │ │             │         │
│  │ staff    │ │          │ │          │ │             │         │
│  └──────────┘ └──────────┘ └──────────┘ └─────────────┘         │
│  ┌────────────────────┐  ┌─────────────────────────────┐         │
│  │ Marketplace ports  │  │ Carrier ports               │         │
│  │ + adapters         │  │ + adapters                  │         │
│  └────────────────────┘  └─────────────────────────────┘         │
└───────────────┬───────────────────┬──────────────────────────────┘
                │                   │
         ┌──────▼──────┐     ┌──────▼──────┐
         │  PostgreSQL │     │ Object store│
         └─────────────┘     └─────────────┘
                │
         ┌──────▼──────┐
         │ Queue/Jobs  │
         └─────────────┘
```

### Why not microservices yet

- Team size and MVP scope do not justify distributed complexity.
- ParcelOS, FreightOS, and DocsOS share tenancy, auth, audit, and files.
- Multiple portals need the **same** domain APIs with different permission views.
- Cross-module transactions (order → shipment → document) are simpler in-process.
- Module boundaries preserve a future split path without paying microservice tax now.

**Future consideration:** Extract workers, carrier gateway, FreightOS bidding service, or a dedicated admin app only if scaling, compliance, or team boundaries demand it.

### Why not four separate frontend apps initially (Confirmed for initial stage)

- Duplicates auth, design system, and API clients.
- Slows ParcelOS MVP.
- Portal isolation is achieved with **route groups + layout boundaries + auth gates** in one Next.js app.
- **Future consideration:** split Admin and/or Ops into separate deployables if blast-radius or compliance requires it—without domain rewrite.
## 3. Domain modules

| Module | Responsibility | Priority |
|--------|----------------|----------|
| **Platform** | Organizations, users, memberships, RBAC, platform staff privileges, sessions/tokens, audit, idempotency, webhooks envelope, notifications stub | MVP |
| **ParcelOS** | Products/SKUs, orders, packages, shipments, tracking, carrier orchestration | MVP |
| **Marketplace adapters** | Etsy (first), later Amazon SP-API, Shopify | MVP (Etsy); others later |
| **Carrier adapters** | Delhivery, India Post first; capability negotiation | MVP |
| **DocsOS** | Document metadata, storage pointers, associations, access checks | Minimal MVP |
| **FreightOS** | RFQs, eligibility, claims, quotations, bookings, messaging hooks, cross-org access rules | Model early; feature later |

Portals are **not** domain modules. They are presentation/auth-context shells that call Platform/ParcelOS/FreightOS/DocsOS APIs.

### Internal dependency rules (Proposed)

- Adapters depend inward on ports defined by ParcelOS/FreightOS—not the reverse.
- ParcelOS and FreightOS may both use DocsOS and Platform.
- ParcelOS must not call FreightOS internals (and vice versa) except via explicit shared kernel types if needed.
- No business rule duplication in the frontend; UI calls API.
- Portal layouts may share design-system components but must not share “god” pages that mix admin + customer workflows.
## 4. Recommended technology stack

| Layer | Choice | Status | Why |
|-------|--------|--------|-----|
| **Frontend** | TypeScript + Next.js (App Router) | Proposed | Strong TS ecosystem, good auth/session patterns, SSR optional for marketing/app shell, one language with backend |
| **UI kit** | Decision required (e.g. Radix + Tailwind) | Decision required | Speed vs design system investment |
| **Backend** | TypeScript + **NestJS + Fastify** | **Confirmed** | NestJS modules/DI fit the modular monolith; Fastify as the HTTP adapter for performance and Nest ecosystem support |
| **API style** | REST JSON (OpenAPI) first | Proposed | Simplicity for MVP; GraphQL **Future consideration** |
| **DB** | PostgreSQL (managed) | **Confirmed** | Relational integrity for tenancy, uniqueness, RFQ claim constraints, audit |
| **ORM / query** | **Prisma** | **Confirmed** | Velocity for MVP schema + typed client; migrations via Prisma Migrate |
| **Migrations** | Prisma Migrate (mandatory) | **Confirmed** | Aligns with AGENTS.md |
| **Auth** | **Better Auth** + cookie sessions stored in **PostgreSQL**; staff **TOTP** + **backend-enforced step-up** | **Confirmed** | See [Security](./SECURITY.md). Optional bearer tokens for API remain **Future consideration** |
| **Password hashing** | Argon2id (or bcrypt if constrained) | Proposed | Modern default (library default as applicable) |
| **Jobs/queue** | PostgreSQL-backed jobs (e.g. Graphile Worker / BullMQ + Redis) | Proposed | PG-backed jobs for MVP to reduce moving parts; Redis+BullMQ if needed later |
| **Object storage** | **Private S3** (S3-compatible) | **Confirmed** | Labels, documents, invoice PDFs; private buckets only |
| **Email** | Transactional provider (SES/Resend/Postmark)—**Decision required** | Decision required | Notifications |
| **Observability** | Structured JSON logs + OpenTelemetry (phased) + error tracker (Sentry or similar)—**Decision required** vendor | Decision required | Debuggability |
| **Hosting** | **Render** + managed PostgreSQL + private S3 | **Confirmed** | Single-region MVP ops fit for a small team |
| **CI** | GitHub Actions | Proposed | Already on GitHub |

### Language rationale

TypeScript end-to-end reduces context switching for a small team building adapter-heavy integrations and a rich ops UI.

**Alternatives considered (not selected for MVP):** Fastify-only modular folders without NestJS; Drizzle; Python (Django/FastAPI).

## 5. Multi-portal frontend architecture (Confirmed direction; details as noted)

### 5.1 Recommendation for modular-monolith stage (Confirmed)

**Confirmed:** One web application with **isolated portal modules** (separate route groups + layouts), sharing:

- auth session client
- design system / UI primitives
- typed API client
- org/portal context providers

**Not for initial stage:** four fully separate frontend repositories.  
**Future consideration:** split Admin and/or Ops into separate deployables if blast-radius or compliance requires it.

### 5.2 Portal modules

| Portal | Purpose | Typical consumers | Sequencing |
|--------|---------|-------------------|------------|
| **Customer / Exporter** | ParcelOS + exporter FreightOS | Sellers/exporters | **First** (with thin Admin) |
| **Admin** | Platform administration | ShipBoy admins | **Thin Admin first** (with Customer) |
| **Employee / Ops** | Support & investigation | ShipBoy employees | **Later** |
| **Forwarder** | FreightOS forwarder workflows | Forwarders | **When FreightOS starts** |

### 5.3 Separation without wasteful duplication

- Shared: buttons, forms, tables, auth helpers, error boundaries.
- Isolated: navigation, page compositions, data views, permission-gated menus per portal.
- Domain mutations always via backend APIs with server-side authz.
- No embedding of carrier/marketplace secrets in the browser.
- Frontend may hide unauthorized nav items (**UX only**); APIs remain the enforcement point ([Security](./SECURITY.md)).

### 5.4 URL / route strategy (Confirmed: path prefixes initially)

**Confirmed for initial stage:** path-prefix portals on one domain:

| Prefix (illustrative) | Portal |
|-----------------------|--------|
| `/app/...` | Customer / Exporter |
| `/forwarder/...` | Forwarder |
| `/ops/...` | Employee / Operations |
| `/admin/...` | Admin |
| `/api/v1/...` | Backend API (same origin or API host—host split still **Decision required**) |
| `/login`, `/invite/...` | Auth entry (**Proposed**) |

Exact prefix strings may be adjusted; **path-prefix approach** is confirmed. Subdomains remain a **Future consideration** for stronger browser isolation later.

### 5.5 MVP frontend delivery (Confirmed sequencing)

1. Ship **Customer Portal** + **thin Admin** routes + auth + org switcher.
2. Add **Employee / Ops Portal** later.
3. Add **Forwarder Portal** when FreightOS implementation starts—without redesigning domain modules.

## 6. Backend architecture (Confirmed runtime; details as noted)

- **NestJS + Fastify** modular monolith process exposing versioned REST (`/api/v1/...`).
- Separate **worker process** sharing the same codebase for jobs (imports, tracking poll, webhook post-processing).
- Persistence via **Prisma** against managed PostgreSQL.
- Request pipeline: **authn (Better Auth) → resolve user → resolve org context and/or platform-staff context → authz (permissions + resource rules; staff step-up when required) → validation → domain service → adapters**.
- Optional `X-Portal` / path metadata for telemetry only—**must not** replace authz (**Proposed**).
- Carrier/marketplace HTTP clients isolated in adapter packages/folders.
- Feature flags (**Proposed**) to disable FreightOS / portal route modules until launch.
- Same domain endpoints may be called from multiple portals; authorization differs by caller privileges.
## 7. Database

- Single **managed PostgreSQL** database for MVP (hosted with **Render**).
- ORM: **Prisma**; all schema changes via **Prisma Migrate**. No direct production mutation.
- Logical separation by schemas **optional** (`platform`, `parcel`, `freight`, `docs`)—**Proposed** for clarity, not required day one.
- **Tenant isolation for MVP:** enforced at the **application layer** (tenant predicates on every tenant-owned query, indexes/constraints, automated IDOR tests). **PostgreSQL RLS is deferred** (not MVP)—revisit as defense-in-depth later ([Database](./DATABASE.md), [Security](./SECURITY.md)).

## 8. Authentication and authorization boundaries

See [Security](./SECURITY.md) for normative security detail. Architecture summary:

### Authentication

- Authenticate **user identity** once (shared across portals) via **Better Auth**.
- Establish **cookie-based sessions** persisted in **PostgreSQL**.
- **Platform staff:** **TOTP** MFA; **step-up** re-auth enforced on the **backend** for privileged actions ([Security](./SECURITY.md)).

### Context resolution (Decision required mechanism; Confirmed need)

After authn, resolve:

1. **Active organization** (for customer/forwarder membership paths)—header, path, or explicit switcher payload; **always** verify membership.
2. **Platform staff context** (for `/ops` and `/admin`)—via staff privilege records, **not** by inventing a fake customer membership.
3. **Portal** as UX context (optional claim)—never sufficient for authorization alone.

### Authorization (Confirmed)

- Enforce permissions server-side on every protected operation.
- Check resource ownership **or** explicit cross-org relationship (FreightOS) **or** scoped platform-staff permission.
- Deny by default.
- Workers re-check tenancy/ownership on job payloads.

### Portal vs authorization (Confirmed)

```text
Frontend portal route  →  UX only (what screens exist)
Backend permission     →  actual allow/deny
```

## 9. Tenant isolation and cross-organization access

- Every tenant-owned table includes owning `organization_id` (or equivalent).
- Domain repositories require tenant predicates for default access (**application-level isolation for MVP**; RLS deferred).
- Adapter credentials stored per organization connection, encrypted at rest ([Security](./SECURITY.md)).
- FreightOS cross-org reads/writes go through relationship-aware policies (RFQ/claim/quote/booking)—see [Business Rules §14](./BUSINESS_RULES.md) and [Database](./DATABASE.md).
- Platform staff access is a separate authorization mode with audit.
## 10. API architecture

- REST + OpenAPI spec as contract ([future `docs/API.md`](./API.md)).
- Consistent error envelope.
- Idempotency-Key support on shipment create / bulk fulfill / payments ([Business Rules](./BUSINESS_RULES.md)).
- Webhooks inbound: `/api/v1/webhooks/{provider}` with signature verification, raw body capture, idempotent processing.
- Public tracking pages (**Future consideration**).

## 11. Background jobs and queues

Use jobs for:

- Marketplace order import batches
- Tracking refresh / carrier poll
- Webhook async follow-up
- Bulk shipment fan-out
- (Later) RFQ expiry, eligibility recompute, notifications

**Proposed MVP:** PostgreSQL job queue to avoid Redis ops cost. Revisit Redis if throughput demands.

## 12. Object / file storage

- **Private S3** (S3-compatible) bucket(s).
- Store only object keys + metadata in DB (DocsOS).
- Access via short-lived signed URLs after authz check—never public buckets for private docs/labels.

## 13. Webhook processing

1. Verify signature / authenticity.
2. Persist raw event + unique delivery key (idempotency).
3. Enqueue processing job.
4. Processor applies domain updates transactionally; duplicates no-op.

Applies to carriers and marketplaces alike.

## 14. Notifications (Confirmed MVP scope)

MVP supports:

- a **basic notification abstraction** (provider/channel interchangeable later)
- **essential transactional notifications** (auth/security and critical operational messages as needed)

Do **not** build a large notification platform initially.

**Decision required (technical phase):** concrete email/SMS provider. WhatsApp/SMS beyond essentials: **Future consideration**.

Architecture: notification outbox table + worker; templates versioned—**Proposed** implementation shape.

## 15. Audit logging

- Append-only `audit_logs` written in the same transaction as the business mutation when feasible.
- Actor user id, organization id, action, entity type/id, metadata (non-secret), IP/user-agent **Proposed**.

## 16. Observability

- Structured logs with `requestId`, `organizationId`, `userId` (where safe).
- Metrics: API latency, job failures, adapter error rates, RFQ claim contention (later).
- Tracing optional in MVP; add OTel when pain appears.
- Never log secrets, access tokens, or full card data (no card data expected in ParcelOS MVP).

## 17. Carrier adapter architecture (Confirmed)

```text
ParcelOS ShipmentService
        │
        ▼
 CarrierPort (interface)
   createShipment()
   getRates()
   generateLabel()
   trackShipment()
   cancelShipment()
   schedulePickup()
   getServiceability()
        │
   ┌────┴─────┬────────────┐
   ▼          ▼            ▼
Delhivery   IndiaPost    Future...
Adapter     Adapter
```

- Capability discovery: adapter reports supported methods.
- Mapping layer translates ShipBoy shipment model ↔ carrier payloads.
- Timeouts, retries, and circuit breaking **Proposed** per adapter.
- Credentials resolved from org’s carrier connection secrets.

## 18. Marketplace adapter architecture (Confirmed)

```text
ParcelOS OrderImportService
        │
        ▼
 MarketplacePort
   connect / refresh tokens
   list/import orders
   (optional) push fulfillment / tracking
        │
   ┌────┴────┬──────────┐
   ▼         ▼          ▼
 Etsy      Amazon     Shopify
 Adapter   (SP-API)   (later)
```

- Amazon adapter must gate features on granted SP-API roles/permissions.
- Core Order entity stores `provider`, `external_order_id`, raw snapshot **Proposed** for support/debug (PII-minimized).

## 19. Deployment architecture (Confirmed hosting; shape Proposed)

**MVP (Confirmed platform):**

- **Render:** 1× API service + 1× worker service
- **Managed PostgreSQL** (Render)
- **Private S3** bucket
- Optional Redis only if job stack requires it

**Environments:** `local`, `staging`, `production`.

**No** multi-region active-active for MVP.

## 20. Local development architecture (Proposed)

- Docker Compose: PostgreSQL (+ MinIO as local stand-in for **private S3**).
- API + worker + web run on host or Compose—**Decision required** team preference.
- `.env.example` for non-secret defaults (file not created in this documentation pass).
- Seed script for demo org **Future consideration**.

## 21. ParcelOS-first delivery slices

1. Platform module (org types, auth, memberships, hybrid RBAC, platform staff membership, audit)
2. Customer Portal shell + thin Admin routes (path-prefix multi-portal-ready routing)
3. Products/SKUs + manual orders
4. Carrier port + Delhivery + India Post adapters (capability-honest)
5. Shipments, labels, tracking ingest
6. Bulk create
7. Etsy marketplace adapter + import
8. Minimal DocsOS attachments + `resource_access_grants` model for explicit shares
9. Employee / Ops Portal later
10. FreightOS + Forwarder Portal + eligible RFQ discovery when FreightOS starts

## 22. Explicit non-goals (architecture)

- Kubernetes requirement for MVP
- Event-sourcing-everything
- Separate database per tenant (unless later Decision required)
- Shared-nothing microservices before product-market fit
- Duplicating business logic in the frontend
- One generic dashboard for all roles
- Frontend-only authorization
- Separate user tables per portal
- Four separate frontend apps at initial stage

## 23. Decisions summary

| Topic | Status |
|-------|--------|
| Modular monolith backend | Confirmed |
| Multi-portal product | Confirmed |
| Shared backend modules across portals | Confirmed |
| Single web app initially | Confirmed |
| Path-prefix routes initially | Confirmed |
| Portal sequencing | Confirmed: Customer + thin Admin → Ops later → Forwarder with FreightOS |
| Thin Admin MVP scope | Confirmed ([BR-R12](./BUSINESS_RULES.md)) |
| Org types | Confirmed: `exporter`, `forwarder`, `both` |
| Org roles | Confirmed: `owner`, `admin`, `ops`, `viewer` |
| Platform staff roles | Confirmed: `platform_admin`, `platform_ops`, `platform_support` |
| Hybrid permission storage | Confirmed |
| Staff MFA/step-up | **Confirmed:** Better Auth; staff **TOTP**; **backend-enforced step-up** |
| International ParcelOS MVP | Confirmed: core workflow; verified integrations only |
| Billing | Confirmed deferred from MVP; architecture billing-ready |
| Notifications | Confirmed: basic abstraction + essential transactional |
| Carrier + marketplace adapters | Confirmed |
| PostgreSQL | **Confirmed** (managed, with Render) |
| **NestJS + Fastify** | **Confirmed** |
| **Prisma** | **Confirmed** |
| **Auth** | **Confirmed:** Better Auth + PostgreSQL sessions |
| **Hosting** | **Confirmed:** Render + managed PostgreSQL + private S3 |
| **Tenant isolation (MVP)** | **Confirmed:** application-level; **RLS deferred** |
| TypeScript + Next.js frontend | Proposed |
| PG jobs vs Redis/BullMQ | Proposed: PG jobs MVP |
| Org / staff context resolution mechanism | Decision required |
| Subdomains later | Future consideration |
| Split Admin/Ops apps later | Future consideration |
| PostgreSQL RLS (post-MVP) | Future consideration |

---

*Do not invent provider API capabilities in adapters. Align Database + Security when stack decisions change.*
