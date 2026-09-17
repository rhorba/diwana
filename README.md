# Diwana.ma — ديوانة

**Sachez ce que votre importation coûte vraiment. Avant qu'elle n'embarque.**
_Know what your import really costs. Before it ships._

Diwana is a landed-cost and customs-duty platform for Moroccan importers and transitaires:
AI-assisted HS classification, duty and VAT calculation against a versioned ADII tariff
dataset, preferential-origin savings under Morocco's free-trade agreements, and a shipment
workspace that keeps every quote as a reproducible audit record.

> ⚠️ Diwana produces **indicative estimates**. It is not a customs broker and its output is
> not a binding ADII ruling. For a binding determination, request an RTC from the ADII.

---

## Status

**Foundation phase — no product code yet.** This repository currently contains the CTS
framework and the ten approved foundation documents. Sprint 0 is a hard GO/NO-GO gate on
tariff-data feasibility before implementation begins.

| Sprint | Scope | Status |
|---|---|---|
| Foundation | 10 expert docs | ✅ Done |
| Sprint 0 ⛔ | Data feasibility spike (GO/NO-GO) | Pending |
| Sprint 1 | Scaffold, schema, multi-tenant auth | Not started |
| Sprint 2 | Tariff ingestion pipeline | Not started |
| Sprint 3 | Duty engine | Not started |
| Sprint 4 | HS classification | Not started |
| Sprint 5 | Workspace UI (FR / AR-RTL / EN) | Not started |
| Sprint 6 | Launch readiness | Not started |

## Documentation

| Doc | Owner |
|---|---|
| [PRD](docs/prd-diwana.md) | Project Manager |
| [System Design](docs/system-design-diwana.md) | System Designer |
| [Architecture](docs/architecture-diwana.md) | Software Architect |
| [Security Baseline](docs/security-diwana.md) | Security Engineer |
| [Database Design](docs/database-diwana.md) | DBA |
| [UX Foundation](docs/ux-diwana.md) | UX Designer |
| [UI Foundation](docs/ui-diwana.md) | UI Designer |
| [Test Strategy](docs/test-strategy-diwana.md) | Test Architect |
| [DevOps Foundation](docs/devops-diwana.md) | DevOps/DevSecOps |
| [Epics & Stories](docs/stories-diwana.md) | Scrum Master |

## Planned stack

Next.js 15 (App Router) · TypeScript · Drizzle · Neon Postgres + pgvector · Clerk (orgs) ·
Tailwind + shadcn/ui · Zod · Vitest + Playwright · Upstash Redis · Sentry + PostHog · Vercel

## Why now

Morocco's trade deficit widened **26.5%** over the seven months to September 2026 on energy
prices, while imports remain the cost centre SMEs understand least. Duty depends on an HS
code most importers cannot determine, and on origin-based preferential rates that are
routinely left unclaimed.

## Working method

Built with the [CTS framework](CLAUDE.md) — six mandatory phases, specialist handoffs,
append-only logs in `.logs/`, and document-first development.
