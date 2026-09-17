# Architecture: Diwana.ma
**PRD Reference**: docs/prd-diwana.md
**System Design**: docs/system-design-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: Software Architect + Tech Lead

## 1. Overview
A single Next.js 15 App Router application on Vercel, backed by Neon Postgres, with duty calculation isolated as a pure domain module and tariff data maintained by a nightly ingestion cron. Chosen for one reason above all others: a solo developer must be able to operate it.

## 2. Architecture Decision Records

### ADR-1: Modular monolith, not services
- **Context**: The system has four distinct concerns — ingestion, classification, calculation, workspace CRUD.
- **Decision**: One Next.js deployment with enforced internal module boundaries: `lib/duty/`, `lib/tariff/`, `lib/classify/`, `lib/shipments/`. Cross-module imports go only through each module's `index.ts`.
- **Alternatives**: NestJS service split as in Terroir.ma — rejected: no independent scaling need at 20 RPS, and it triples operational surface for one person.
- **Consequences**: A later refactor to services stays possible because boundaries are explicit from day one. Discipline is enforced by an ESLint `no-restricted-imports` rule, not by convention.

### ADR-2: Clerk organisations as the tenancy primitive, from migration 001
- **Context**: FR-8 requires multi-user orgs with roles. Moqawil's v2 pain is retrofitting multi-tenancy onto a single-tenant schema.
- **Decision**: Multi-tenant from the first migration. Every tenant-owned table carries `org_id NOT NULL`. The Clerk `org_id` flows into a JWT claim, and every query passes through a repository that injects the `org_id` predicate.
- **Alternatives**: Single-tenant MVP, multi-tenant later — explicitly rejected. We have first-hand evidence of that cost from Moqawil.
- **Consequences**: Slightly more ceremony per query, plus a hard rule that no raw `db.select()` escapes the repository layer.

### ADR-3: Tariff data is immutable and versioned; quotes pin a version
- **Context**: NFR-5 — a quote must be reproducible in 12 months, across Finance Law changes.
- **Decision**: `tariff_versions` is append-only. Each `tariff_lines` row belongs to exactly one version. A `quotes` row stores `tariff_version_id` plus the fully materialised breakdown.
- **Alternatives**: Mutable tariff table with `updated_at` — rejected: it destroys reproducibility the moment a rate changes.
- **Consequences**: Storage grows per ingestion run. At ~20k lines that is negligible. Ingestion writes a new version only when a diff actually exists.

### ADR-4: Store the computed breakdown, not just the inputs
- **Context**: Re-deriving an old quote requires the old engine code, not merely the old data.
- **Decision**: Persist the full `breakdown` JSON alongside the inputs and an `engine_version` string.
- **Alternatives**: Recompute on read — rejected: a later bug fix in the engine would silently rewrite history.
- **Consequences**: Quotes become audit records. "Re-run against current tariffs" creates a *new* quote and never mutates the old one.

### ADR-5: AI classification is advisory and always overridable
- **Context**: A wrong HS code is the highest-liability failure mode in the product (PRD risk table).
- **Decision**: Claude returns ranked candidates with confidence scores. Below a threshold the UI refuses to auto-select and surfaces the RTC (binding ruling) path instead. Whatever code the user confirms is what the engine uses.
- **Alternatives**: Auto-apply the top result — rejected on liability grounds.
- **Consequences**: Slightly more friction, far less legal exposure. Overrides are recorded and become signal for prompt tuning.

## 3. System Design
```
[Client]
   |
   v
[Next.js App Router]
   |
   +-- (server action) createQuote
   |        |
   |        +--> classify.suggest(description)   --> [Claude + pgvector]
   |        +--> tariff.getLine(hsCode, version)  --> [Postgres]
   |        +--> duty.calculateLandedCost(input, line)    [pure, no I/O]
   |        +--> shipments.saveQuote(breakdown)   --> [Postgres]
   |
   +-- (cron) ingestTariffs
            +--> scrape ADIL / PDF --> parse --> diff --> new tariff_version
```

## 4. Data Model
```
Organization --1:N--> Shipment --1:N--> Quote
Quote --N:1--> TariffVersion
TariffVersion --1:N--> TariffLine
TariffLine --1:N--> PreferentialRate (by origin agreement)
Quote --1:1--> ClassificationAttempt (nullable: null when HS entered manually)
```

## 5. API Design
Server Actions are the default. Route Handlers exist only where a real HTTP surface is needed.

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | /api/v1/classify | Suggest HS codes from a description | Org member |
| POST | /api/v1/quotes | Create a landed-cost quote | Org member |
| GET | /api/v1/quotes/:id | Get one quote (immutable) | Org member |
| GET | /api/v1/shipments | List org shipments | Org member |
| POST | /api/v1/shipments | Create a shipment | Org member |
| GET | /api/v1/shipments/:id/export | PDF / CSV export | Org member |
| GET | /api/v1/tariff/search | HS code lookup | Org member |
| POST | /api/webhooks/billing | CMI / Stripe subscription events | Signature |
| POST | /api/cron/ingest-tariffs | Nightly ingestion | Cron secret |

## 6. Security Considerations
Full baseline in `docs/security-diwana.md`.
- Authentication: Clerk, org-scoped JWT.
- Authorization: RBAC (owner / member / viewer) plus mandatory org-level row scoping.
- Data protection: no PII beyond user identity and commercial shipment values; TLS enforced; secrets in Vercel env.
- Key risks: cross-tenant leakage via a query that bypasses the repository; prompt injection through product descriptions that reach the Claude call.

## 7. Infrastructure
- Hosting: Vercel (web + cron)
- Database: Neon Postgres with pgvector
- Cache / rate limit: Upstash Redis
- CI/CD: GitHub Actions into Vercel
- Monitoring: Sentry (errors), PostHog (product analytics)

## 8. Technical Risks
| Risk | Mitigation | Owner |
|---|---|---|
| ADII source unavailable or ToS-hostile | Sprint 0 spike is a hard GO/NO-GO before any build | DevOps |
| PDF parse drift silently corrupts rates | Diff detection plus review queue; ingestion never auto-publishes a version where more than 5% of lines changed | Backend Dev |
| Cross-tenant leak | Repository layer, lint ban on raw db access, integration test asserting isolation | Security Engineer |
| Prompt injection via product description | Treat the description as untrusted data, use structured output, never let model output select the final code without user confirmation | Security Engineer |
| Cron time limit exceeded by ingestion | Chunk ingestion by HS chapter, resume via cursor | DevOps |
