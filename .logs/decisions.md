# Decision Log
<!-- Tracks architecture decisions, approach selections, tool choices -->
<!-- Format: ### [YYYY-MM-DD HH:MM] ARCHITECTURE/APPROACH/TOOL — Title -->


### [2026-09-17 08:55] [BRAINSTORM] — Which product to build from the Sept 2026 news
- Options presented: SIMPLE Da3m.ma subsidy radar / BALANCED Da3m+Tahwil money property / COMPREHENSIVE Diwana.ma customs SaaS
- Selected: COMPREHENSIVE — Diwana.ma
- Rationale: Highest revenue ceiling, empty lane vs existing repos, anchored to the 26.5% trade-deficit widening. User also asked that the full 7-idea pool be preserved for later.
- Status: resolved

### [2026-09-17 09:40] [ARCHITECTURE] — Foundation decisions recorded
- Specialist: Software Architect + System Designer + DBA + Security Engineer
- Summary: SDR-1 batch tariff ingestion over live ADIL proxy; SDR-2 serverless-only, no worker; SDR-3 pgvector in primary DB; SDR-4 pure duty engine with zero I/O. ADR-1 modular monolith; ADR-2 multi-tenant from migration 001 (lesson from Moqawil); ADR-3 versioned immutable tariff data; ADR-4 store computed breakdown; ADR-5 AI classification advisory only.
- Status: resolved
- Impact: high
