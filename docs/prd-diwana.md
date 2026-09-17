# PRD: Diwana.ma (ديوانة)
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: PM | **Status**: Draft

## 1. Problem Statement
Moroccan importers cannot know what a shipment will actually cost until it clears customs. Duty depends on an HS code most importers cannot determine correctly, and on origin-based preferential rates under Morocco's free-trade agreements that are routinely missed. So SMEs either overpay a transitaire to guess for them, or discover a 30% duty line after the goods have shipped. Morocco's trade deficit widened 26.5% over seven months to Sept 2026, and import cost control is now a survival issue for import-dependent SMEs.

## 2. Goals & Success Metrics
| Goal | Metric | Target |
|---|---|---|
| Importers trust our landed cost | Variance vs. actual cleared cost | ≤ 2% on 90% of shipments |
| HS classification is usable | Top-1 code accepted by user | ≥ 75% |
| Product is worth paying for | Trial → paid conversion | ≥ 8% |
| Retention (it's an ops tool, not a calculator) | Shipments created per paying org / month | ≥ 4 |
| Tariff data stays current | Max staleness of a tariff line | ≤ 7 days |

## 3. User Stories
As an **importer**, I want to enter a product description and get a duty estimate, so that I can price before I commit to a purchase order.
As an **importer**, I want to know if my supplier's country gives me a preferential rate, so that I stop overpaying duty I don't owe.
As an **importer**, I want all my shipments in one place with their real costs, so that I can compare suppliers and route decisions over time.
As a **transitaire**, I want to produce a client-ready cost breakdown in under a minute, so that I win quotes without unpaid analyst hours.
As a **finance manager**, I want to export the breakdown, so that I can reconcile against the actual DUM and invoice.

- [ ] S1: HS code lookup + AI classification from free-text description
- [ ] S2: Landed-cost calculation (CIF → DI → PFI → TVA)
- [ ] S3: Preferential origin rates (EU / USA / Turkey / Agadir Agreement)
- [ ] S4: Shipment workspace — save, list, compare, re-run
- [ ] S5: Export breakdown (PDF / CSV)
- [ ] S6: Multi-user organisations with role separation

## 4. Scope
### In Scope (v1.0)
- Import direction only, goods only
- Tariff dataset ingested from ADII/ADIL, versioned, with an `as_of` date on every quote
- AI HS classification with confidence score and a "not confident → request an RTC" fallback
- Landed cost: CIF value, Import Duty (2.5–30%), PFI 0.25%, TVA (20% standard + reduced rates)
- Preferential duty by certificate of origin
- FR (primary), AR (RTL), EN
- Multi-tenant orgs, subscription billing

### Out of Scope (v1.0)
- Export duties and export procedures
- Filing anything with ADII on the user's behalf — we never submit, we estimate
- Binding tariff rulings (RTC) — we link to the process, we do not replace it
- Freight booking, insurance sales, or customs brokerage services
- TIC on energy/alcohol/tobacco, antidumping and safeguard duties (v1.1)
- Accounting integration (v2)

## 5. Requirements
### Functional
- FR-1: User enters a product description in FR/AR/EN and receives ranked candidate HS codes with confidence scores.
- FR-2: User can override the AI-suggested HS code; the override is stored and used for the calculation.
- FR-3: System computes landed cost from CIF value, HS code, and country of origin.
- FR-4: System applies the preferential rate when the origin has an agreement AND the user asserts a valid certificate of origin; otherwise MFN rate.
- FR-5: Every quote is stamped with the tariff dataset version and `as_of` date, and is reproducible.
- FR-6: User can save a quote as a shipment, list shipments, and re-run a shipment against current tariffs.
- FR-7: User can export a shipment breakdown as PDF and CSV.
- FR-8: Orgs have owner / member / viewer roles; data is isolated per org.
- FR-9: Every calculation displays a legal disclaimer: indicative, not a binding ADII ruling.

### Non-Functional
- NFR-1: Performance — landed-cost calculation p99 < 300ms server-side; HS classification p99 < 4s.
- NFR-2: Security — org-level data isolation enforced at the query layer, not just the UI.
- NFR-3: Accessibility — WCAG 2.1 AA; full RTL parity for Arabic (not just mirrored layout — real translated copy).
- NFR-4: Correctness — the duty engine is a pure, fully unit-tested module; 100% branch coverage on it.
- NFR-5: Auditability — a quote produced today must be reproducible byte-for-byte in 12 months.

## 6. Constraints & Assumptions
- **ADII publishes no open tariff API.** Data comes from the ADIL assistant and `douane.gov.ma/adil/Tarif_pdf.asp`. Ingestion is scraping + PDF parsing. Verified 2026-09-17.
- We assume tariff structure (DI / PFI / TVA) is stable within a Finance Law year; rates change at LF boundaries (PLF 2027 pending).
- We assume importers know their CIF value or its components; we do not source freight rates.
- Legal posture: we are an estimation tool, never a customs agent. This must be explicit in-product and in the ToS.
- Team: solo developer. Architecture must be operable by one person.

## 7. Risks
| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| ADII scraping blocked, rate-limited, or ToS-hostile | M | **Critical** | Sprint 0 spike before any build; fallback = licensed dataset or manual curation of top 500 HS lines |
| PDF parse accuracy too low to trust | M | High | Measure on a full chapter in Sprint 0; accept only if ≥ 98% on structured fields; human review queue for diffs |
| Wrong duty quoted → user financial loss → liability | L | **Critical** | Disclaimer + `as_of` stamping + no filing on user's behalf + capped-liability ToS |
| Free calculators commoditise the core (macalculatriceenligne.com) | H | Medium | The calculator is the funnel, not the product; the moat is the workspace, history and preferential-origin logic |
| tariq-ai.com expands from fiscal Q&A into landed cost | M | Medium | Move fast on shipment ops depth, which is not their positioning |
| Tariff rates change at LF 2027 and quotes silently break | H | Medium | Versioned dataset + `as_of`; diff detection alerts on rate changes |

## 8. Timeline
| Milestone | Target Date |
|---|---|
| Foundation docs approved | 2026-09-17 |
| Sprint 0 GO/NO-GO (data feasibility) | 2026-09-19 |
| Architecture + schema implemented | 2026-09-26 |
| Duty engine + classifier done | 2026-10-10 |
| MVP ready (private beta, 5 importers) | 2026-10-24 |
