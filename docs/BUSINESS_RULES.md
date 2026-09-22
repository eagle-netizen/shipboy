# ShipBoy Business Rules

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md)

This document defines **enforceable** business rules. Engineering must not silently change them.

Legend:

- **Confirmed** — required by current product context; implement as stated.
- **Proposed** — recommended default; needs founder confirmation before treating as law.
- **Decision required** — must be chosen before implementation of that area.
- **Future consideration** — not binding for ParcelOS MVP.

---

## 1. Tenancy and membership

### BR-T1 — Organization is the tenant boundary (Confirmed)

Every tenant-owned record belongs to exactly one **owning Organization** (or is global platform reference data). Queries and APIs for tenant-owned data must default to the active organization context. **Unrestricted** cross-tenant access is forbidden.

**Exception (Confirmed concept):** FreightOS defines **authorized cross-organization relationships** (e.g., exporter RFQ ↔ forwarder claim/quote/booking). Those paths are **not** tenancy violations when explicitly modeled, permission-checked, and auditable. They do **not** grant unrestricted access to the other organization’s data. See §14.

Employee/Admin privileged access is a separate platform-staff path ([Security](./SECURITY.md)), not a substitute for customer membership—and is never “automatic full access.”

### BR-T2 — User membership (Confirmed)

A User accesses a customer or forwarder Organization only through an active **Membership**. No membership ⇒ no tenant access via that org path.

### BR-T3 — Multi-org users (Confirmed intent; details Proposed)

A single User identity **may** hold memberships in multiple Organizations (e.g., exporter org in one context, forwarder org in another). Roles are **not globally fixed**. The user must select (or be scoped to) one organization context per request/session. **Proposed:** explicit org switcher in applicable portals.

### BR-T4 — Organization lifecycle (Decision required)

Rules for org suspension, deletion, and data export/retention on offboarding.

### BR-T5 — Organization types (Confirmed)

`organizations.organization_type` is one of: **`exporter`**, **`forwarder`**, **`both`**.

Portals available to a membership depend on org type + permissions (e.g., forwarder-type orgs use Forwarder Portal; exporter-type use Customer Portal; `both` may use either when the user has appropriate membership permissions).

Platform/CLICKBITS staff are **not** modeled as a normal customer org type for employee access—see platform staff membership ([BR-R6](./BUSINESS_RULES.md)).

---

## 2. Roles, permissions, and portals

### BR-R1 — RBAC required (Confirmed)

Protected operations require authentication **and** authorization via permissions. Portal UI must not be the sole control.

### BR-R2 — Portal ≠ role ≠ permission (Confirmed)

| Concept | Definition |
|---------|------------|
| **Portal** | Interface/context (Customer, Forwarder, Employee/Ops, Admin) |
| **Role** | Assigned responsibilities in an organization or platform-staff context |
| **Permission** | Specific authorized action or access scope |

Authorization **MUST** be enforced server-side. Reaching a frontend route does not grant access.

### BR-R3 — Authorization context (Confirmed)

Server authorization should consider, as applicable:

- user identity
- organization (active tenant context)
- membership
- role
- permissions
- resource ownership
- portal/context (for UX and telemetry—**Proposed** portal hint in session, never sole check)
- organization relationships (FreightOS participants)
- platform-level privileges (employee/admin)

### BR-R4 — Dual role catalogs (Confirmed)

ShipBoy uses **two separate role systems**:

1. **Organization roles** — assigned via `organization_memberships` (customer/exporter and forwarder orgs).
2. **Platform staff roles** — assigned via a **separate staff membership model** (Employee/Ops and Admin), not by adding staff as members of customer orgs.

#### Organization roles (Confirmed names)

| Role | Intent |
|------|--------|
| `owner` | Full control including billing (when billing exists) and destructive org actions |
| `admin` | Manage members, connections, settings (not ownership transfer / billing ownership transfer) |
| `ops` | Orders, shipments, documents day-to-day |
| `viewer` | Read-only |

These four names apply to organization memberships. Forwarder organizations use the **same role name set** (`owner`, `admin`, `ops`, `viewer`) within the forwarder org context; permissions differ by org type and assigned permission codes—not by a separate forwarder role vocabulary (**Confirmed** naming; permission matrices may still be refined).

#### Platform staff roles (Confirmed names)

| Role | Intent |
|------|--------|
| `platform_admin` | Platform administration (thin Admin and later admin capabilities) |
| `platform_ops` | Internal operational privileges |
| `platform_support` | Support / investigation privileges |

### BR-R5 — Forwarder org roles (Confirmed naming via org role set)

Forwarder actions require membership in a `forwarder` or `both` organization with an organization role (`owner` / `admin` / `ops` / `viewer`) plus appropriate permissions. No separate `forwarder_*` role name catalog.

### BR-R6 — Platform staff membership (Confirmed)

ShipBoy employees/admins use a **separate platform staff membership model**:

- must **not** automatically have unrestricted customer-data access
- receive **explicit** permissions for support/investigation/admin actions
- **MFA and/or step-up authentication** required for privileged actions ([Security](./SECURITY.md))
- generate **auditable** events for sensitive/privileged reads and for mutating privileged actions

### BR-R7 — Platform admin privileges (Confirmed)

Admin Portal privileges are distinct from customer-org roles. Customer `owner` ≠ platform admin. Admin access must be strongly protected, including MFA/step-up for privileged actions.

### BR-R8 — Portal access rules (Confirmed)

| Portal | Who may use (conceptually) |
|--------|----------------------------|
| Customer / Exporter | Active membership in an `exporter` or `both` organization + relevant permissions |
| Forwarder | Active membership in a `forwarder` or `both` organization + relevant permissions |
| Employee / Ops | Platform staff membership with employee permissions—not a customer membership alone |
| Admin | Platform staff membership with admin privileges—strongly gated |

**Proposed:** After login, route users to an allowed portal based on available memberships/privileges; still re-check on every API call.

### BR-R9 — Least privilege (Confirmed)

Default grants should be minimal; elevation is explicit.

### BR-R10 — Permission storage (Confirmed — hybrid)

**Hybrid model:**

- **Roles and role↔permission assignments** are stored in the **database** (so assignments can be administered and audited).
- **Permission definitions** (canonical permission codes, descriptions, and what each permission means in code) live in **application code** as the source of truth for the permission vocabulary.

Do not invent ad-hoc permission strings only in the DB without a matching code definition. Do not store “portal” as a permission substitute.

### BR-R11 — Portal sequencing (Confirmed)

1. **Customer / Exporter Portal + thin Admin Portal** first (ParcelOS MVP).
2. **Employee / Operations Portal** later.
3. **Forwarder Portal** when FreightOS implementation starts.

### BR-R12 — Thin Admin MVP scope (Confirmed)

Initial Admin Portal includes:

- organization / user management
- platform staff management
- carrier / integration configuration
- basic system health
- audit log access
- basic support / operational lookup

**Out of initial Admin MVP:** advanced analytics, billing administration, and complex operational tooling.

---

## 3. Orders and ownership

### BR-O1 — Order ownership (Confirmed)

Orders are owned by an Organization. Only members with appropriate permissions may create, view, update, or fulfill them.

### BR-O2 — Marketplace-imported orders (Confirmed)

Imported orders retain external marketplace identity (provider + external order id). Uniqueness: within a tenant (or within a marketplace connection—**Decision required** exact uniqueness grain), the same external order must not create duplicate active ShipBoy orders.

### BR-O3 — Unified order list (Confirmed)

Orders from all channels appear in one organizational order management surface, distinguishable by channel metadata.

### BR-O4 — Order mutability after shipment (Proposed)

After a shipment is created in a non-cancelable state, core ship-to address changes on the order require explicit handling (block, warn, or create amendment)—exact policy **Decision required**.

---

## 4. Products, SKUs, and package memory

### BR-P1 — Product/SKU ownership (Confirmed)

Products/SKUs are owned by an Organization. They store at minimum: SKU, product name, weight, dimensions, packaging configuration, and HSN where applicable.

### BR-P2 — Authoritative product data (Confirmed)

The Product/SKU record is the **authoritative** source for stored product weight, dimensions, packaging, and HSN.

### BR-P3 — Historical shipment as suggestion only (Confirmed)

Historical shipment information may be used to **suggest** address, numbers, dimensions, weight, or HSN. It must **never silently overwrite** authoritative product information.

### BR-P4 — Updating authoritative product data (Proposed)

Authoritative product fields change only when a user (or explicit import mapping job with user-visible rules) saves to the Product/SKU. Shipment creation with different dims/weight does not by itself mutate the product.

### BR-P5 — Autofill from previous orders (Confirmed)

For order import / fulfillment UX, the system may offer autofill of address, contact numbers, saved dimensions/weight/HSN from previous orders/products to save time. Autofill is assistive; user confirmation remains part of the fulfillment workflow (**Proposed:** require review before first shipment of a new SKU).

---

## 5. Shipments and carriers

### BR-S1 — Shipment ownership (Confirmed)

Shipments are owned by an Organization and typically reference one or more Orders / order items / packages.

### BR-S2 — Carrier selection (Confirmed)

Users select carrier and service from options exposed by carrier adapters for that organization. Core domain must not hardcode carrier-specific APIs.

### BR-S3 — Adapter capability subset (Confirmed)

Adapters may support a subset of: `createShipment`, `getRates`, `generateLabel`, `trackShipment`, `cancelShipment`, `schedulePickup`, `getServiceability`. UI/API must only offer actions the adapter declares as supported. Do not invent carrier capabilities.

### BR-S4 — Initial domestic carriers (Confirmed)

Initial domestic carrier adapters: **Delhivery** and **India Post**. Future carriers added via new adapters.

### BR-S5 — External shipment identity (Confirmed)

Store carrier-side IDs (AWB/tracking/shipment refs) needed for label, track, and cancel. Uniqueness constraints should prevent duplicate active shipments for the same carrier external id within relevant scope (**Proposed:** unique per carrier + external id globally or per tenant—**Decision required**).

### BR-S6 — Shipment state transitions (Proposed)

**Proposed** high-level states (final names Decision required):

`draft` → `created` → `label_generated` → `in_transit` → `delivered` → `cancelled` / `failed` / `rto` (as applicable)

Rules:

- Illegal transitions must be rejected.
- Carrier webhooks/pollers may advance tracking-derived states.
- Cancel only when adapter supports and current state allows.

### BR-S7 — Bulk shipment creation (Confirmed)

Users may select multiple orders and fulfill them together. **Proposed:** partial success—each order succeeds or fails independently; overall job reports per-item outcomes.

### BR-S8 — Rates and serviceability (Confirmed as capability)

When adapters provide `getRates` / `getServiceability`, selection UX may use them. Absence of rates from a carrier is not treated as a platform outage if that capability is unsupported.

### BR-S9 — International shipping (Confirmed MVP policy)

ParcelOS MVP **supports the core international shipment workflow** (collect/confirm international ship-to and shipment fields, create shipment via adapters, labels/tracking where supported).

Carrier and international-service coverage in MVP is **limited to integrations that are actually implemented and verified**. Do **not** attempt every country/carrier/service in MVP. ShipBoy is not a CHA substitute ([Product](./PRODUCT.md) non-goals).

---

## 6. Marketplace integrations

### BR-M1 — Marketplace adapter architecture (Confirmed)

Marketplace integrations are separate from core order logic and accessed via marketplace adapters.

### BR-M2 — Etsy initial focus (Confirmed)

Initial import focus includes Etsy one-click/assisted import. Exact Etsy API operations depend on Etsy’s platform—do not invent endpoints.

### BR-M3 — Amazon SP-API constraint (Confirmed)

Amazon functionality must be explicitly treated as subject to Amazon SP-API permissions and capabilities. Unsupported operations must not be presented as available.

### BR-M4 — Shopify and others (Confirmed as planned)

Amazon Seller and Shopify are planned marketplace integrations, post or alongside Etsy depending on roadmap (**Decision required** sequencing).

### BR-M5 — Fulfillment/tracking sync (Confirmed intent; support-gated)

Sync fulfillment/tracking to marketplaces only where the marketplace adapter supports it and credentials allow.

### BR-M6 — Credential ownership (Confirmed)

Marketplace credentials/tokens are secrets owned by the Organization connection; never logged in plaintext; never committed to Git.

### BR-M7 — Connection disable (Proposed)

Disabling a marketplace connection stops new imports/sync jobs but does not delete historical orders.

---

## 7. Idempotency and webhooks

### BR-I1 — Idempotent damaging writes (Confirmed)

Operations where duplicate execution could cause damage (payments, shipment create with external side effects, webhook processing, RFQ claim) must support idempotency.

### BR-I2 — Webhook handling (Confirmed)

Webhook handlers must be idempotent. Retries must not duplicate side effects. Store webhook event receipts keyed by provider + delivery id (or equivalent).

### BR-I3 — Client idempotency keys (Proposed)

APIs for shipment create, bulk fulfill, and payment-related calls accept an Idempotency-Key; replays return the original result.

---

## 8. Audit

### BR-A1 — Auditable important actions (Confirmed)

Important business actions must be auditable, including at minimum:

- membership/role changes
- marketplace connect/disconnect
- shipment create/cancel
- RFQ publish/claim/book (FreightOS)
- document access policy changes / sensitive deletes (**Proposed**)
- permission elevation
- **privileged employee/admin access to tenant resources** (**Confirmed**)
- **sensitive/privileged reads** per [BR-A3](#br-a3--sensitive-read-audit-allowlist-confirmed) (**Confirmed**)
- **authorized cross-organization mutating actions** (**Confirmed**); cross-org reads when they match BR-A3

Audit entries include actor, organization/context, action, timestamp, portal/context if known (**Proposed**), and relevant entity references. See [Security](./SECURITY.md) and [Database](./DATABASE.md).

### BR-A2 — Audit immutability (Proposed)

Audit logs are append-only for application roles; correction via compensating events, not silent edits.

### BR-A3 — Sensitive-read audit allowlist (Confirmed)

Audit **privileged reads** involving:

- customer PII
- private documents
- financial / billing data
- security / authentication settings
- cross-organization resources
- platform / admin data
- support or impersonation access

Do **not** audit ordinary dashboard / list / detail reads.

---

## 9. FreightOS — RFQ, claims, quotes, booking

### BR-F1 — RFQ content (Confirmed)

An RFQ may include: origin, destination, cargo type, weight, dimensions, number of packages, air/sea, FCL/LCL, incoterm, cargo value, pickup details, and other relevant logistics requirements.

### BR-F2 — RFQ lifecycle (Proposed)

**Proposed** states: `draft` → `open` → `claims_full` / `closed_for_claims` → `quoted` → `booked` → `cancelled` / `expired`

Exact transitions **Decision required** at FreightOS build time; concurrency rule BR-F4 is Confirmed regardless.

### BR-F3 — Forwarder eligibility and RFQ discovery (Confirmed)

**Discovery model:** Eligible marketplace visibility with **controlled eligibility**.

- Open RFQs are visible to forwarders who pass eligibility rules—not to the entire world and not unrestricted to all forwarders.
- Eligibility may consider: lane, cargo type, transportation mode, FCL/LCL, service area, performance, capacity, historical quote accuracy, ratings.
- Exact scoring/filter formula for v1 is still refinable (**Proposed:** start with simple lane/mode/service-area filters; richer signals later).

### BR-F4 — Maximum 10 successful claims (Confirmed)

At most **10** eligible forwarders may successfully claim a given RFQ.

### BR-F5 — Concurrency safety (Confirmed)

Claim issuance must be concurrency-safe. If 100 forwarders attempt to claim simultaneously, **exactly at most 10** successful claims are possible (never more than 10). Implementation must use transactional constraints / atomic counters / equivalent—not check-then-act without locking. See [Database](./DATABASE.md).

### BR-F6 — One claim per forwarder per RFQ (Confirmed)

The same forwarder must not claim the same RFQ more than once.

### BR-F7 — Quotations (Confirmed intent)

Forwarders submit quotations through ShipBoy. Exporters compare quotations in ShipBoy.

**Proposed:** One active quotation per claim unless revision workflow is defined; revisions supersede prior version.

### BR-F8 — Booking (Confirmed intent)

Exporters can eventually book through ShipBoy. Booking lifecycle details, payment capture, and protection mechanisms are **Future consideration** / **Decision required** at FreightOS payment phase.

### BR-F9 — Messaging, ratings, disputes (Confirmed as architectural support)

Architecture should support messaging, ratings, and disputes. Detailed rules are **Future consideration**.

### BR-F10 — Payment/booking protection (Confirmed as architectural support)

Platform should support payment/booking protection concepts. Specific escrow/payment rules are **Decision required** when payments launch; do not claim a payment product exists until specified.

### BR-F11 — Anti-circumvention principles (Confirmed)

Controls may discourage bypassing ShipBoy for introductions made on-platform, but the platform must provide **genuine value** rather than attempting to make legitimate users unable to leave. Punitive dark patterns are out of product philosophy ([Product](./PRODUCT.md)).

**Proposed examples (need confirmation):** watermarked contact sharing after booking milestones; audit of off-platform solicitation reports—not hostage data export.

### BR-F12 — Portal pairing for FreightOS (Confirmed)

Exporters perform RFQ/compare/book in the **Customer / Exporter Portal**. Forwarders perform claim/quote/booking management in the **Forwarder Portal**. Shared backend domain; separate UX.

---

## 10. Documents (DocsOS)

### BR-D1 — Document ownership (Confirmed)

Documents are owned by an Organization. Access is tenant-scoped and permission-gated.

### BR-D2 — Associations (Confirmed)

Documents may associate with organizations and, where appropriate, shipments, orders, RFQs, quotations, or bookings.

### BR-D3 — Deletion (Proposed)

Soft-delete by default for auditability; hard-delete restricted and audited.

### BR-D4 — Cross-org document access (Confirmed)

Cross-organization document access is **default deny**.

Access is granted only via **explicit resource-based access** (e.g., a grant or participant-linked authorization on a specific document/resource)—never by implying “all of counterparty’s docs” from an RFQ relationship alone.

Owning organization retains ownership; grantees receive only the scoped permission recorded on the resource access rule. See [Database](./DATABASE.md) `resource_access_grants`.

---

## 11. External integrations (general)

### BR-E1 — No invented provider capabilities (Confirmed)

Do not claim Amazon, Etsy, Delhivery, India Post, or any provider supports a feature unless verified for the integration. Adapters declare capabilities explicitly.

### BR-E2 — Failures are explicit (Confirmed)

Carrier/marketplace failures surface as structured errors; silent “success” on external failure is forbidden.

### BR-E3 — Rate limits and retries (Proposed)

Respect provider rate limits; retry with backoff on transient errors; do not retry non-idempotent external calls without idempotency protection.

---

## 12. Data retention and privacy (policy hooks)

### BR-PRIV1 — PII minimization in logs (Confirmed)

Do not log full secrets, passwords, or unnecessary PII. See [Security](./SECURITY.md).

### BR-PRIV2 — Retention (Decision required)

Retention periods for orders, tracking, documents, and audit logs require founder/legal input. Do not claim GDPR/DPDP certification in-product unless established.

---

## 12b. Billing and notifications (platform product rules)


### BR-BILL1 — Billing deferred from initial MVP (Confirmed)

Subscription billing and automated platform billing are **deferred** from the initial MVP. Architecture must remain **billing-ready** (extensible org/plan hooks, no hard-coded assumption that billing does not exist later) without implementing a billing product in MVP.

### BR-NOTIF1 — Notifications MVP (Confirmed)

MVP supports:

- a **basic notification abstraction** (so channels/providers can be added later)
- **essential transactional notifications** (e.g., auth/security and critical operational messages as needed)

Do **not** build a large notification platform initially. Channel vendor selection remains **Decision required** at implementation time.

---

## 13. Rule change control

### BR-CTRL1 — No silent rule changes (Confirmed)

Changes to confirmed business rules require documentation updates in this file and related docs, plus explicit product acknowledgment.

### BR-CTRL2 — Proposed → Confirmed (Confirmed process)

Proposed rules become Confirmed only after founder approval noted in this document or PRD.

---

## 14. Authorized cross-organization access (FreightOS)

Strict tenant isolation remains the **default** ([BR-T1](#br-t1--organization-is-the-tenant-boundary-confirmed)).

FreightOS introduces legitimate multi-party flows:

```text
Exporter Organization
        ↓
      RFQ
        ↓
Forwarder Organization
        ↓
    Quotation
        ↓
     Booking
```

### BR-X1 — Not a tenancy violation (Confirmed)

Authorized cross-organization access along RFQ → claim → quotation → booking is **not** treated as a violation of tenant isolation. It must be explicitly modeled.

### BR-X2 — Explicit paths only (Confirmed)

Every cross-organization access path must be:

1. **Explicit** in domain rules (e.g., claim grants forwarder read of that RFQ’s allowed fields)
2. **Permission-checked** server-side
3. **Auditable** for mutating actions and for reads that match [BR-A3](#br-a3--sensitive-read-audit-allowlist-confirmed)

### BR-X3 — Examples (Confirmed intent)

- A forwarder can discover/access an RFQ when **eligibility** allows (eligible marketplace visibility) and/or after claiming it.
- An exporter can access quotations submitted for its RFQ.
- Both parties can access booking information they are authorized to see.
- Neither organization gains unrestricted access to the other’s data (orders, other RFQs, unrelated documents, credentials, etc.).
- Documents follow **default deny** + explicit resource-based grants ([BR-D4](#br-d4--cross-org-document-access-confirmed)).

### BR-X4 — No implied peer access (Confirmed)

Participation in one RFQ/booking does not grant browsing rights into the counterparty’s full tenant.

### BR-X5 — Employee/Admin vs cross-org (Confirmed)

Platform staff access is **not** modeled as FreightOS cross-org participation. It uses platform privileges with separate audit rules ([BR-R6](#br-r6--platform-staff-membership-confirmed), [BR-R7](#br-r7--platform-admin-privileges-confirmed)).

---

## Appendix — Quick index

| ID | Topic | Status |
|----|-------|--------|
| BR-T1 | Tenant isolation + authorized cross-org exception | Confirmed |
| BR-T5 | Org types exporter/forwarder/both | Confirmed |
| BR-T3 | Multi-org users; roles not globally fixed | Confirmed intent |
| BR-R2 | Portal ≠ role ≠ permission; server-side authz | Confirmed |
| BR-R4 | Org roles `owner`/`admin`/`ops`/`viewer`; staff `platform_admin`/`platform_ops`/`platform_support` | Confirmed |
| BR-R6 | Staff membership + MFA/step-up for privileged actions | Confirmed |
| BR-R10 | Hybrid permission storage (DB assignments, code definitions) | Confirmed |
| BR-R11 | Portal sequencing Customer+thin Admin → Ops → Forwarder | Confirmed |
| BR-R12 | Thin Admin MVP scope | Confirmed |
| BR-P2/P3 | Authoritative product vs suggestions | Confirmed |
| BR-S2/S3 | Carrier adapters | Confirmed |
| BR-S9 | International core workflow; verified integrations only | Confirmed |
| BR-M1/M3 | Marketplace adapters / Amazon SP-API | Confirmed |
| BR-I1/I2 | Idempotency / webhooks | Confirmed |
| BR-A1 | Audit important actions | Confirmed |
| BR-A3 | Sensitive-read audit allowlist | Confirmed |
| BR-F3 | Eligible marketplace RFQ discovery | Confirmed |
| BR-F4/F5/F6 | RFQ max 10, concurrency, one claim | Confirmed |
| BR-F11 | Anti-circumvention with real value | Confirmed |
| BR-D4 | Cross-org docs: default deny + resource grants | Confirmed |
| BR-BILL1 | Billing deferred; architecture billing-ready | Confirmed |
| BR-NOTIF1 | Basic notification abstraction + essential transactional | Confirmed |
| BR-X1–X4 | Cross-org FreightOS access | Confirmed |
| BR-S6 | Shipment states | Proposed |
| BR-F2 | RFQ states | Proposed |
