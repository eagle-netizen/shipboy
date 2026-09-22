# ShipBoy Architecture

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md)

**Status:** Initial architecture proposal for ParcelOS-first implementation. Stack choices below are **Proposed** until founder approval.

---

## 1. Goals of the architecture

- Ship **ParcelOS** first with clean boundaries for **FreightOS** and **DocsOS**.
- Prefer a **modular monolith** over microservices initially.
- Deliver a **multi-portal** frontend architecture (Customer, Forwarder, Employee/Ops, Admin)—not one generic dashboard ([Product](./PRODUCT.md)).
- Share **one backend / domain module set** across portals; enforce authz server-side.
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
## 4. Recommended technology stack (Proposed)

| Layer | Proposal | Why |
|-------|----------|-----|
| **Frontend** | TypeScript + Next.js (App Router) | Strong TS ecosystem, good auth/session patterns, SSR optional for marketing/app shell, one language with backend if Node chosen |
| **UI kit** | Decision required (e.g. Radix + Tailwind) | Speed vs design system investment |
| **Backend** | TypeScript + NestJS **or** Node/Fastify with modular folders | NestJS gives clear modules/DI fitting modular monolith; Fastify is lighter. **Decision required** between NestJS vs Fastify |
| **API style** | REST JSON (OpenAPI) first | Simplicity for MVP; GraphQL **Future consideration** |
| **DB** | PostgreSQL | Relational integrity for tenancy, uniqueness, RFQ claim constraints, audit |
| **ORM / query** | Prisma **or** Drizzle | Prisma = velocity; Drizzle = SQL-friendly. **Decision required** |
| **Migrations** | ORM migrations or Flyway/Liquibase-style—must be mandatory | Aligns with AGENTS.md |
| **Auth** | Session cookies (web) + optional bearer tokens for API | See [Security](./SECURITY.md). Prefer battle-tested library (e.g. Auth.js / Lucia / custom on Passport)—**Decision required** |
| **Password hashing** | Argon2id (or bcrypt if constrained) | Modern default |
| **Jobs/queue** | PostgreSQL-backed jobs (e.g. Graphile Worker / BullMQ + Redis) | Start with PG-backed if avoiding Redis; Redis+BullMQ if needed. **Proposed:** PG-backed jobs for MVP to reduce moving parts |
| **Object storage** | S3-compatible (AWS S3 or Cloudflare R2) | Labels, documents, invoice PDFs |
| **Email** | Transactional provider (SES/Resend/Postmark)—**Decision required** | Notifications |
| **Observability** | Structured JSON logs + OpenTelemetry (phased) + error tracker (Sentry or similar)—**Decision required** vendor | Debuggability |
| **Hosting** | Single region container (Fly.io / Railway / AWS ECS / Render)—**Decision required** | Match team ops comfort |
| **CI** | GitHub Actions | Already on GitHub |

### Language rationale

TypeScript end-to-end reduces context switching for a small team building adapter-heavy integrations and a rich ops UI.

**Alternative considered:** Python (Django/FastAPI) — excellent for backends, weaker default pairing with a modern TS React app unless split stacks are acceptable. **Decision required** if founder prefers Python.

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

## 6. Backend architecture (Proposed)

- Modular monolith process exposing versioned REST (`/api/v1/...`).
- Separate **worker process** sharing the same codebase for jobs (imports, tracking poll, webhook post-processing).
- Request pipeline: **authn → resolve user → resolve org context and/or platform-staff context → authz (permissions + resource rules) → validation → domain service → adapters**.
- Optional `X-Portal` / path metadata for telemetry only—**must not** replace authz (**Proposed**).
- Carrier/marketplace HTTP clients isolated in adapter packages/folders.
- Feature flags (**Proposed**) to disable FreightOS / portal route modules until launch.
- Same domain endpoints may be called from multiple portals; authorization differs by caller privileges.
## 7. Database

- Single PostgreSQL database for MVP.
- Logical separation by schemas **optional** (`platform`, `parcel`, `freight`, `docs`)—**Proposed** for clarity, not required day one.
- Tenant isolation enforced in application queries **and** supported by indexes/constraints; **PostgreSQL RLS** is **Decision required** for the technical architecture phase ([Database](./DATABASE.md), [Security](./SECURITY.md)).
- All schema changes via migrations. No direct production mutation.

## 8. Authentication and authorization boundaries

See [Security](./SECURITY.md) for normative security detail. Architecture summary:

### Authentication

- Authenticate **user identity** once (shared across portals).
- Establish session (cookie-based **Proposed** for web).

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
- Domain repositories require tenant predicates for default access.
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

- S3-compatible bucket(s).
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

## 19. Deployment architecture (Proposed)

**MVP:**

- 1× API service
- 1× worker service
- 1× PostgreSQL
- 1× S3-compatible bucket
- Optional Redis only if job stack requires it

**Environments:** `local`, `staging`, `production`.

**No** multi-region active-active for MVP.

## 20. Local development architecture (Proposed)

- Docker Compose: PostgreSQL (+ MinIO for S3-compatible local storage).
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
| Staff MFA/step-up required | Confirmed; **implementation details Decision required** |
| International ParcelOS MVP | Confirmed: core workflow; verified integrations only |
| Billing | Confirmed deferred from MVP; architecture billing-ready |
| Notifications | Confirmed: basic abstraction + essential transactional |
| Carrier + marketplace adapters | Confirmed |
| PostgreSQL | Proposed (strong recommendation) |
| TypeScript + Next.js frontend | Proposed |
| **NestJS vs Fastify** (or Python alt) | **Decision required** |
| **ORM** (Prisma vs Drizzle, etc.) | **Decision required** |
| PG jobs vs Redis/BullMQ | Proposed: PG jobs MVP |
| Org / staff context resolution mechanism | Decision required |
| **Hosting** | **Decision required** |
| **PostgreSQL RLS** | **Decision required** |
| **MFA implementation details** | **Decision required** |
| Subdomains later | Future consideration |
| Split Admin/Ops apps later | Future consideration |---

*When architecture decisions are approved, update this file and align Database + Security docs. Do not invent provider API capabilities in adapters.*
