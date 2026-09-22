# ShipBoy Product Requirements Document (PRD)

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md)

**Status:** Draft for founder review. Items marked **Proposed**, **Decision required**, or **Future consideration** are not final.

---

## 1. Product goals

1. Reduce time-to-ship for Indian SME sellers/exporters by importing orders and reusing product/package knowledge.
2. Deliver ShipBoy as a **multi-portal** product—not a single generic dashboard—with role-specific interfaces for customers, forwarders, employees, and admins ([Product](./PRODUCT.md)).
3. Provide the Customer / Exporter Portal as the primary operational surface for domestic (and later international) parcel shipment creation, labels, and tracking.
4. Integrate carriers behind adapters so Delhivery, India Post, and future carriers do not rewrite core domain logic.
5. Integrate marketplaces behind adapters; treat Amazon capabilities as strictly limited to SP-API permissions.
6. Establish a multi-tenant, auditable, secure platform foundation reusable by FreightOS and DocsOS, with **server-side** RBAC independent of portal UI.
7. Design FreightOS RFQ claim semantics (max 10 concurrent-safe claims) and **authorized cross-organization** access into the platform model before that marketplace launches.
8. Keep DocsOS as shared document association capability, not a separate MVP product.
9. Keep Employee/Ops and Admin portals separate from customer UX; privileged access must be permission-based and auditable.

## 2. User personas and portals

| Persona | Description | Primary portal | Primary system |
|---------|-------------|----------------|----------------|
| **Seller / Exporter Owner** | SME selling on Etsy/Amazon/Shopify; ships from India; cares about speed, cost, correctness | Customer / Exporter | ParcelOS (+ FreightOS later) |
| **Customer Ops Staff** | Team member who fulfills orders, prints labels, handles exceptions **within the customer org** | Customer / Exporter | ParcelOS |
| **Freight Exporter** | Needs air/sea quotes via RFQs; compares forwarders | Customer / Exporter | FreightOS |
| **Freight Forwarder** | Claims RFQs (within caps), submits quotes, wins bookings | Forwarder | FreightOS |
| **ShipBoy Employee / Ops** | Internal support and operational investigation | Employee / Operations | Platform support over Parcel/Freight |
| **Platform Admin** (CLICKBITS) | Platform configuration, security, org/employee management | Admin | Platform |

Roles are **organization- or platform-context scoped**, not globally fixed. Example: one user may be an exporter-org member in one context and a forwarder-org member in another ([Business Rules](./BUSINESS_RULES.md)).

**Confirmed:** Dual role catalogs with fixed names:

- Organization: `owner`, `admin`, `ops`, `viewer`
- Platform staff: `platform_admin`, `platform_ops`, `platform_support`

Permission storage is **hybrid** (DB assignments, code definitions).

## 2.1 Portal vs role vs permission (requirements)

| Term | Definition | Enforcement |
|------|------------|-------------|
| **Portal** | Application/interface/context (Customer, Forwarder, Ops, Admin) | UX routing and navigation only |
| **Role** | Bundle of responsibilities in an org or platform staff context | Mapped to permissions server-side |
| **Permission** | Specific authorized action or access scope | **Must** be enforced by the backend |

A user must **not** gain access merely because they can reach a frontend route. Frontend portal restrictions are UX; authorization is server-side ([Security](./SECURITY.md)).

Authorization context should consider: user identity, organization, membership, role, permissions, resource ownership, portal/context, organization relationships (e.g., RFQ participants), and platform-level privileges.
## 3. Core user problems

| # | Problem | Desired outcome |
|---|---------|-----------------|
| P1 | Orders trapped in marketplaces | One-click / assisted import into ShipBoy |
| P2 | Re-entering addresses, phones, dims, weight, HSN | Suggestions from products and prior shipments; user confirms |
| P3 | Multiple carrier portals | Unified shipment creation via adapters |
| P4 | No SKU/package memory | Authoritative product records reusable on future orders |
| P5 | One-by-one fulfillment | Select many orders → bulk create shipments |
| P6 | Tracking scattered | Centralized shipment + tracking events |
| P7 | Marketplace not updated | Sync fulfillment/tracking **where supported** |
| P8 | Freight quote chaos | Structured RFQ → capped claims → quotes → book |
| P9 | Documents scattered | Associate docs to domain objects (DocsOS) |

## 4. MVP scope

### 4.1 In scope (ParcelOS MVP — Customer Portal first)

Confirmed MVP intent (implementation detail may phase within MVP):

| Capability | Notes |
|------------|-------|
| Organization (tenant) + membership + RBAC | Required foundation; roles not globally fixed |
| Auth (session/token) + active org + portal context | See [Security](./SECURITY.md); portal ≠ authz |
| **Customer / Exporter Portal** shell | Primary ParcelOS UX—not a generic all-roles dashboard |
| Manual order create/edit | Needed even before marketplace import is live |
| Product/SKU memory | Name, weight, dimensions, packaging config, HSN where applicable |
| Suggestion from history | Never silent overwrite of authoritative product fields |
| Marketplace connection model | Adapter-ready; Etsy import is initial channel focus |
| Etsy order import | **Subject to Etsy API/permissions**; one-click/assisted import |
| Address/contact autofill options | From prior orders / saved data with user confirmation |
| Domestic shipping | Initial carriers: **Delhivery**, **India Post** via adapters |
| **International shipment workflow (core)** | Supported in MVP; only **verified/implemented** carrier-international capabilities—not every country/service |
| Carrier/service selection | Based on adapter-reported capabilities |
| Shipment creation | Single and bulk (multi-order select) |
| Label generation | Where carrier adapter supports it |
| Tracking ingestion/display | Where carrier supports it |
| Centralized shipment list/detail | Filters, status, carrier refs |
| Minimal org/team settings in Customer Portal | Invite/manage members with roles `owner`/`admin`/`ops`/`viewer` |
| **Thin Admin Portal** | See §4.3; MFA/step-up required (impl Decision required) |
| Basic notification abstraction + essential transactional notifications | Not a full notification platform |
| Audit log for important actions | Incl. BR-A3 sensitive-read allowlist |
| Idempotent webhooks / critical writes | See Business Rules |
| Document attachment (minimal DocsOS) | Attach files to order/shipment; cross-org access default deny + resource grants when needed |
| Billing-ready architecture hooks | **No** subscription/automated platform billing in initial MVP |

### 4.2 Explicitly deferred from MVP (still product intent)

| Capability | Status |
|------------|--------|
| Full Employee / Operations Portal | **Later** (confirmed sequencing) |
| Full Admin beyond thin Admin | Phased (no advanced analytics / billing admin / complex ops tooling in Admin MVP) |
| Forwarder Portal | **When FreightOS starts** (confirmed sequencing) |
| Subscription / automated platform billing | **Deferred** (architecture remains billing-ready) |
| Large notification platform / multi-channel marketing | Out of MVP |
| Finance / warehouse / enterprise specialized portals | Future consideration |
| Amazon Seller integration | Future; SP-API constrained |
| Shopify integration | Future |
| Exhaustive international country/carrier/service coverage | Out of MVP (core workflow yes; verified integrations only) |
| Marketplace fulfillment/tracking sync | Future where supported |
| FreightOS live RFQ marketplace | Future (model reserved; eligible discovery when live) |
| Forwarder ratings, disputes, payment protection | Future FreightOS |
| Rich DocsOS document types & workflows | Future |
| Multi-warehouse / full WMS | Non-goal near term |
| Customer billing UI | Deferred with billing product |

## 4.3 Portal-specific requirements

### Customer / Exporter Portal (Confirmed — first wave with thin Admin)

Must support (as features land): dashboard; orders; products/SKUs; shipments (domestic + core international via verified adapters); bulk fulfillment; carrier/service selection; labels; tracking; marketplace connections; shipping documents; org/team management; settings; FreightOS RFQs, quotations comparison, and bookings when FreightOS is live. Billing UI deferred with billing product.

### Employee / Operations Portal (Confirmed surface; **later** sequencing)

Must remain separate from customer experience. Capabilities may include customer/account lookup; shipment/order investigation; support ops; exception handling; integration monitoring; webhook/event inspection; RFQ/booking operational support; document/support workflows; issue resolution; audit/event visibility **according to employee permissions**.

**Confirmed:** Employees do not automatically receive unrestricted customer-data access. Sensitive/privileged reads are audited (not every read).

### Admin Portal (Confirmed — **thin Admin** in first wave)

Thin Admin MVP (**Confirmed**):

- organization / user management
- platform staff management
- carrier / integration configuration
- basic system health
- audit log access
- basic support / operational lookup

**Out of Admin MVP:** advanced analytics, billing administration, complex operational tooling.

**Confirmed:** Admin access is not equivalent to ordinary customer access; MFA/step-up for privileged actions (**implementation Decision required**). Staff roles: `platform_admin`, `platform_ops`, `platform_support`.

### Forwarder Portal (Confirmed for FreightOS; starts **with FreightOS**)

Must support forwarder profile; eligibility; **eligible** RFQ marketplace discovery; claiming; quotation submit/manage; booking management; related communication; authorized shipment/document views via explicit grants; performance/ratings; org/team management.

### Future portals

Finance, support-specialist, warehouse/fulfillment, enterprise, logistics partners: **Future consideration**—architecture must allow adding portals without full redesign ([Architecture](./ARCHITECTURE.md)).
## 5. ParcelOS MVP workflow

Happy path (conceptual):

1. User creates/joins an **Organization**.
2. User connects a marketplace (initially Etsy) **or** creates orders manually.
3. Orders appear in unified order list with channel metadata.
4. For each order (or bulk selection):
   - System suggests address/phone from order or prior shipments.
   - System suggests weight/dims/HSN/packaging from linked Product/SKU or prior shipment history.
   - User confirms or edits; product authoritative fields update **only** on explicit user save to product, not from shipment alone.
5. User selects carrier + service (Delhivery / India Post initially).
6. System calls carrier adapter (`getRates` / `getServiceability` / `createShipment` as available).
7. Labels generated when supported; tracking ID stored.
8. Shipment appears in centralized management; tracking events update over time.
9. (**Future**) Sync tracking/fulfillment back to marketplace when adapter supports it.

## 6. Future ParcelOS capabilities

- Amazon Seller (explicitly SP-API permission-bounded)
- Shopify and additional marketplaces
- Additional domestic/international carriers via adapters (expand verified coverage over time)
- Deeper international customs field assistance (not a CHA replacement)
- `schedulePickup`, richer cancel/void flows where carriers allow
- Advanced bulk rules / presets
- Rate shopping across multiple adapters
- Broader notification channels after the basic abstraction exists

## 7. FreightOS scope

### 7.1 Product intent (confirmed)

- Exporter creates RFQ with origin, destination, cargo type, weight, dimensions, package count, air/sea, FCL/LCL, incoterm, cargo value, pickup details, and other logistics requirements.
- Eligibility may consider lane, cargo type, mode, FCL/LCL, service area, performance, capacity, historical quote accuracy, ratings.
- Maximum **10** eligible forwarders may successfully claim an RFQ.
- Claim limit must be **concurrency-safe** (100 simultaneous attempts → at most 10 successes).
- Same forwarder cannot claim the same RFQ more than once.
- Forwarders submit quotations; exporters compare and eventually book through ShipBoy.
- Architecture should support: RFQs, eligibility, claims, quotations, booking, messaging, audit logs, ratings, disputes, payment/booking protection, anti-circumvention controls.
- Platform should provide genuine value rather than only preventing users from leaving.

### 7.2 MVP for FreightOS

**Out of ParcelOS MVP.** FreightOS implementation is a later phase. Data model and claim concurrency design should be documented now ([Database](./DATABASE.md), [Business Rules](./BUSINESS_RULES.md)) so ParcelOS tenancy patterns do not block FreightOS.

### 7.3 Proposed FreightOS phase-1 (when started)

**Proposed:** RFQ create → eligibility → claim (cap 10) → quote submit → exporter compare → soft-book (no payments) → messaging lite.  
Ratings, disputes, payment protection: **Future consideration**.

## 8. DocsOS scope

### 8.1 Intent

Support association and storage of:

- commercial invoices
- freight quotations
- packing lists
- shipping documents
- customs documents
- bills / invoices
- PODs
- export/import documents
- logistics records

Documents associate with organizations and optionally orders, shipments, RFQs, quotations, or bookings.

### 8.2 MVP

**Minimal:** upload/attach file metadata + storage object + ACL within tenant; link to order and/or shipment.  
Full document-type taxonomy and generation workflows: **Future consideration**.

## 9. Functional requirements

### 9.1 Platform

| ID | Requirement |
|----|-------------|
| F-P1 | Multi-tenant organizations with strict data isolation by default |
| F-P2 | Users belong to organizations via memberships; roles are org-scoped (not globally fixed) |
| F-P3 | RBAC enforced **server-side** on protected operations; portal route ≠ authorization |
| F-P4 | Multi-portal UX: Customer, Forwarder, Employee/Ops, Admin as distinct interfaces |
| F-P5 | Same backend domain modules serve multiple portals |
| F-P6 | Audit log for important business actions and privileged/cross-org access |
| F-P7 | Idempotency for webhooks and damaging duplicate writes |
| F-P8 | Secrets never in Git; server-side secrets management |
| F-P9 | Background jobs for imports, tracking polls, webhook fan-out as needed |
| F-P10 | Structured API errors; validated inputs |
| F-P11 | Authorized FreightOS cross-organization access is explicit, permission-checked, and auditable—not treated as a tenancy bug |

### 9.2 ParcelOS

| ID | Requirement |
|----|-------------|
| F-A1 | Create/list/filter orders; channel metadata when imported |
| F-A2 | Order items linked to products/SKUs when known |
| F-A3 | Product/SKU stores name, weight, dims, packaging, HSN |
| F-A4 | Suggestions from history; no silent overwrite of product authority |
| F-A5 | Marketplace adapter interface; Etsy import first |
| F-A6 | Amazon features only if SP-API permits (when Amazon is built) |
| F-A7 | Carrier adapter interface with capability methods as supported |
| F-A8 | Initial adapters: Delhivery, India Post |
| F-A9 | Select carrier/service; create shipment; store external IDs |
| F-A10 | Generate label when adapter supports |
| F-A11 | Track shipment / ingest tracking events when supported |
| F-A12 | Bulk select orders and create shipments together |
| F-A13 | Cancel shipment when adapter supports; reflect state safely |
| F-A14 | Centralized shipment management UI/API in Customer Portal |

**Note:** Adapter methods (`createShipment`, `getRates`, `generateLabel`, `trackShipment`, `cancelShipment`, `schedulePickup`, `getServiceability`) are **capability targets**. Individual carriers may implement a subset. Do not invent provider support.

### 9.3 FreightOS (future implementation)

| ID | Requirement |
|----|-------------|
| F-F1 | Create RFQ with required logistics fields (Customer Portal) |
| F-F2 | Eligibility evaluation (rules evolve) |
| F-F3 | Claim RFQ with hard max 10 successful claims; concurrency-safe (Forwarder Portal) |
| F-F4 | Unique claim per forwarder per RFQ |
| F-F5 | Submit/update quotations per claim rules (Forwarder Portal) |
| F-F6 | Exporter compare quotations (Customer Portal) |
| F-F7 | Booking lifecycle through ShipBoy (both authorized parties) |
| F-F8 | Messaging, ratings, disputes, payment protection as phased |
| F-F9 | Cross-org access limited to RFQ/claim/quote/booking relationship—no unrestricted peer-org access |

### 9.4 DocsOS

| ID | Requirement |
|----|-------------|
| F-D1 | Store document metadata + blob reference |
| F-D2 | Associate documents to allowed domain entities |
| F-D3 | Enforce tenant-scoped access control; cross-org document access only via explicit authorization rules |

### 9.5 Portals and privileged access

| ID | Requirement |
|----|-------------|
| F-PORT1 | Customer Portal is primary ParcelOS UX |
| F-PORT2 | Employee/Ops Portal is separate from customer UX |
| F-PORT3 | Admin Portal is strongly protected and not equivalent to customer access |
| F-PORT4 | Forwarder Portal shows only authorized RFQs/data |
| F-PORT5 | Architecture allows future portals without full redesign |
| F-PORT6 | Employee customer-data access is permission-scoped and audited |
## 10. Non-functional requirements

| ID | Requirement |
|----|-------------|
| NF1 | Modular monolith initially ([Architecture](./ARCHITECTURE.md)) |
| NF2 | Secure by default; server-side authz ([Security](./SECURITY.md)) |
| NF3 | Schema changes via migrations only |
| NF4 | Observable: logs, metrics, traces (phased) |
| NF5 | Automated tests for domain rules (esp. RFQ claims, tenancy, idempotency, portal IDOR) |
| NF6 | Rate limiting on public/auth APIs |
| NF7 | Design for scale without premature microservices |
| NF8 | API-first where appropriate for integrations |
| NF9 | Maintainable domain boundaries: Parcel / Freight / Docs / Platform |
| NF10 | Avoid unnecessary UI duplication across portals while keeping portal UX strongly separated |

## 11. Important user journeys

### J1 — Etsy import to domestic label (Customer Portal — MVP target)

Seller in Customer Portal connects Etsy → imports order → confirms suggested package data → selects Delhivery or India Post → creates shipment → downloads label → tracks in ShipBoy.

### J2 — Bulk fulfill (Customer Portal)

Seller selects N imported/manual orders → reviews suggestions → chooses service → bulk create → handles per-order failures without aborting entire batch unpredictably (**Proposed:** partial success with per-item results).

### J3 — Product memory improvement (Customer Portal)

Seller corrects weight on shipment screen → optionally saves to Product/SKU (**Proposed:** explicit “Save to product” action) → future orders for that SKU suggest updated values.

### J4 — Freight RFQ claim race (Customer + Forwarder portals)

Exporter publishes RFQ in Customer Portal → **eligible** forwarders see it in Forwarder Portal (controlled eligibility marketplace) → many claim → system allows exactly ≤10 successes under concurrency → duplicates for same forwarder rejected → quotes submitted → exporter compares/books in Customer Portal → both parties see booking they are authorized to see; documents remain default-deny unless explicit resource grants exist.

### J5 — Document attach (Customer Portal)

User uploads commercial invoice → links to shipment → teammate with permission downloads; outsider/other tenant cannot (unless explicit authorized share rule applies later).

### J6 — Support investigation (Employee / Ops Portal)

Support agent with permission looks up a customer shipment → views allowed fields → resolves exception → access is audited. Agent **without** permission cannot open unrestricted tenant data. Reaching `/ops/...` alone is insufficient.

### J7 — Platform configuration (Admin Portal)

Admin enables a carrier flag or suspends an organization → change is permission-checked and audited. Admin session is not a customer-org membership shortcut.

### J8 — Multi-context user (identity)

User belongs to exporter Org A and forwarder Org B → switches organization/portal context → permissions and visible data follow the active membership; roles are not globally fixed.

## 12. Acceptance criteria (major capabilities)

### AC — Tenant isolation

- Given two customer organizations A and B, a member of A cannot read/write B’s orders, products, shipments, or documents via API or UI, except where an **explicit** authorized cross-org relationship applies (FreightOS).

### AC — Portal is not authorization

- Given a user who can manually navigate to another portal’s URL, APIs still deny actions lacking permission. Frontend hiding is insufficient.

### AC — Employee least privilege

- An employee without a specific customer-data permission cannot retrieve arbitrary tenant orders/shipments. Granted access is audited.

### AC — Admin separation

- Admin Portal privileges are distinct from customer-org roles; customer `owner` cannot perform platform admin actions.

### AC — Product authority

- Given a Product with weight W, creating a shipment with weight W′ does not change Product weight unless the user explicitly saves to the product.
- Historical shipment may appear as a suggestion only.

### AC — Carrier adapter boundary

- Adding a new carrier requires a new adapter implementing the carrier port; core order/shipment domain does not embed carrier-specific HTTP details.

### AC — Marketplace adapter boundary

- Marketplace import/sync logic lives behind marketplace adapters; core Order model remains channel-agnostic aside from external IDs/metadata.

### AC — Bulk shipment

- User can select multiple eligible orders and request shipment creation; system returns per-order success/failure (**Proposed** partial success model).

### AC — Webhook idempotency

- Replaying the same provider webhook delivery does not create duplicate domain side effects.

### AC — RFQ claims (FreightOS)

- Under concurrent claim attempts, successful claims for one RFQ never exceed 10.
- The same forwarder organization cannot hold two claims on the same RFQ.

### AC — Authorized cross-organization access (FreightOS)

- A forwarder can access an RFQ only when eligibility/claim rules allow.
- An exporter can access quotations submitted for its RFQ.
- Neither party gains unrestricted access to the other’s unrelated data.
- Cross-org access paths are auditable.

### AC — Amazon honesty

- Any Amazon feature ships only within documented SP-API permissions for the app’s approved roles; unsupported actions are not exposed as if available.

### AC — Audit

- Creating a shipment, claiming an RFQ, changing roles, connecting a marketplace, and privileged employee/admin access produce audit events with actor, tenant/context, action, and timestamp.

## 13. Out of scope (explicit)

- A single generic dashboard for all roles
- Frontend-only authorization
- Standing unrestricted employee access to all customer data
- Acting as a carrier or freight forwarder ourselves
- Full inventory/WMS/ERP replacement
- Licensed customs brokerage / CHA services
- Guaranteeing rate accuracy beyond what carriers return
- Claiming marketplace capabilities not granted by the provider
- Consumer (individual) shipping product
- Direct production DB edits outside migrations
- Microservices split as a prerequisite to MVP
- Building all portals in ParcelOS MVP

## 14. Open decisions

| Topic | Status |
|-------|--------|
| Whether rates are always fetched live vs cached | Proposed: live with short cache where safe |
| Bulk partial-success semantics | Proposed: partial success |
| Notification provider vendor | Decision required (technical phase) |
| **NestJS vs Fastify** | **Decision required** (technical architecture phase) |
| **ORM** | **Decision required** (technical architecture phase) |
| **Hosting** | **Decision required** (technical architecture phase) |
| **PostgreSQL RLS** | **Decision required** (technical architecture phase) |
| **MFA implementation details** | **Decision required** (technical architecture phase) |

### Founder-confirmed (cumulative)

| # | Decision | Choice |
|---|----------|--------|
| 1 | Role catalog | Org roles + separate platform staff roles |
| 1a | Org role names | `owner`, `admin`, `ops`, `viewer` |
| 1b | Staff role names | `platform_admin`, `platform_ops`, `platform_support` |
| 2 | Permission storage | Hybrid: roles/permissions in DB, permission definitions in code |
| 3 | Portal sequencing | Customer + thin Admin first; Ops later; Forwarder when FreightOS starts |
| 3a | Thin Admin MVP | Org/user, staff, carrier/integration config, basic health, audit logs, basic support lookup; no advanced analytics/billing admin/complex ops tooling |
| 4 | Routing | Path prefixes initially |
| 5 | Web architecture | Single web app initially |
| 6 | Organization type | `exporter`, `forwarder`, `both` |
| 7 | Platform staff | Separate staff membership + MFA/step-up (impl details Decision required) |
| 8 | RFQ discovery | Eligible marketplace visibility with controlled eligibility |
| 9 | Privileged reads | BR-A3 allowlist; not ordinary list/detail reads |
| 10 | Cross-org documents | Explicit resource-based access, default deny |
| 11 | International ParcelOS MVP | Core international workflow; verified integrations only |
| 12 | Billing | Deferred from initial MVP; architecture billing-ready |
| 13 | Notifications | Basic abstraction + essential transactional only |---

*Update this PRD when scope changes. Do not silently expand MVP.*
