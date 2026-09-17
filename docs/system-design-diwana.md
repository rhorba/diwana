# System Design: Diwana.ma
**PRD Reference**: docs/prd-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: System Designer

## 1. Non-Functional Requirements
| Attribute | Target | Notes |
|---|---|---|
| Availability | 99.5% SLA | B2B office-hours tool; no overnight criticality |
| Latency (p99) | 300ms (calc), 4s (AI classify) | Calc is pure CPU over indexed lookups |
| Throughput | 20 RPS peak | ~200 orgs, low-frequency use; not a consumer app |
| Data Volume | < 1 GB total | Tariff dataset ~20k lines x versions; shipments are small rows |
| Retention | Indefinite for quotes | NFR-5 auditability requires it |
| Recovery (RTO) | 4 hours | Managed Postgres PITR |
| Recovery (RPO) | 5 minutes | Neon PITR default |

## 2. Component Topology
```
[Browser: FR / AR-RTL / EN]
        |  HTTPS
        v
[Vercel Edge - Next.js 15 App Router]
        |
        +--> [Clerk] org auth, JWT with org_id claim
        |
        +--> [Server Actions / Route Handlers]
              |
              +--> [Duty Engine]  pure TS module, no I/O
              |
              +--> [Tariff Repository] --> [Neon Postgres]
              |                                ^
              +--> [Classifier Service]        |
              |        +--> [Claude API]       |
              |        +--> [pgvector search] -+
              |
              +--> [Upstash Redis] rate limit + classify cache

[Ingestion Worker - Vercel Cron, nightly]
        |
        +--> [ADII / ADIL scraper + PDF parser]
        +--> writes new tariff_version, diffs vs previous
        +--> raises review task on unexpected diff

[Observability: Sentry (errors) + PostHog (product analytics)]
```

## 3. Integration Patterns
| Integration | Pattern | Reason |
|---|---|---|
| ADII / ADIL tariff source | Scheduled batch scrape into versioned snapshot | No API exists; source is a slow-changing document, not a stream |
| Claude API (HS classification) | Synchronous request/response + cache | User is waiting; identical descriptions repeat often |
| Clerk | JWT with org_id claim | Org isolation must be provable at the query layer |
| CMI / Stripe billing | Webhook to subscription state | Standard; CMI is required for Moroccan cards |
| PDF export | On-demand server render | Low volume; no queue needed |

## 4. Scalability Strategy
- **Scaling approach**: horizontal, serverless (Vercel). No always-on compute to operate — a solo-dev constraint carried from the PRD.
- **Cache strategy**: Upstash Redis for classification results keyed by `hash(description + lang)`. Tariff lookups hit Postgres directly — the dataset is small and fully indexed.
- **Queue strategy**: **none**. Ingestion is one nightly cron job; PDF export is synchronous. Adding a queue now would be over-engineering (YAGNI). Re-evaluate if ingestion exceeds the function timeout.

## 5. System Design Decision Records

### SDR-1: Batch-ingest the tariff dataset instead of proxying ADIL live
- **NFR Driver**: NFR-1 (300ms p99) and NFR-5 (reproducibility).
- **Decision**: Scrape ADII/ADIL nightly into a versioned local `tariff_lines` table. Every quote references a `tariff_version_id`.
- **Alternatives**: Live-proxy ADIL per request — rejected: unpredictable latency, it would hammer a government site on every keystroke, and a quote could never be reproduced. Licensed commercial dataset — rejected for v1 on cost, retained as the Sprint 0 fallback.
- **Re-evaluate when**: ADII publishes a real API, or scraping is blocked.

### SDR-2: Serverless-only, no long-running worker
- **NFR Driver**: 20 RPS peak, solo operator.
- **Decision**: Vercel functions + Vercel Cron. No container, no Railway worker.
- **Alternatives**: Railway worker as in Bina — rejected: nothing here runs longer than a cron tick, so it would be infrastructure with no job to do.
- **Re-evaluate when**: ingestion exceeds the function timeout, or async bulk import is added.

### SDR-3: pgvector in the primary database rather than a dedicated vector store
- **NFR Driver**: < 1 GB data volume.
- **Decision**: Store HS description embeddings in Postgres via pgvector, queried alongside tariff lines.
- **Alternatives**: Pinecone / Qdrant — rejected: a second datastore to operate and pay for, for ~20k vectors.
- **Re-evaluate when**: vector count exceeds ~1M, or recall degrades measurably.

### SDR-4: The duty engine is a pure module with zero I/O
- **NFR Driver**: NFR-4 (100% branch coverage) and NFR-5 (auditability).
- **Decision**: `calculateLandedCost(input, tariffSnapshot) -> Breakdown`. All data is passed in; the function touches no database, no clock, no network.
- **Alternatives**: Service class with an injected repository — rejected: harder to test exhaustively, and non-determinism could leak in via the clock.
- **Re-evaluate when**: ideally never. This is the correctness core of the product.
