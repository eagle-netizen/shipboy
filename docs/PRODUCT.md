# ShipBoy Product

**Owner:** CLICKBITS Technologies Pvt. Ltd.  
**Related docs:** [PRD](./PRD.md) · [Business Rules](./BUSINESS_RULES.md) · [Architecture](./ARCHITECTURE.md) · [Database](./DATABASE.md) · [Security](./SECURITY.md)

---

## Vision

ShipBoy is a Logistics Operating System that helps sellers and exporters move goods—parcels and freight—with less manual work, fewer errors, and clearer operational control.

ShipBoy aims to become the operating layer between sales channels, carriers, freight partners, and logistics documents—not merely another label-printing tool.

ShipBoy is **not a single generic dashboard**. It is a **multi-portal platform**: distinct role-specific interfaces for customers/exporters, freight forwarders, ShipBoy employees/operations, and platform administrators—each exposing only the workflows and information relevant to that user’s role and organization. See [Multi-portal product shape](#multi-portal-product-shape).

## Positioning

> Automating parcel shipping. Simplifying cargo logistics.

ShipBoy sits at the intersection of:

- e-commerce order fulfillment
- domestic and international parcel shipping
- freight procurement between exporters and forwarders
- logistics document management

## Major user groups

ShipBoy serves distinct user groups through different portals. A person’s **role is not globally fixed**: the same user identity may be an exporter-org member in one context and a forwarder-org member in another, where architecture supports multi-org membership ([Business Rules](./BUSINESS_RULES.md), [Security](./SECURITY.md)).

| User group | Who | Primary portal | Notes |
|------------|-----|----------------|-------|
| **Customers / exporters / sellers** | Indian SME exporters and e-commerce sellers (Etsy, Amazon, Shopify, etc.) and their team members | Customer / Exporter Portal | Primary ParcelOS experience; also creates FreightOS RFQs when live |
| **Freight forwarders** | Forwarder organizations participating in FreightOS | Forwarder Portal | Claims, quotes, bookings; see only authorized RFQs/data |
| **ShipBoy employees / operations** | Internal CLICKBITS staff supporting customers | Employee / Operations Portal | Separate from customer UX; **no automatic unrestricted customer-data access** |
| **Platform administrators** | Authorized ShipBoy admins | Admin Portal | Platform config, security, org/user/employee management; strongly protected |
| **Future roles** | Finance, support specialists, warehouse/fulfillment, enterprise, logistics partners | Future portals / specialized UIs | **Future consideration**—do not redesign the platform for each |

### Customer / exporter profile (initial commercial focus)

Indian exporters and e-commerce sellers, especially small and medium-sized sellers who:

- sell on Etsy, Amazon, Shopify, and similar channels
- ship domestically and internationally from India
- currently juggle carrier portals, spreadsheets, marketplace dashboards, and export paperwork
- feel friction around address entry, product dimensions/HSN, label creation, tracking sync, and customs documentation

Founder experience with Etsy international orders, India Post, Shiprocket, customs issues, and export documentation informs product priorities. That experience is **not** treated as proof that any external provider capability is available; integrations remain subject to each provider’s actual APIs, permissions, and terms.

## Problem being solved

Sellers and exporters typically face:

1. **Fragmented order intake** — orders live in marketplaces; shipping happens elsewhere.
2. **Repetitive data entry** — addresses, weights, dimensions, HSN, and packaging are re-typed order after order.
3. **Carrier tooling sprawl** — different portals for Delhivery, India Post, and future carriers.
4. **Weak product memory** — SKU/package knowledge is tribal or buried in past shipments.
5. **Bulk fulfillment friction** — fulfilling many orders one-by-one is slow and error-prone.
6. **Tracking and marketplace sync gaps** — shipment status does not reliably flow back to sales channels when supported.
7. **Opaque freight procurement** — exporters struggle to request, compare, and book forwarder quotes safely.
8. **Document chaos** — invoices, packing lists, PODs, and customs docs are scattered across email and drives.

ShipBoy addresses these by unifying order/shipment operations (ParcelOS), structured freight RFQ/quote/booking (FreightOS), and shared logistics documents (DocsOS).

## Product philosophy

- **Seller time first** — reduce clicks and re-entry for every shipment.
- **Role-specific portals, not one mega-UI** — each portal shows only relevant workflows; frontend route access is never authorization.
- **Server-enforced permissions** — portal selection and backend authorization are separate; the backend enforces real boundaries. See [Security](./SECURITY.md).
- **Authoritative data over silent guesses** — historical shipments may suggest values; they must never silently overwrite stored product truth. See [Business Rules](./BUSINESS_RULES.md).
- **Adapters over lock-in** — carriers and marketplaces plug in behind stable interfaces.
- **Honest integrations** — never claim a marketplace or carrier can do something ShipBoy has not verified (especially Amazon SP-API).
- **Multi-tenant by default** — organization isolation is a product requirement, not an afterthought.
- **Explicit cross-org access where FreightOS requires it** — authorized relationships (RFQ → claim → quote → booking), never unrestricted peeking.
- **Auditable operations** — important business actions—and privileged employee/admin access—leave a trail.
- **Value before captivity** — anti-circumvention should protect platform integrity without making legitimate users unable to leave.
- **Modular monolith first** — ship ParcelOS quickly; keep boundaries clean for later separation. See [Architecture](./ARCHITECTURE.md).
- **No premature scale theater** — design for future scale without unnecessary microservices or infra.

## Multi-portal product shape

ShipBoy’s product surface is organized by **portal** (interface/context), **role** (responsibilities within a context), and **permission** (specific authorized actions). Detail: [PRD](./PRD.md), [Architecture](./ARCHITECTURE.md).

### 1. Customer / Exporter Portal

Used by businesses, exporters, and sellers. This is the **primary ParcelOS user experience**.

Capabilities include: dashboard; orders; products/SKUs; shipments; bulk fulfillment; carrier/service selection; labels; tracking; marketplace connections; shipping documents; FreightOS RFQs, forwarder quotations, and bookings (when FreightOS is live); organization/team management; settings. Billing UI when billing is implemented later (billing deferred from initial MVP).

### 2. Employee / Operations Portal

Used by ShipBoy internal employees and operations staff. **Must be separate from the customer experience.**

Potential capabilities: customer/account lookup; shipment/order investigation; support operations; shipment exception handling; integration monitoring; webhook/event inspection; RFQ/booking operational support; document/support workflows; customer issue resolution; audit/event visibility according to **employee permissions**.

Employees must **not** automatically have unrestricted access to customer data. Access is explicitly permission-based and auditable.

### 3. Admin Portal

Used by authorized ShipBoy administrators. **Must not be equivalent to ordinary customer access** and must be strongly protected.

**Thin Admin MVP (Confirmed):** organization/user management; platform staff management; carrier/integration configuration; basic system health; audit log access; basic support/operational lookup.

**Out of Admin MVP:** advanced analytics; billing administration; complex operational tooling.

Staff roles: `platform_admin`, `platform_ops`, `platform_support`.

### 4. Forwarder Portal

Used by freight forwarders on FreightOS.

Capabilities should include: forwarder organization profile; eligibility information; available RFQs; RFQ claiming; quotation submission and management; booking management; communication related to RFQs/bookings; shipment/document information they are **authorized** to access; performance/rating information; organization/team management.

Forwarders must only see RFQs and information they are authorized to see.

### 5. Future role-specific portals

Architecture should allow additional portals or specialized interfaces later without redesigning the whole platform (finance, support, warehouse/fulfillment, enterprise, logistics partners, other operational roles). These remain **Future consideration** unless explicitly brought into MVP.

### Portal vs role vs permission (product terms)

| Concept | Meaning |
|---------|---------|
| **Portal** | The application/interface/context through which a user works (e.g., Customer Portal). |
| **Role** | Permissions and responsibilities assigned to a user in a given organization or platform context. |
| **Permission** | A specific authorized action or access scope. |

Reaching a frontend route does **not** grant access. Authorization is enforced server-side.

## The three systems

### ParcelOS

ParcelOS automates parcel shipping for e-commerce sellers and exporters.

**Role:** order import → product/SKU memory → carrier selection → shipment & label → tracking → marketplace sync (where supported).

**Initial product focus.** ParcelOS is the first system to implement and sell.

Planned capability themes (detail in [PRD](./PRD.md)):

- Marketplace order import (Etsy first; Amazon Seller and Shopify planned)
- Auto-suggest from prior orders/products (address, contacts, dimensions, weight, HSN)
- Domestic shipping (initially Delhivery and India Post)
- Core international shipment workflow (verified/implemented carrier capabilities only in MVP)
- Unified order management across channels
- Product/SKU memory (name, weight, dimensions, packaging, HSN where applicable)
- Bulk shipment creation
- Carrier/service selection, shipment creation, labels, tracking
- Centralized shipment management
- Carrier adapter architecture for future carriers

### FreightOS

FreightOS is a freight procurement marketplace connecting exporters with freight forwarders.

**Role:** RFQ → eligibility → claim (max 10, concurrency-safe) → quotation → compare → book → messaging / ratings / disputes (as scoped).

FreightOS is **not** the initial implementation priority, but its concepts (RFQ claims concurrency, forwarder eligibility, bookings) must be designed into the overall platform model early enough to avoid painful rewrites. See [Database](./DATABASE.md) and [Business Rules](./BUSINESS_RULES.md).

### DocsOS

DocsOS provides shared logistics document management.

**Role:** store and associate commercial invoices, packing lists, shipping/customs documents, bills, invoices, PODs, freight quotations, and related logistics records with organizations, orders, shipments, RFQs, quotations, or bookings as appropriate.

DocsOS is a **shared capability layer** used by ParcelOS and FreightOS, not a standalone MVP product.

## Relationship between the three systems

```text
                    ┌─────────────────────────────────────────────────┐
                    │                    ShipBoy                       │
                    │     tenants · identity · RBAC · audit · files    │
                    └─────────────────────────────────────────────────┘
           ┌─────────────┬─────────────┬─────────────┬────────────────┐
           │ Customer /  │ Forwarder   │ Employee /  │ Admin Portal   │
           │ Exporter    │ Portal      │ Ops Portal  │                │
           │ Portal      │             │             │                │
           └──────┬──────┴──────┬──────┴──────┬──────┴───────┬────────┘
                  │             │             │              │
                  └─────────────┴──────┬──────┴──────────────┘
                                       │ shared API / domain modules
                       ┌───────────────┼───────────────┐
              ┌────────┴───┐   ┌──────┴──────┐  ┌───┴────┐
              │  ParcelOS  │   │  FreightOS  │  │ DocsOS │
              │  orders &  │   │ RFQ / quote │  │ docs & │
              │ shipments  │   │ / booking   │  │ files  │
              └────────────┘   └─────────────┘  └────────┘
```

- **ParcelOS** owns parcel order and shipment workflows (primarily Customer Portal).
- **FreightOS** owns freight RFQ, claim, quotation, and booking workflows (Customer + Forwarder portals; Employee/Admin support views as permitted).
- **DocsOS** owns document entities and access; other systems attach documents to their domain objects.
- Shared platform concerns: organizations (tenants), users/memberships, RBAC, portal-aware auth context, audit logs, webhooks/idempotency, notifications, object storage, observability.
- Multiple portals share the **same backend domain modules**; they differ in UX and in which permissions are typically exercised—not in duplicated business logic.

Documents may link across domains (e.g., a commercial invoice for a ParcelOS shipment; a freight quotation PDF for a FreightOS quote). Ownership and access still respect tenant boundaries, except where FreightOS defines **explicit authorized cross-organization** access ([Business Rules](./BUSINESS_RULES.md)).

## Initial product focus

**Build and validate ParcelOS first**, oriented to Indian SME sellers/exporters via the **Customer / Exporter Portal**:

1. Organization/tenant foundation, auth, RBAC (portal ≠ permission)
2. Customer Portal shell (not a generic all-roles dashboard)
3. Core order and product/SKU memory
4. Etsy-oriented order import workflow (subject to Etsy’s actual APIs/permissions)
5. Domestic carrier adapters for Delhivery and India Post (subject to each carrier’s actual capabilities)
6. Shipment creation, labels, tracking, centralized shipment management
7. Suggestion UX from historical data without silent overwrite
8. Bulk shipment creation for selected orders

**Confirmed MVP portal sequencing:**

1. **Customer / Exporter Portal + thin Admin Portal** first  
2. **Employee / Operations Portal** later  
3. **Forwarder Portal** when FreightOS implementation starts  

See [PRD](./PRD.md) and [Architecture](./ARCHITECTURE.md).

Marketplace sync, Amazon, Shopify, broader international coverage beyond verified integrations, FreightOS marketplace liquidity, subscription billing, rich notification platforms, and full DocsOS product surfaces follow after ParcelOS MVP is solid—see [PRD](./PRD.md) for scope cuts.

## Long-term product direction

- Multi-portal maturity: Forwarder, Employee/Ops, Admin, then future role-specific interfaces
- Broader marketplace coverage (Amazon Seller within SP-API limits, Shopify, others as justified)
- More domestic and international carrier adapters
- Richer international/customs-assisted parcel workflows
- FreightOS live marketplace with eligibility, capped concurrent claims, quoting, booking, messaging, ratings, disputes, and booking/payment protection
- DocsOS as a first-class document hub for export/import operations
- Stronger analytics, notifications, and operational automation
- API-first surfaces for integration-friendly customers where appropriate

## Explicit non-goals

ShipBoy does **not**, in the near term, aim to be:

- A single generic dashboard for all roles
- A product that treats frontend route access as authorization
- A general-purpose WMS or inventory ERP
- A courier company or asset-based carrier
- A full customs brokerage or licensed CHA replacement
- A consumer shipping app for individuals (B2C retail shipping)
- A microservices platform from day one
- A system that silently mutates authoritative product data from past shipments
- A platform that claims marketplace/carrier features beyond verified provider capabilities
- A product whose primary strategy is trapping users rather than delivering operational value
- A system where employees have standing unrestricted access to all customer data
---

*Product decisions that affect engineering must be reflected in PRD, Business Rules, Architecture, Database, and Security. Do not silently change business rules.*
