# ShipBoy Database Model

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [Product](./PRODUCT.md) · [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Security](./SECURITY.md)

**Status:** Logical data model for design. **No SQL migrations in this document.** Field lists are indicative; exact types finalized at migration time.

**Stack assumption:** PostgreSQL ([Architecture](./ARCHITECTURE.md) — Proposed).

---

## 1. Modeling principles

1. **Tenant ownership:** Nearly all business tables include owning `organization_id` (UUID FK → `organizations`). Default queries filter by it ([BR-T1](./BUSINESS_RULES.md)).
2. **One user identity:** Do **not** create separate user tables per portal. Portals are UX; identity is global; access is via memberships and/or platform staff privileges.
3. **Roles are contextual:** Membership roles are per-organization; platform privileges are separate. A user may be exporter-member in Org A and forwarder-member in Org B.
4. **External IDs:** Store `provider` + `external_id` (and connection id where needed) for marketplaces and carriers; enforce uniqueness to prevent duplicates.
5. **Idempotency:** Dedicated tables for idempotency keys and webhook receipts.
6. **Concurrency:** RFQ claims use constraints that make “max 10 claims” and “one claim per forwarder” enforceable under concurrency—not only application checks.
7. **Authorized cross-org access:** Model via FreightOS relationship tables (`rfq_claims`, `quotations`, `bookings`)—not by relaxing `organization_id` filters globally. Optional grant tables only if needed beyond relationships.
8. **Suggestions vs authority:** Shipment package measurements are facts of a shipment; product measurements are authoritative product fields ([BR-P2](./BUSINESS_RULES.md)–[BR-P3](./BUSINESS_RULES.md)).
9. **Soft deletes:** **Proposed** `deleted_at` on major entities; audit remains.
10. **Timestamps:** `created_at`, `updated_at` on mutable entities.
11. **Money:** Store currency code + integer minor units **or** `numeric`—**Decision required**; be consistent.
12. **Weights/dims:** Store canonical units (e.g., grams, millimeters) **Proposed**; convert at edges.

---

## 2. Platform entities

### 2.1 `organizations`

| | |
|--|--|
| **Purpose** | Tenant root for customers/exporters and forwarders |
| **Important fields** | `id`, `name`, `slug` (unique), `organization_type` (**Confirmed:** `exporter` / `forwarder` / `both`), `status` (`active`/`suspended`—Proposed), `timezone`, `default_currency`, billing fields later |
| **Relationships** | Has many memberships, orders, products, shipments, documents, connections; may have `forwarder_profiles` |
| **Tenant ownership** | Is the tenant |
| **Constraints** | Unique `slug` |
| **Indexes** | Unique(`slug`); index(`status`); index(`organization_type`) |

**Note:** Platform/CLICKBITS itself is **not** modeled as a normal customer org for employee access. Staff use platform privilege tables (§2.5). **Decision required** if a special internal org is also needed for tooling.

### 2.2 `users`

| | |
|--|--|
| **Purpose** | Global identity shared across **all portals** |
| **Important fields** | `id`, `email` (unique), `password_hash` (nullable if SSO later), `name`, `email_verified_at`, `status` |
| **Relationships** | Has many organization_memberships; optional platform_staff_profile |
| **Tenant ownership** | Global; access via memberships and/or staff privileges |
| **Constraints** | Unique `email` |
| **Indexes** | Unique(`email`) |

Do **not** create `customer_users`, `forwarder_users`, `admin_users` tables. Portal differences are authorization + UX, not separate identities.

### 2.3 `organization_memberships`

| | |
|--|--|
| **Purpose** | User ↔ Organization with **org-scoped** role |
| **Important fields** | `id`, `organization_id`, `user_id`, `role`, `status` (`active`/`invited`/`revoked`—Proposed), `invited_by`, `accepted_at` |
| **Relationships** | Belongs to organization, user |
| **Tenant ownership** | `organization_id` |
| **Constraints** | Unique(`organization_id`, `user_id`) |
| **Indexes** | (`user_id`); (`organization_id`, `role`) |

Same user may have one membership in an exporter org and another in a forwarder org—roles are **not** globally fixed ([BR-T3](./BUSINESS_RULES.md)).

**Note:** Organization role names and platform staff role names are **Confirmed** ([Business Rules](./BUSINESS_RULES.md) BR-R4).

### 2.4 Roles / permissions (Confirmed — hybrid)

**Confirmed hybrid model:**

1. **Permission definitions** live in **application code** (canonical codes, descriptions, enforcement points)—source of truth for the permission vocabulary.
2. **Roles** and **role↔permission assignments** are stored in the **database** so they can be administered and audited.

#### Suggested tables (logical)

| Table | Purpose |
|-------|---------|
| `org_roles` | Seeded organization roles: `owner`, `admin`, `ops`, `viewer` |
| `org_role_permissions` | Assignments of permission codes to org roles (may vary by org type) |
| `organization_memberships.role_id` (or `role` FK) | Membership → org role |
| `platform_roles` | Seeded staff roles: `platform_admin`, `platform_ops`, `platform_support` |
| `platform_role_permissions` | Assignments of permission codes to staff roles |
| `platform_staff_memberships` | User → platform staff membership / role(s) |

Do **not** invent permission codes only in DB without a matching code definition. Do **not** store “portal” as a permission substitute.

**Proposed (optional later):** membership-level permission overrides—avoid early unless needed.

### 2.5 Platform staff membership (Confirmed)

Separate from organization memberships so employees are not “members of every customer.”

#### `platform_staff_memberships` (name TBD; Confirmed concept)

| | |
|--|--|
| **Purpose** | Separate staff membership linking a `users` row to platform staff status/roles |
| **Important fields** | `id`, `user_id`, `status`, `created_at`, … |
| **Constraints** | Staff access requires active staff membership + role permissions |
| **Security** | Privileged actions require MFA/step-up; **MFA implementation details Decision required** |

#### `platform_roles` / role assignments

| | |
|--|--|
| **Purpose** | Staff privileges for Admin / Ops / Support |
| **Confirmed role codes** | `platform_admin`, `platform_ops`, `platform_support` |
| **Notes** | Permission codes assigned in DB; definitions in code |

**Confirmed principle:** No staff permission ⇒ no customer-data access. Sensitive/privileged reads audited per BR-A3 (not ordinary list/detail reads).

### 2.6 Session / auth artifacts (Proposed)

Session store or token tables as required by chosen auth library—single identity works for all portals. May store `last_portal` / `active_organization_id` as UX hints only.
---

## 3. ParcelOS — catalog

### 3.1 `products`

| | |
|--|--|
| **Purpose** | Authoritative product memory |
| **Important fields** | `id`, `organization_id`, `name`, `description` (optional), `hsn_code` (nullable), `status` |
| **Relationships** | Has many SKUs (or product is SKU—see below) |
| **Tenant ownership** | `organization_id` |
| **Constraints** | — |
| **Indexes** | (`organization_id`, `name`); (`organization_id`, `hsn_code`) |

### 3.2 `skus`

| | |
|--|--|
| **Purpose** | Sellable/stock-keeping identity with package defaults |
| **Important fields** | `id`, `organization_id`, `product_id`, `sku_code`, `title`, `weight_g`, `length_mm`, `width_mm`, `height_mm`, `packaging_config` (JSONB—Proposed), `hsn_code` (nullable override), `marketplace_refs` (JSONB—Proposed) |
| **Relationships** | Belongs to product; referenced by order items |
| **Tenant ownership** | `organization_id` (denormalized for isolation queries) |
| **Constraints** | Unique(`organization_id`, `sku_code`) |
| **Indexes** | Unique(`organization_id`, `sku_code`); (`organization_id`, `product_id`) |

**Decision required:** Whether MVP collapses Product and SKU into one table. If collapsed, keep authoritative measurement fields on that single entity.

---

## 4. ParcelOS — orders

### 4.1 `marketplace_connections`

| | |
|--|--|
| **Purpose** | Org’s connection to a sales channel |
| **Important fields** | `id`, `organization_id`, `provider` (`etsy`/`amazon`/`shopify`/…), `status`, `external_shop_id`, `display_name`, encrypted credential refs, `scopes`, `last_sync_at` |
| **Relationships** | Has many marketplace orders / imported orders |
| **Tenant ownership** | `organization_id` |
| **Constraints** | Unique(`organization_id`, `provider`, `external_shop_id`) **Proposed** |
| **Indexes** | (`organization_id`, `provider`) |

Secrets: store ciphertext or secret-manager reference—not plaintext ([Security](./SECURITY.md)).

### 4.2 `orders`

| | |
|--|--|
| **Purpose** | Unified order regardless of channel |
| **Important fields** | `id`, `organization_id`, `source` (`manual`/`marketplace`), `marketplace_connection_id` (nullable), `external_order_id` (nullable), `status`, `currency`, buyer contact fields, ship-to address fields (structured), `ordered_at`, `raw_snapshot` (JSONB—Proposed, PII-aware), `idempotency` metadata |
| **Relationships** | Has many order_items; has many shipments (M:N or 1:N—see below) |
| **Tenant ownership** | `organization_id` |
| **Constraints** | Unique(`marketplace_connection_id`, `external_order_id`) where external id present (**Proposed**; Confirmed intent to prevent duplicate imports) |
| **Indexes** | (`organization_id`, `status`, `ordered_at` DESC); (`organization_id`, `external_order_id`) |

### 4.3 `order_items`

| | |
|--|--|
| **Purpose** | Line items |
| **Important fields** | `id`, `organization_id`, `order_id`, `sku_id` (nullable), `title`, `sku_code` (denormalized), `quantity`, `unit_price`, `external_line_id`, weight/dim overrides for this line (nullable) |
| **Relationships** | Belongs to order; optional SKU |
| **Tenant ownership** | `organization_id` |
| **Constraints** | FK order same org |
| **Indexes** | (`order_id`); (`organization_id`, `sku_id`) |

### 4.4 `packages`

| | |
|--|--|
| **Purpose** | Physical package to ship (may group items) |
| **Important fields** | `id`, `organization_id`, `order_id` (nullable if multi-order later), `shipment_id` (nullable until assigned), `weight_g`, dims, `packaging_type`, sequence |
| **Relationships** | Belongs to order and/or shipment; has package_items **Proposed** |
| **Tenant ownership** | `organization_id` |
| **Constraints** | — |
| **Indexes** | (`shipment_id`); (`order_id`) |

**Proposed:** MVP may treat one package per shipment and skip a separate `packages` table initially—**Decision required**. Logical model retains packages for international/multi-piece.

---

## 5. ParcelOS — shipments and carriers

### 5.1 `carriers`

| | |
|--|--|
| **Purpose** | Platform-level carrier registry |
| **Important fields** | `id`, `code` (`delhivery`, `india_post`, …), `name`, `status`, capability flags |
| **Relationships** | Has many carrier_services; org connections |
| **Tenant ownership** | Global reference data |
| **Constraints** | Unique(`code`) |

### 5.2 `carrier_services`

| | |
|--|--|
| **Purpose** | Services offered by a carrier (e.g., surface/express) |
| **Important fields** | `id`, `carrier_id`, `code`, `name`, `mode` (domestic/international—Proposed), capability metadata |
| **Constraints** | Unique(`carrier_id`, `code`) |

### 5.3 `organization_carrier_connections`

| | |
|--|--|
| **Purpose** | Org credentials/config for a carrier |
| **Important fields** | `id`, `organization_id`, `carrier_id`, `status`, encrypted credentials, account numbers, label preferences |
| **Constraints** | Unique(`organization_id`, `carrier_id`) **Proposed** (or multiple accounts—Decision required) |

### 5.4 `shipments`

| | |
|--|--|
| **Purpose** | Shipment / consignment record |
| **Important fields** | `id`, `organization_id`, `carrier_id`, `carrier_service_id`, `carrier_connection_id`, `status`, `tracking_number`, `carrier_shipment_ref`, ship-from / ship-to snapshots, `label_object_key`, `rate_amount`/`currency`, `created_by_user_id`, `idempotency_key` (nullable), `metadata` JSONB |
| **Relationships** | Links to orders via `shipment_orders` or `shipment_items`; has tracking_events |
| **Tenant ownership** | `organization_id` |
| **Constraints** | Unique(`carrier_id`, `carrier_shipment_ref`) where ref not null—**Proposed**; Unique(`organization_id`, `idempotency_key`) where key not null—**Proposed** |
| **Indexes** | (`organization_id`, `status`, `created_at` DESC); (`organization_id`, `tracking_number`); (`carrier_id`, `tracking_number`) |

### 5.5 `shipment_orders` (join)

| | |
|--|--|
| **Purpose** | Support bulk / multi-order shipments if needed |
| **Important fields** | `shipment_id`, `order_id`, `organization_id` |
| **Constraints** | Unique(`shipment_id`, `order_id`); both FKs same org |

**Proposed MVP simplification:** `shipments.order_id` single FK; join table when multi-order packages required.

### 5.6 `shipment_items`

| | |
|--|--|
| **Purpose** | What was shipped |
| **Important fields** | `id`, `organization_id`, `shipment_id`, `order_item_id`, `sku_id`, `quantity`, weight/dim as shipped |
| **Indexes** | (`shipment_id`) |

**Important:** Weight/dims here do **not** update `skus` automatically ([BR-P3](./BUSINESS_RULES.md)).

### 5.7 `tracking_events`

| | |
|--|--|
| **Purpose** | Timeline of tracking updates |
| **Important fields** | `id`, `organization_id`, `shipment_id`, `status_code`, `description`, `event_time`, `location`, `raw_payload` (JSONB—Proposed), `source` (`webhook`/`poll`) |
| **Constraints** | **Proposed** uniqueness on (`shipment_id`, `event_time`, `status_code`, `description` hash) or provider event id when available |
| **Indexes** | (`shipment_id`, `event_time`) |

---

## 6. Marketplace order staging (optional)

### 6.1 `marketplace_orders`

| | |
|--|--|
| **Purpose** | Raw/normalized import buffer before/alongside `orders` |
| **Status** | **Proposed**—may merge into `orders.raw_snapshot` for MVP |
| **Fields** | `organization_id`, `marketplace_connection_id`, `external_order_id`, `payload`, `import_status`, `order_id` |

---

## 7. FreightOS entities

*(Implement tables when near FreightOS build; design constraints now.)*

### 7.1 `forwarder_profiles`

| | |
|--|--|
| **Purpose** | Forwarder organization profile / eligibility attributes |
| **Important fields** | `id`, `organization_id` (forwarder’s tenant), lanes/modes/FCL-LCL/service areas (tables or JSONB), performance metrics, ratings aggregates |
| **Tenant ownership** | Forwarder’s `organization_id` |

**Decision required:** Whether exporters and forwarders share the same `organizations` model with a `type` flag—**Resolved:** yes, with Confirmed `organization_type` ∈ {`exporter`, `forwarder`, `both`} + optional `forwarder_profiles`.

### 7.2 `rfqs`

| | |
|--|--|
| **Purpose** | Exporter freight request |
| **Important fields** | `id`, `organization_id` (exporter), `status`, origin/destination, cargo type, weight, dimensions, package count, mode (`air`/`sea`), `load_type` (`fcl`/`lcl`), `incoterm`, `cargo_value`, `currency`, pickup details, `claims_count`, `max_claims` (default 10), `opens_at`, `closes_at` |
| **Constraints** | `max_claims` = 10 for v1 policy; `claims_count` BETWEEN 0 AND `max_claims` |
| **Indexes** | (`status`, `opens_at`); (`organization_id`, `created_at` DESC) |

### 7.3 `rfq_claims`

| | |
|--|--|
| **Purpose** | Successful claim by a forwarder on an RFQ; **primary authorized cross-org relationship** for claim access |
| **Important fields** | `id`, `rfq_id`, `forwarder_organization_id`, `claimed_at`, `status`, `claimed_by_user_id` (**Proposed**) |
| **Tenant note** | RFQ owned by exporter org; claim references forwarder org—cross-org by design within FreightOS rules |
| **Access implication** | Presence of an active claim (plus permissions) authorizes the forwarder org to access that RFQ’s allowed fields—not the exporter’s entire tenant |
| **Constraints (Confirmed requirements)** | 1. **Unique(`rfq_id`, `forwarder_organization_id`)** — one claim per forwarder per RFQ ([BR-F6](./BUSINESS_RULES.md)) 2. **At most 10 claims per RFQ** — enforce with one of the patterns in §10 |
| **Indexes** | (`forwarder_organization_id`, `claimed_at`); (`rfq_id`) |

### 7.4 `quotations`

| | |
|--|--|
| **Purpose** | Forwarder quote against a claim/RFQ; authorizes exporter read of that quote |
| **Important fields** | `id`, `rfq_id`, `rfq_claim_id`, `forwarder_organization_id`, `exporter_organization_id`, `status`, amount breakdown JSONB, currency, validity, transit estimates, terms, `version` |
| **Constraints** | **Proposed:** Unique active quote per claim; revisions increment `version` |
| **Indexes** | (`rfq_id`); (`exporter_organization_id`, `status`) |

### 7.5 `bookings`

| | |
|--|--|
| **Purpose** | Accepted commercial booking; authorizes both parties’ access to booking-scoped data |
| **Important fields** | `id`, `rfq_id`, `quotation_id`, exporter/forwarder org ids, `status`, amounts, payment fields later |
| **Constraints** | **Proposed:** One booking per RFQ when booked |

### 7.6 Cross-org access modeling notes

**Confirmed approach:**

1. Derive authorization from **relationship rows** (`rfq_claims`, `quotations`, `bookings`) for RFQ/quote/booking fields.
2. **RFQ discovery:** eligible marketplace visibility with controlled eligibility—not world-readable.
3. **Documents and similar resources:** **default deny** across orgs; grant only via explicit **resource-based access**.

### 7.7 `resource_access_grants` (Confirmed for cross-org resources such as documents)

| | |
|--|--|
| **Purpose** | Explicit, resource-scoped cross-organization (or cross-principal) access; default deny without a grant |
| **Important fields** | `id`, `resource_type` (e.g. `document`), `resource_id`, `grantee_type` (`organization`/`user`), `grantee_id`, `permission` (code from app definitions, e.g. `document.read`), `granted_by_user_id`, `grantor_organization_id`, `created_at`, `expires_at` (nullable), `revoked_at` (nullable) |
| **Constraints** | Unique active grant per (`resource_type`, `resource_id`, `grantee_type`, `grantee_id`, `permission`) **Proposed**; indexes on resource and grantee |
| **Tenant note** | Does not transfer ownership; owning org remains on the resource row |

### 7.8 Messaging / ratings / disputes

**Future consideration** tables: `threads`, `messages`, `ratings`, `disputes`. Reserve naming; do not over-build in ParcelOS MVP migrations.
---

## 8. DocsOS

### 8.1 `documents`

| | |
|--|--|
| **Purpose** | Logistics document metadata |
| **Important fields** | `id`, `organization_id`, `doc_type`, `title`, `storage_key`, `content_type`, `byte_size`, `checksum`, `created_by_user_id`, `status` |
| **Tenant ownership** | `organization_id` |
| **Indexes** | (`organization_id`, `doc_type`, `created_at` DESC) |

### 8.2 `document_links`

| | |
|--|--|
| **Purpose** | Associate document to domain entities |
| **Important fields** | `id`, `organization_id`, `document_id`, `entity_type` (`order`/`shipment`/`rfq`/`quotation`/`booking`/…), `entity_id` |
| **Constraints** | Unique(`document_id`, `entity_type`, `entity_id`) **Proposed** |
| **Indexes** | (`organization_id`, `entity_type`, `entity_id`) |

Access always re-checked against document’s owning `organization_id`, user permissions, **or** an explicit active `resource_access_grants` row ([BR-D4](./BUSINESS_RULES.md)). Default deny for cross-org.

---

## 9. Platform reliability tables

### 9.1 `audit_logs`

| | |
|--|--|
| **Purpose** | Append-only business audit, including privileged and cross-org actions |
| **Important fields** | `id`, `organization_id` (nullable for pure platform events; set to affected tenant when staff accesses tenant data—**Proposed**), `actor_user_id`, `actor_type` (`member`/`staff`—**Proposed**), `portal` (**Proposed** UX context), `action`, `entity_type`, `entity_id`, `metadata` JSONB, `ip`, `user_agent`, `created_at` |
| **Constraints** | No updates from app roles **Proposed** |
| **Indexes** | (`organization_id`, `created_at` DESC); (`entity_type`, `entity_id`); (`actor_user_id`, `created_at` DESC) |

### 9.2 `webhook_events`

| | |
|--|--|
| **Purpose** | Idempotent inbound webhook receipt |
| **Important fields** | `id`, `provider`, `delivery_id` (or hash of payload+headers), `organization_id` (nullable until resolved), `signature_valid`, `payload`, `received_at`, `processed_at`, `status`, `error` |
| **Constraints** | Unique(`provider`, `delivery_id`) |
| **Indexes** | (`status`, `received_at`) |

### 9.3 `idempotency_records`

| | |
|--|--|
| **Purpose** | Client Idempotency-Key storage |
| **Important fields** | `id`, `organization_id`, `user_id`, `key`, `route`, `request_hash`, `response_code`, `response_body`, `created_at`, `expires_at` |
| **Constraints** | Unique(`organization_id`, `key`) **Proposed** (or include route) |
| **Indexes** | (`expires_at`) for TTL cleanup |

### 9.4 `background_jobs` (if PG-backed)

| | |
|--|--|
| **Purpose** | Job queue persistence |
| **Important fields** | Depends on chosen library (Graphile Worker uses its own schema). Ensure job payloads include `organization_id` when tenant-scoped. |
| **Status** | **Proposed** aligned with Architecture job choice |

---

## 10. Concurrent RFQ claims — enforcement patterns

**Confirmed requirement:** ≤10 successful claims; concurrency-safe; one per forwarder ([BR-F4](./BUSINESS_RULES.md)–[BR-F6](./BUSINESS_RULES.md)).

**Proposed implementation patterns (choose at FreightOS build):**

### Pattern A — Atomic counter on `rfqs` (recommended starting point)

In one transaction:

1. `SELECT … FROM rfqs WHERE id = $id FOR UPDATE`
2. Abort if `claims_count >= max_claims` or status not open
3. `INSERT INTO rfq_claims …` (unique on `rfq_id, forwarder_organization_id`)
4. `UPDATE rfqs SET claims_count = claims_count + 1` with `CHECK (claims_count <= max_claims)`

Unique constraint handles duplicate forwarder; row lock + check handles the cap under concurrency.

### Pattern B — Partial unique index / claim slots

Pre-create claim slot numbers `1..10` or use exclusion constraints. More complex; optional later.

**Tests required:** concurrent claim simulations (e.g., 100 parallel attempts → ≤10 rows).

---

## 11. Relationship overview

```text
users ─────────────────────────────────────────────┐
 ├── organization_memberships ── organizations      │
 │     (org roles; exporter / forwarder / both)     │
 ├── platform_staff_memberships ── platform roles   │
 │     (Employee/Ops + Admin; MFA/step-up)          │
org_roles / platform_roles ── role_permissions      │
     (assignments in DB; permission defs in code)   │
organizations                                       │
 ├── products ── skus                               │
 ├── marketplace_connections ── orders ── items     │
 ├── carrier_connections ── carriers / services     │
 ├── shipments ── shipment_items ── tracking_events │
 ├── documents ── document_links                     │
 ├── resource_access_grants (explicit cross-org)     │
 ├── rfqs ── rfq_claims ── quotations ── bookings    │
 │         (authorized cross-org relationships)     │
 ├── audit_logs                                      │
 ├── webhook_events                                  │
 └── idempotency_records                             │
```

Portals (Customer, Forwarder, Ops, Admin) are **not** tables.

---

## 12. Indexing & uniqueness checklist

| Concern | Approach |
|---------|----------|
| Tenant list pages | Composite indexes leading with `organization_id` |
| Marketplace dedupe | Unique(connection, external_order_id) |
| SKU identity | Unique(org, sku_code) |
| Carrier refs | Unique(carrier, carrier_shipment_ref) where present |
| Webhooks | Unique(provider, delivery_id) |
| Idempotency | Unique(org, key) |
| RFQ claims | Unique(rfq, forwarder_org) + atomic cap |
| Membership | Unique(org, user) |
| Staff | Unique user ↔ staff profile |

---

## 13. What not to do yet

- No SQL migrations in-repo from this doc alone
- No per-tenant databases for MVP
- No event-sourcing tables required for MVP
- Do not denormalize in ways that silently sync shipment dims back to SKUs
- Do not create separate user tables per portal
- Do not grant staff access by adding them as members of all customer orgs

---

## 14. Open database decisions

| Topic | Status |
|-------|--------|
| Product vs SKU table split | Decision required |
| Packages table in MVP | Decision required |
| Money storage type | Decision required |
| **PostgreSQL RLS** | **Decision required** |
| Shipment↔Order cardinality | Proposed: 1 order per shipment in MVP |
| ORM choice | **Decision required** (technical architecture phase) |

### Founder-confirmed (cumulative)

| Topic | Decision |
|-------|----------|
| Organization type | `exporter`, `forwarder`, `both` |
| Org roles | `owner`, `admin`, `ops`, `viewer` |
| Platform staff roles | `platform_admin`, `platform_ops`, `platform_support` |
| Permission storage | Hybrid: DB role/assignments; code permission definitions |
| Platform staff | Separate staff membership model |
| Cross-org documents | `resource_access_grants`; default deny |
| RFQ discovery | Eligible marketplace with controlled eligibility |
| Billing fields | May exist as future hooks; billing product deferred from MVP |---

*When migrations are authored, they must match Confirmed business rules—especially RFQ claim constraints and tenant columns.*
