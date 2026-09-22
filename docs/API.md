# ShipBoy API / Domain Contract

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md) · [Roadmap](./ROADMAP.md)

**Status:** High-level API/domain contract for a modular monolith. Not an OpenAPI spec. Framework, ORM, and hosting choices remain unresolved where marked in Architecture (**Architecture decision required**).

---

## 1. Contract principles

1. **API-first where appropriate** — portals and future integrations call the same domain API.
2. **Server-side authorization** — portal routes are UX only; every protected operation is authorized on the server ([Security](./SECURITY.md)).
3. **Tenant isolation by default** — tenant-owned resources are scoped to an organization; FreightOS cross-org access is explicit and relationship-based ([Business Rules](./BUSINESS_RULES.md)).
4. **Adapters at the edge** — carriers and marketplaces are ports/adapters; core resources stay provider-agnostic aside from external IDs and capability flags.
5. **Honest capabilities** — do not expose carrier/marketplace operations the adapter has not declared as supported. Do not invent provider features.
6. **Idempotency for damaging writes** — shipment create, bulk fulfill, webhook processing, RFQ claim, and future payments.
7. **Audit** — important mutations and sensitive/privileged reads (BR-A3 allowlist) produce audit events.
8. **Versioning** — **Proposed:** `/api/v1/...` prefix; exact transport (REST JSON) aligned with Architecture.

### 1.1 Authn / authz context on requests

Protected requests resolve:

| Context | Meaning |
|---------|---------|
| **User identity** | Authenticated principal |
| **Active organization** | Selected membership org (`exporter` / `forwarder` / `both`) when acting as org member |
| **Org role** | `owner` \| `admin` \| `ops` \| `viewer` |
| **Platform staff** | Separate staff membership + role (`platform_admin` \| `platform_ops` \| `platform_support`) when acting as staff |
| **Permissions** | Hybrid: assignments in DB, definitions in code |

Client-supplied org IDs are never trusted without membership (or scoped staff permission) checks.

### 1.2 Common cross-cutting behaviors

| Concern | Contract expectation |
|---------|----------------------|
| Validation | Reject invalid input with structured errors |
| Errors | Consistent error envelope (code, message, details)—**Proposed** shape |
| Idempotency-Key | Supported on designated write operations; replay returns original result |
| Pagination / filtering | List endpoints support cursor or offset—**Proposed** |
| Webhooks (inbound) | Signature verification + idempotent processing by provider delivery id |
| Notifications | Basic abstraction; enqueue essential transactional notifications |

---

## 2. Authentication

### Purpose

Establish and manage user identity for all portals (Customer, Forwarder, Ops, Admin) with a **single user identity**.

### Major operations

| Operation | Notes |
|-----------|-------|
| Register / invite accept | Org invite flows for members |
| Login / logout | Session-based web auth **Proposed** |
| Password reset | Time-limited tokens |
| Email verification | **Proposed** before sensitive actions |
| Session refresh / revoke | On password change / membership revoke **Proposed** |
| Step-up / MFA challenge | Required for privileged staff actions; **Architecture decision required** for MFA implementation details |
| List available contexts | Orgs + staff portals the user may enter |

### Important relationships

- User → many `organization_memberships`
- User → optional `platform_staff_memberships`
- Session → user (+ optional active org / portal hint for UX only)

### Authorization boundaries

- Unauthenticated: only public auth endpoints and verified webhooks.
- Authenticated without org context: limited to identity/context selection.
- Staff MFA/step-up: required before privileged admin/ops actions (mechanism **Architecture decision required**).

### Idempotency requirements

- Login itself need not be idempotent; password-reset token consumption must be single-use.
- Invite accept should be safe under retry (idempotent accept).

### Important external IDs

- None required for core auth. Future SSO subject IDs—**Future consideration**.

---

## 3. Organizations and memberships

### Purpose

Multi-tenant root and team access control.

### Major operations

| Operation | Notes |
|-----------|-------|
| Create organization | Sets `organization_type`: `exporter` \| `forwarder` \| `both` |
| Get / update organization | Settings, timezone, etc. |
| Suspend / reactivate org | Thin Admin / `platform_admin` |
| Invite member | Email invite to role |
| List / update / revoke memberships | Role changes audited |
| Switch active organization | Multi-org users |
| Manage platform staff memberships | Thin Admin |

### Important relationships

- Organization 1→N memberships, products, orders, shipments, documents, connections
- Organization type gates which portals are appropriate
- Forwarder orgs may have `forwarder_profiles` (FreightOS)

### Authorization boundaries

| Actor | Access |
|-------|--------|
| Org `owner` / `admin` | Manage members and org settings (per permission map) |
| Org `ops` / `viewer` | No membership admin unless granted |
| `platform_admin` | Org/user management (thin Admin) |
| Other orgs | No access |

### Idempotency requirements

- Invite create: **Proposed** idempotent by (org, email) for pending invites.
- Role change: last-write-wins with audit; not typically Idempotency-Key based.

### Important external IDs

- `organizations.slug` (unique human identifier)
- Membership unique on (`organization_id`, `user_id`)

---

## 4. Products / SKUs

### Purpose

Authoritative product/package memory (name, SKU, weight, dimensions, packaging, HSN where applicable).

### Major operations

| Operation | Notes |
|-----------|-------|
| Create / update / archive product or SKU | Authoritative fields |
| Get / list / search | By sku_code, name |
| Explicit “save shipment measurements to product” | Only on user intent—never silent overwrite from shipment |

### Important relationships

- Belongs to organization
- Referenced by order items and shipment items
- Historical shipments may **suggest** values only

### Authorization boundaries

- Org members with catalog permissions (`ops`+ typically write; `viewer` read).
- Other tenants: deny.
- Staff: only with explicit support permission; PII-light but still tenant-scoped—sensitive reads if PII attached audited per BR-A3.

### Idempotency requirements

- Create with client key **Proposed** for import jobs.
- Updates are not silently merged from shipment create.

### Important external IDs

- Unique (`organization_id`, `sku_code`)
- Optional marketplace SKU refs in metadata—adapter-defined, not invented

---

## 5. Orders

### Purpose

Unified order record across manual entry and marketplace import.

### Major operations

| Operation | Notes |
|-----------|-------|
| Create / update manual order | Customer Portal |
| List / filter / get order | Channel metadata when imported |
| Import / sync from marketplace | Via marketplace adapter; capability-honest |
| Link / unlink SKUs on line items | |
| Cancel / archive order | Policy-dependent |

### Important relationships

- Organization-owned
- Optional `marketplace_connection` + external order id
- 1→N order items; linked to shipments (typically 1:1 in early MVP **Proposed**)

### Authorization boundaries

- Default: active exporter/`both` org membership + order permissions.
- Forwarder: no access to exporter orders unless later explicit grant (not default).
- Staff support lookup: permission-scoped; sensitive reads audited.

### Idempotency requirements

- Marketplace import must not create duplicate active orders for same connection + external order id.
- Manual create: optional Idempotency-Key **Proposed**.

### Important external IDs

- (`marketplace_connection_id`, `external_order_id`) unique when present
- External line ids on items when provided by marketplace

---

## 6. Shipments

### Purpose

Carrier-facing consignment lifecycle: create, label, track, cancel (when supported).

### Major operations

| Operation | Notes |
|-----------|-------|
| Create shipment | From order(s); selects carrier service |
| Bulk create shipments | Partial success **Proposed**: per-order results |
| Get rates / serviceability | Only if adapter supports |
| Generate / download label | Only if adapter supports |
| Cancel / void | Only if adapter supports and state allows |
| Get shipment / list | Centralized management |
| Attach packages / items | Domestic and core international workflows |

### Important relationships

- Owned by organization
- References carrier, carrier_service, carrier_connection
- Links to order(s) / shipment items
- 1→N tracking events
- May link documents

### Authorization boundaries

- Org members with shipment permissions.
- Cross-org: deny by default.
- Staff: scoped support permission; audits on sensitive reads.

### Idempotency requirements

- **Confirmed need:** shipment create and bulk create accept Idempotency-Key (or equivalent) so retries do not double-create carrier shipments.
- Cancel: safe retry once terminal.

### Important external IDs

- `tracking_number`, `carrier_shipment_ref` (store when returned)
- Uniqueness **Proposed:** unique active (`carrier_id`, `carrier_shipment_ref`) where present
- Do not assume a carrier returns any particular id shape beyond what the adapter maps

### International note

Core international workflow is in MVP scope; only **verified** adapter capabilities are offered. No invented country/service matrix.

---

## 7. Carriers

### Purpose

Platform carrier registry, org carrier connections, and capability negotiation via adapters.

### Major operations

| Operation | Notes |
|-----------|-------|
| List platform carriers / services | Global reference |
| Connect / update / disable org carrier credentials | Encrypted secrets |
| List capabilities for connection | Declared by adapter |
| Admin configure carrier availability | Thin Admin |

### Important relationships

- Platform `carriers` → `carrier_services`
- Org → `organization_carrier_connections`
- Shipments use connection + service

### Authorization boundaries

- Org `admin`/`owner` (typical) manage connections; secrets never returned in plaintext.
- `platform_admin` manages platform carrier configuration (thin Admin).
- Runtime createShipment only uses connections the org owns.

### Idempotency requirements

- Connection upsert **Proposed** idempotent by (`organization_id`, `carrier_id`) if single-account model.

### Important external IDs

- Platform `carriers.code` (e.g. `delhivery`, `india_post`)
- Provider account identifiers as returned/stored by adapter—not invented

### Capability contract (adapter port)

Potential methods (subset per carrier): `createShipment`, `getRates`, `generateLabel`, `trackShipment`, `cancelShipment`, `schedulePickup`, `getServiceability`.  
UI/API must only offer what the adapter declares.

**Initial adapters:** Delhivery, India Post—exact supported methods depend on verification at integration time.

---

## 8. Tracking

### Purpose

Normalize and expose shipment tracking timeline.

### Major operations

| Operation | Notes |
|-----------|-------|
| List tracking events for shipment | |
| Ingest tracking webhook | Carrier → ShipBoy |
| Poll / refresh tracking | Job when adapter supports |

### Important relationships

- Belongs to shipment (+ organization denormalized)
- Source: `webhook` or `poll`

### Authorization boundaries

- Same as parent shipment (org member or scoped staff).
- Public tracking pages: **Future consideration**.

### Idempotency requirements

- Webhook ingest unique on (`provider`, `delivery_id`) or provider event id.
- Duplicate events must not corrupt timeline (dedupe **Proposed**).

### Important external IDs

- Provider event ids when available
- Tracking number on shipment

---

## 9. Marketplace connections

### Purpose

Connect sales channels and import orders through marketplace adapters.

### Major operations

| Operation | Notes |
|-----------|-------|
| Start connect / OAuth or credential flow | Provider-specific; capability-honest |
| Refresh tokens | Server-side |
| List / disable connection | Disable stops new imports; keeps history **Proposed** |
| Trigger import / sync | Jobs |
| Push fulfillment / tracking | **Only where adapter supports** (often post-MVP) |

### Important relationships

- Org-owned connection → imported orders
- Amazon features explicitly limited to SP-API permissions when built

### Authorization boundaries

- Org admins/owners typically manage connections; secrets never logged/returned.
- Staff may inspect connection health with permission—not raw secrets.

### Idempotency requirements

- Import jobs must dedupe by external order id.
- Webhooks from marketplaces: signature + delivery idempotency.

### Important external IDs

- (`organization_id`, `provider`, `external_shop_id`) unique **Proposed**
- Provider shop/account ids as returned by the provider

**Initial focus:** Etsy import. Amazon and Shopify planned later—do not invent endpoints or permissions.

---

## 10. RFQs (FreightOS)

### Purpose

Exporter freight request for quotation.

### Major operations

| Operation | Notes |
|-----------|-------|
| Create / update / publish / cancel / expire RFQ | Customer Portal |
| List own RFQs | Exporter org |
| List discoverable RFQs | Forwarders: **eligible marketplace** visibility only |

### Important relationships

- Owned by exporter organization
- 0..10 claims; quotations; optional booking
- Fields: origin, destination, cargo, weight, dims, packages, air/sea, FCL/LCL, incoterm, value, pickup, etc.

### Authorization boundaries

- Create/manage: exporter/`both` org + permissions.
- Discover: forwarders passing **controlled eligibility** only—not world-readable, not all forwarders.
- Staff: scoped operational support.

### Idempotency requirements

- Publish: safe retry.
- Create: optional Idempotency-Key **Proposed**.

### Important external IDs

- Internal RFQ id primary; optional exporter reference number

---

## 11. RFQ claims

### Purpose

Forwarder claim on an open RFQ (max 10, concurrency-safe, one per forwarder).

### Major operations

| Operation | Notes |
|-----------|-------|
| Claim RFQ | Forwarder Portal |
| List claims for RFQ / for forwarder | |

### Important relationships

- RFQ ↔ forwarder organization
- Enables quotation submission

### Authorization boundaries

- Claim: forwarder/`both` org + permissions + eligibility.
- Exporter can list claims on own RFQ.
- Same forwarder cannot claim twice; global cap 10 successful claims.

### Idempotency requirements

- **Confirmed:** claim must be concurrency-safe and idempotent per forwarder (unique constraint + transactional cap). Retries must not exceed 10 or duplicate the same forwarder.

### Important external IDs

- Unique (`rfq_id`, `forwarder_organization_id`)

---

## 12. Quotations

### Purpose

Forwarder commercial offer against a claim/RFQ.

### Major operations

| Operation | Notes |
|-----------|-------|
| Submit / revise quotation | Forwarder |
| List quotations for RFQ | Exporter compares |
| Withdraw quotation | Policy-dependent |

### Important relationships

- Tied to `rfq_claim` + RFQ
- May become booking
- May attach documents via resource grants

### Authorization boundaries

- Forwarder: own quotes only.
- Exporter: quotes on own RFQs only.
- No unrestricted peer-org browsing.

### Idempotency requirements

- Submit with Idempotency-Key **Proposed**; revisions versioned **Proposed**.

### Important external IDs

- Internal quote id; optional forwarder quote reference

---

## 13. Bookings

### Purpose

Accepted commercial engagement between exporter and forwarder through ShipBoy.

### Major operations

| Operation | Notes |
|-----------|-------|
| Create booking from quotation | Exporter |
| Get / list / update status | Both parties as authorized |
| Cancel | Policy-dependent |

### Important relationships

- Links RFQ, quotation, exporter org, forwarder org
- Messaging/ratings/disputes/payment protection: later phases

### Authorization boundaries

- Only participant orgs (and scoped staff).
- Documents: default deny unless `resource_access_grants`.

### Idempotency requirements

- Create booking Idempotency-Key **Proposed**; **Proposed** one booking per RFQ when booked.
- Future payments: mandatory idempotency.

### Important external IDs

- Internal booking id; payment provider refs when billing/protection exists (deferred)

---

## 14. Documents (DocsOS)

### Purpose

Store and associate logistics files with domain entities.

### Major operations

| Operation | Notes |
|-----------|-------|
| Upload / metadata create | Private object storage |
| Get download URL | Short-lived signed URL after authz |
| Link / unlink to entity | order, shipment, RFQ, quotation, booking, … |
| Grant / revoke cross-org resource access | Explicit grants; default deny |
| Soft delete | **Proposed** |

### Important relationships

- Owned by organization
- `document_links` to entities
- `resource_access_grants` for cross-org access

### Authorization boundaries

- Default: owning org + permission.
- Cross-org: **only** explicit resource-based grants.
- Staff: permission-scoped; private document reads are BR-A3 audited.

### Idempotency requirements

- Upload complete **Proposed** idempotent by checksum + client key.
- Grant create unique per active (resource, grantee, permission) **Proposed**.

### Important external IDs

- `storage_key`, checksum
- No public permanent URLs

---

## 15. Notifications

### Purpose

Basic notification abstraction for essential transactional messages (not a full notification platform).

### Major operations

| Operation | Notes |
|-----------|-------|
| Enqueue notification | Domain events / auth flows |
| List user notifications (optional in-app) | **Proposed** minimal |
| Admin/provider health | Thin / later |

### Important relationships

- User and/or organization recipients
- Outbox → worker → provider

### Authorization boundaries

- Users see own notifications.
- No cross-tenant notification reads.
- Provider credentials: server-only.

### Idempotency requirements

- Enqueue with dedupe key for essential events **Proposed** (avoid duplicate “password reset” spam on retry).

### Important external IDs

- Provider message ids when returned

**Out of MVP contract:** marketing campaigns, complex preference centers, multi-channel orchestration platforms.

---

## 16. Audit logs

### Purpose

Append-only record of important business actions and sensitive/privileged reads.

### Major operations

| Operation | Notes |
|-----------|-------|
| Append (internal) | Written by domain services |
| List / filter (Admin / scoped staff) | Thin Admin audit access |
| Export | **Future consideration** |

### Important relationships

- Actor user, optional organization, entity type/id, action, metadata (non-secret)
- **Proposed** fields: `actor_type` (`member`/`staff`), `portal`

### Authorization boundaries

- Org members: limited self-org audit **Proposed** / permission-gated.
- `platform_admin` / permitted staff: audit log access (thin Admin).
- Reading audit logs that expose PII may itself be BR-A3 sensitive.

### Idempotency requirements

- Append-only; duplicates from retried business ops should be avoided by tying to successful transactions.
- No update/delete for normal roles.

### Important external IDs

- None required; may reference entity UUIDs

### Sensitive-read allowlist (Confirmed)

Audit privileged reads involving: customer PII; private documents; financial/billing data; security/authentication settings; cross-organization resources; platform/admin data; support or impersonation access.  
Do **not** audit ordinary dashboard/list/detail reads.

---

## 17. Platform / Admin API surface (thin Admin)

Distinct from customer org APIs; requires platform staff roles.

| Area | Operations (MVP) |
|------|------------------|
| Organizations / users | Lookup, suspend, basic management |
| Platform staff | Assign `platform_admin` / `platform_ops` / `platform_support` |
| Carriers / integrations | Configure availability/connections at platform level |
| System health | Basic health/status |
| Audit logs | Access/filter |
| Support lookup | Basic operational lookup (permission-scoped) |

**Deferred from Admin MVP:** advanced analytics, billing administration, complex operational tooling.

---

## 18. Inbound webhooks (cross-cutting)

| Concern | Contract |
|---------|----------|
| Purpose | Receive carrier/marketplace async events |
| Auth | Provider signature verification—not user session |
| Idempotency | Unique (`provider`, `delivery_id`); processing must be retry-safe |
| Flow | Verify → persist → enqueue → apply domain updates |

Do not invent webhook event types; map only what verified integrations provide.

---

## 19. Explicit non-goals for this contract

- Framework-specific controllers, DTOs, or ORM models
- Claiming Delhivery/India Post/Etsy/Amazon capabilities not verified
- Public unauthenticated access to tenant documents or PII
- Billing/subscription payment APIs in initial MVP (keep extension points only)
- Microservice-per-resource splits

---

## 20. Open technical items

| Topic | Status |
|-------|--------|
| NestJS vs Fastify (or alternative) | Architecture decision required |
| ORM | Architecture decision required |
| Hosting | Architecture decision required |
| PostgreSQL RLS | Architecture decision required |
| MFA implementation details | Architecture decision required |
| Exact error/pagination envelope | Proposed |
| Rate caching policy | Proposed |

---

*When OpenAPI is introduced, it should refine—not contradict—this domain contract and the Business Rules.*
