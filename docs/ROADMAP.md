# ShipBoy Implementation Roadmap

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md) · [API](./API.md)

**Status:** Dependency-ordered roadmap for a **modular monolith**. Avoid premature microservices.

Items that cannot proceed without unresolved stack choices are marked **Architecture decision required**.

---

## Guiding constraints

- ParcelOS first; FreightOS and full DocsOS follow.
- Portals: Customer + thin Admin first → Ops later → Forwarder with FreightOS.
- Single web app, path-prefix portals, shared backend modules.
- Carrier/marketplace adapters; no invented provider capabilities.
- Billing deferred; notifications minimal; international = core workflow + verified integrations only.

### Cross-cutting Architecture decision required (blocks or shapes early phases)

| Decision | Affects |
|----------|---------|
| NestJS vs Fastify (or alternative backend) | Phases 1–2+ |
| ORM | Phases 1–2+ |
| Hosting | Phase 1 deploy targets; later production |
| PostgreSQL RLS | Defense-in-depth design (app isolation mandatory regardless) |
| MFA implementation details | Thin Admin privileged actions (Phase 10+); requirement is Confirmed |

---

## Phase 1 — Repository / tooling foundation

### Objective

Make the monorepo implementable: toolchain, app shells, local dependencies, CI skeleton—without product features.

### Major deliverables

- Workspace layout under `apps/` / `packages/` (API, worker, web)
- Package manager lockfile, lint/format/typecheck scripts
- Local Postgres (+ S3-compatible local storage **Proposed**)
- Migration tooling wired (**Architecture decision required:** ORM)
- Env templates without secrets; `.gitignore`
- Basic CI (install, typecheck, test stub)
- Shared config packages as needed

### Dependencies

- Founder/tech choices where marked **Architecture decision required** (backend framework, ORM, hosting target for later deploys)

### Acceptance criteria

- Empty modular monolith boots locally (API health endpoint, web shell, worker process stub).
- Migrations can be applied to local DB.
- CI runs on PR without product feature tests yet.

### Explicitly deferred

- Production hardening, multi-region, Kubernetes
- Full observability stack
- Real carrier/marketplace credentials

---

## Phase 2 — Authentication + tenancy + RBAC

### Objective

Secure multi-tenant identity foundation for Customer Portal and thin Admin later.

### Major deliverables

- Users, sessions/auth flows
- Organizations with `organization_type` (`exporter` \| `forwarder` \| `both`)
- Memberships with roles `owner` \| `admin` \| `ops` \| `viewer`
- Platform staff memberships with roles `platform_admin` \| `platform_ops` \| `platform_support`
- Hybrid permissions: DB role assignments + code permission definitions
- Active org context resolution
- Audit log writer (mutations + BR-A3 sensitive reads)
- Idempotency record table/helpers
- Customer Portal auth shell (`/app/...` illustrative)
- Basic notification abstraction + essential auth transactional emails

### Dependencies

- Phase 1
- **Architecture decision required:** MFA implementation details (staff step-up); auth library/session store; ORM/framework

### Acceptance criteria

- Cross-tenant IDOR tests fail closed.
- Multi-org user can switch org context; roles are org-scoped.
- Staff without permission cannot read tenant data.
- Portal URL alone does not grant API access.
- Permission codes exist in code; assignments load from DB.

### Explicitly deferred

- Full MFA UX polish beyond required privileged step-up design
- Employee/Ops Portal UI
- Forwarder Portal UI
- SSO/SAML
- Billing

---

## Phase 3 — ParcelOS core domain

### Objective

Manual order lifecycle inside an exporter organization (no marketplace yet).

### Major deliverables

- Order + order item domain/API
- Address/contact models
- Order list/filter/detail in Customer Portal
- Domain services with tenant predicates
- Minimal document attach on order (**optional early**, or wait for Phase 13 lite)

### Dependencies

- Phase 2

### Acceptance criteria

- Org members can CRUD orders per RBAC.
- Other org cannot access orders.
- Audit on important order mutations.

### Explicitly deferred

- Marketplace import
- Shipments/carriers
- Bulk fulfill

---

## Phase 4 — Products / SKUs + saved product information

### Objective

Authoritative product memory and suggestion UX without silent overwrite.

### Major deliverables

- Product/SKU entities (split vs collapse still refinable at schema time)
- Authoritative weight/dims/packaging/HSN fields
- Link order items to SKUs
- Suggestion from product + historical shipments
- Explicit “save to product” action

### Dependencies

- Phase 3

### Acceptance criteria

- Shipment/order edits do not silently change product authority.
- Suggestions appear; user confirmation remains in flow.
- Unique (`organization_id`, `sku_code`).

### Explicitly deferred

- Marketplace catalog sync as source of truth
- Inventory/WMS quantities

---

## Phase 5 — Shipment abstraction

### Objective

Carrier-agnostic shipment model and port interfaces before real carriers.

### Major deliverables

- Shipment, shipment items, package model (as needed)
- Carrier / carrier_service / org connection entities
- `CarrierPort` interface + capability flags
- State transition rules (finalize names as implemented)
- Idempotent create API contract
- Stub/fake carrier adapter for tests

### Dependencies

- Phases 3–4

### Acceptance criteria

- Can create a shipment against stub adapter with Idempotency-Key replay safety.
- UI/API only offers stub-declared capabilities.
- Core domain has no Delhivery/India Post HTTP embedded.

### Explicitly deferred

- Live Delhivery/India Post
- Labels/tracking UI polish
- Bulk create

---

## Phase 6 — Delhivery integration

### Objective

First real domestic carrier adapter (capability-honest).

### Major deliverables

- Delhivery adapter implementing supported port methods only
- Org Delhivery connection + encrypted credentials
- Map ShipBoy shipment ↔ Delhivery payloads for verified operations
- Error mapping, timeouts, webhook/poll ingest as applicable
- Customer Portal: select Delhivery service when serviceable

### Dependencies

- Phase 5
- Delhivery API access/credentials for staging
- Verification of which methods Delhivery actually supports for ShipBoy’s use cases

### Acceptance criteria

- End-to-end create shipment (and any other verified methods) against Delhivery test/sandbox as available.
- Unsupported methods are not exposed.
- Webhook/poll path idempotent if used.
- Secrets never logged or committed.

### Explicitly deferred

- India Post (Phase 7/14 as scheduled below—see Phase 7 note)
- Rate shopping across many carriers
- Every Delhivery product/service line

**Note:** India Post may land adjacent to or immediately after Delhivery once Delhivery path proves the adapter pattern; treat India Post as Phase 6b or early Phase 14 if capacity constrained—still before claiming “all domestic carriers.”

---

## Phase 7 — International shipping foundation

### Objective

Support the **core international shipment workflow** in ParcelOS, limited to verified adapter capabilities.

### Major deliverables

- International ship-to / customs-related fields needed for core flow (not CHA product)
- Adapter capability flags for international services where verified (Delhivery and/or India Post as applicable)
- Validation UX for required international fields
- Docs hooks for commercial invoice attach (lite)

### Dependencies

- Phase 5; ideally Phase 6 (or India Post adapter) with verified international methods
- Confirmation of which international services are actually available via implemented adapters

### Acceptance criteria

- User can complete an international shipment through a verified service path.
- Unverified country/carrier/service combinations are not offered as if supported.
- Product authority rules still hold.

### Explicitly deferred

- Exhaustive country/carrier matrix
- Full customs brokerage automation
- Every international product of every carrier

---

## Phase 8 — Etsy integration

### Objective

Marketplace adapter + assisted/one-click order import for Etsy.

### Major deliverables

- Marketplace port + Etsy adapter (only verified Etsy capabilities)
- Marketplace connection connect/disconnect
- Import job → orders/items with external ids
- Dedupe on connection + external order id
- Customer Portal connection settings + import UX
- Autofill suggestions from products/history on imported orders

### Dependencies

- Phases 2–4 (auth, orders, products)
- Etsy developer app / permissions as required by Etsy

### Acceptance criteria

- Import creates orders without duplicates on retry.
- Credentials encrypted; never exposed to browser.
- No claimed features beyond verified Etsy permissions/APIs.
- Fulfillment push to Etsy deferred unless verified and explicitly scoped.

### Explicitly deferred

- Amazon SP-API
- Shopify
- Marketplace fulfillment/tracking sync (unless separately verified later)

---

## Phase 9 — Bulk fulfillment + labels + tracking

### Objective

Operational speed: multi-order fulfill, labels, centralized tracking.

### Major deliverables

- Bulk shipment create (partial success **Proposed**)
- Label generate/download where adapter supports
- Tracking event ingest + shipment timeline UI
- Centralized shipment list/detail filters
- India Post adapter if not completed in Phase 6b (domestic + any verified international)

### Dependencies

- Phases 5–8 (shipments, at least one real carrier, orders to fulfill)

### Acceptance criteria

- User selects N orders and receives per-order success/failure results.
- Labels available when capability exists.
- Tracking updates appear without duplicate side effects on webhook replay.
- Idempotent bulk/create paths.

### Explicitly deferred

- Advanced automation rules/presets
- Multi-carrier rate shop UX maturity
- Public tracking pages

---

## Phase 10 — Thin Admin portal

### Objective

Strongly protected internal Admin surface for operating the platform.

### Major deliverables

- `/admin/...` (illustrative) portal module in the single web app
- Organization / user management
- Platform staff management (`platform_admin`, `platform_ops`, `platform_support`)
- Carrier / integration configuration
- Basic system health
- Audit log access
- Basic support / operational lookup (permission-scoped)
- MFA/step-up enforcement on privileged actions

### Dependencies

- Phase 2 (staff RBAC, audit)
- Phases 5–6 useful for integration config
- **Architecture decision required:** MFA implementation details

### Acceptance criteria

- Customer `owner` cannot call admin APIs.
- Staff without permission cannot perform admin actions; `/admin` URL insufficient.
- Thin Admin scope matches BR-R12; sensitive reads audited per BR-A3.
- No advanced analytics, billing admin, or complex ops tooling shipped in this phase.

### Explicitly deferred

- Full Employee/Ops Portal (post–thin Admin; separate phase when needed)
- Advanced analytics, billing administration, complex operational tooling
- Feature-flag mega-console beyond basics

---

## Phase 11 — FreightOS foundation

### Objective

Domain model and APIs for RFQ → eligibility → claim (max 10) → quotation → booking, without requiring full marketplace liquidity on day one.

### Major deliverables

- RFQ, claim, quotation, booking entities/APIs
- Eligibility service (controlled; start simple filters **Proposed**)
- Concurrent claim enforcement (≤10; one per forwarder)
- Exporter RFQ UI in Customer Portal
- Audit on publish/claim/book
- Cross-org authz via relationships + default-deny documents

### Dependencies

- Phase 2 (tenancy, RBAC)
- Phase 10 helpful for ops visibility but not strictly blocking
- Forwarder org types ready (`forwarder` / `both`)

### Acceptance criteria

- Concurrent claim tests: 100 parallel attempts → ≤10 successes.
- Duplicate forwarder claim rejected.
- Eligible discovery only—not world-readable RFQs.
- Exporter sees quotes for own RFQs only; forwarder does not see unrelated exporter tenant data.

### Explicitly deferred

- Payments / booking protection
- Ratings, disputes, rich messaging
- Anti-circumvention product suite beyond principles

---

## Phase 12 — Forwarder portal

### Objective

Role-specific Forwarder UX on FreightOS.

### Major deliverables

- `/forwarder/...` portal module
- Forwarder profile / eligibility info
- Discover eligible RFQs, claim, quote, manage bookings
- Org/team management for forwarder orgs
- Authorized document access via `resource_access_grants` only

### Dependencies

- Phase 11
- Phase 2 auth/RBAC

### Acceptance criteria

- Forwarder cannot see ineligible RFQs or peer forwarder quotes inappropriately.
- Same permission server-side enforcement as Customer Portal.
- Communication lite if included; otherwise deferred cleanly.

### Explicitly deferred

- Full messaging suite, ratings UI, dispute workflows
- Payment escrow UX

---

## Phase 13 — DocsOS expansion

### Objective

Move from minimal attachments to broader document types and associations.

### Major deliverables

- Document type taxonomy (commercial invoice, packing list, POD, etc.)
- Associations across orders, shipments, RFQs, quotations, bookings
- Explicit resource grants UX
- Stronger access controls and audited private-doc reads

### Dependencies

- Earlier lite attach (from Parcel/Freight phases)
- Phases 9 and 11–12 for association targets

### Acceptance criteria

- Default deny cross-org; grant works for specific resources.
- Signed URLs only after authz.
- BR-A3 auditing for private document privileged reads.

### Explicitly deferred

- Full document generation/OCR product
- CHA/customs filing automation

---

## Phase 14 — Additional integrations

### Objective

Expand adapters without rewriting core domain.

### Major deliverables (ordered by need, not all parallel)

- India Post completion if still pending
- Additional domestic/international carriers behind `CarrierPort`
- Amazon Seller via SP-API (**permission-bounded only**)
- Shopify marketplace adapter
- Marketplace fulfillment/tracking sync where supported
- Richer webhooks/monitoring for integrations

### Dependencies

- Stable ports from Phases 5 and 8
- Provider approvals/apps

### Acceptance criteria

- New carrier/marketplace = new adapter + config; core order/shipment unchanged.
- Amazon UI never claims unsupported SP-API actions.
- Capability flags drive API/UI.

### Explicitly deferred

- Every global carrier
- Non-verified provider features

---

## Phase 15 — Billing and advanced platform capabilities

### Objective

Add commercial platform billing and advanced ops after core logistics value is proven.

### Major deliverables

- Subscription / plan models (architecture was kept billing-ready)
- Customer billing UI; Admin billing administration
- Payment provider integration (idempotent webhooks)
- Advanced analytics
- Full Employee/Ops Portal (if not delivered earlier as a dedicated phase)
- Broader notification channels on top of the basic abstraction
- Hardening: RLS decision implementation if chosen, deeper observability

### Dependencies

- Stable ParcelOS usage (Phases 9–10)
- **Architecture decision required** items as needed (hosting scale, MFA maturity, RLS, PSP choice)

### Acceptance criteria

- No card data stored on ShipBoy servers.
- Billing webhooks idempotent.
- Advanced Admin features no longer blocked by “thin Admin” constraint.
- Existing tenant isolation and audit rules preserved.

### Explicitly deferred from earlier phases (now in scope here)

- Subscription billing and automated platform billing
- Advanced analytics and complex operational tooling
- Large notification platform

---

## Suggested sequencing diagram

```text
1 Tooling
  → 2 Auth/Tenancy/RBAC
    → 3 Orders
      → 4 Products/SKUs
        → 5 Shipment abstraction
          → 6 Delhivery (+ India Post as 6b)
            → 7 International foundation
              → 8 Etsy
                → 9 Bulk + labels + tracking
                  → 10 Thin Admin
                    → 11 FreightOS foundation
                      → 12 Forwarder portal
                        → 13 DocsOS expansion
                          → 14 More integrations
                            → 15 Billing + advanced platform
```

Ops Portal may insert after Phase 10 when support load requires it (confirmed sequencing: after Customer + thin Admin; before or parallel to FreightOS as capacity allows).

---

## Non-goals for the roadmap era

- Microservices-first delivery
- Building all portals in Phase 2–9
- Claiming unverified carrier/marketplace capabilities
- Full WMS/ERP/CHA replacement
- Hostage anti-circumvention over product value

---

*Update this roadmap when phase scope changes. Do not silently expand MVP phases.*
