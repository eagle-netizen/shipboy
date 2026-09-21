# ShipBoy Engineering Instructions

## Product

ShipBoy is a Logistics Operating System built by CLICKBITS Technologies Pvt. Ltd.

Core systems:

- ParcelOS
- FreightOS
- DocsOS

Read the relevant files in `docs/` before making architectural or business-logic changes.

## Engineering Principles

- Keep the architecture modular.
- Avoid unnecessary microservices.
- Preserve multi-tenant isolation.
- Never bypass authorization checks.
- Never expose secrets or API keys.
- Never commit secrets to Git.
- Prefer existing utilities, services and components over creating duplicates.
- Do not duplicate business logic between frontend and backend.
- Keep marketplace integrations separate from core order logic.
- Keep carrier integrations behind carrier adapters.
- Use idempotency for payments and webhooks.
- Important business actions must be auditable.
- Do not silently change business rules.
- Do not remove existing functionality unless explicitly requested.

## Database

- Never modify the production database directly.
- Database schema changes must use migrations.
- Explain destructive migrations before applying them.
- Preserve tenant isolation in every query involving tenant-owned data.

## API

- Reuse existing API patterns.
- Validate external input.
- Authenticate and authorize every protected operation.
- Make webhook handlers idempotent.
- Handle retries safely.

## Testing

After meaningful code changes:

- Run type checking.
- Run relevant tests.
- Run linting when configured.
- Report failures instead of hiding them.

## Change Discipline

For small requests:
- Make the smallest reasonable change.
- Do not rewrite unrelated code.
- Do not introduce unnecessary dependencies.

For larger requests:
- Inspect the existing implementation first.
- Create a plan before making broad changes.
- Identify affected files.
- Explain architectural consequences.

## Production Safety

- Never deploy directly to production without review.
- Never change production configuration casually.
- Never delete production data.
- Never rotate or expose credentials through source code.
- Review database and payment changes carefully.

## Documentation

When an architectural or business rule changes, update the appropriate file under `docs/`.

Important documents:

- `docs/PRD.md`
- `docs/ARCHITECTURE.md`
- `docs/DATABASE.md`
- `docs/SECURITY.md`
- `docs/BUSINESS_RULES.md`
- `docs/API.md`
- `docs/ROADMAP.md`
