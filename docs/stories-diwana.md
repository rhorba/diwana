# Stories: Diwana.ma
**PRD**: docs/prd-diwana.md
**Architecture**: docs/architecture-diwana.md
**Test Strategy**: docs/test-strategy-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: Scrum Master + Test Architect

---

## Epic 0: Data Feasibility Spike ⛔ GO/NO-GO
Prove the ADII tariff dataset can be legally and reliably obtained before a line of product code is written.

### Story 0.1: Probe the ADII/ADIL source
**Priority**: Must | **Size**: S | **Specialist**: DevOps/DevSecOps
**Description**: As the team, we need to know whether the tariff source is reachable and reusable, so that we do not build a product on data we cannot have.
**Acceptance Criteria**:
```gherkin
Given the ADIL assistant and the tariff PDF endpoint
When I probe response shape, robots.txt, rate limits and terms of use
Then I can state GO or NO-GO on automated retrieval with evidence
```
**Technical Notes**: Source of record is `douane.gov.ma/adil/` plus `Tarif_pdf.asp`. Document findings in `docs/adr/adr-006-tariff-source.md`.
**Dependencies**: none

### Story 0.2: Parse one full HS chapter
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev
**Description**: As the team, we need measured parse accuracy on real data, so that the ingestion risk is quantified rather than assumed.
**Acceptance Criteria**:
```gherkin
Given one complete HS chapter from the ADII tariff
When I parse it into structured hs_code, description, di_rate, tva_rate
Then extraction accuracy on those fields is measured against a hand-checked sample
And the result is at least 98% or the risk is escalated
```
**Dependencies**: 0.1

### Story 0.3: Competitive teardown
**Priority**: Should | **Size**: S | **Specialist**: Creative Intelligence
**Description**: As the PM, I want to know exactly what tariq-ai.com and the free simulators do not do, so that we build the part that is defensible.
**Dependencies**: none

### Story 0.4: Five importer/transitaire discovery conversations
**Priority**: Must | **Size**: M | **Specialist**: Project Manager
**Description**: As the PM, I want evidence of willingness to pay, so that we do not build a well-engineered product nobody buys.
**Acceptance Criteria**: 5 conversations logged; at least 3 confirm they currently pay for, or lose money on, this problem.
**Dependencies**: none

> ⛔ **GATE**: if 0.1 or 0.2 fails, stop. Fall back to a licensed dataset, to manual curation of the top 500 HS lines, or pivot to idea A / E from `Desktop/idea-pool-sept-2026.md`.

---

## Epic 1: Foundation
A running, multi-tenant, CI-green skeleton.

### Story 1.1: Project scaffold + CI
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev + DevOps
**Acceptance Criteria**:
```gherkin
Given a fresh clone
When I run install, lint, typecheck and test
Then all pass, and CI runs the same stages on every PR
```
**Technical Notes**: Next.js 15 App Router, TS strict (no `any`), Drizzle, Vitest, Playwright, Prettier + ESLint incl. the `no-restricted-imports` rule banning raw `db.` access outside repositories (security baseline §6).

### Story 1.2: Schema migrations 0001–0005
**Priority**: Must | **Size**: M | **Specialist**: DBA
**Technical Notes**: Exactly as specified in `docs/database-diwana.md` §3 and §5. Multi-tenant from migration one (ADR-2).
**Dependencies**: 1.1

### Story 1.3: Clerk org auth + repository layer
**Priority**: Must | **Size**: L | **Specialist**: Backend Dev + Security Engineer
**Acceptance Criteria**:
```gherkin
Given organisation A has a quote with a known id
And I am authenticated as a member of organisation B
When I request that quote by id
Then the response is 404 and does not reveal the quote exists
```
**Technical Notes**: `org_id` predicate injected in the repository layer. This story delivers the tenant-isolation test suite, which becomes CI-blocking from here on.
**Dependencies**: 1.2

---

## Epic 2: Tariff Data Pipeline
Versioned, trustworthy, current tariff data.

### Story 2.1: Scraper + PDF parser
**Priority**: Must | **Size**: L | **Specialist**: Backend Dev
**Technical Notes**: Chunk by HS chapter with cursor resume (architecture risk table). Golden-file tests per the test strategy.
**Dependencies**: 0.2, 1.2

### Story 2.2: Versioned ingestion with diff gate
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev
**Acceptance Criteria**:
```gherkin
Given a published tariff version exists
When ingestion produces a version where more than 5% of lines changed
Then the new version is stored unpublished, a review task is raised
And quotes continue to use the last published version
```
**Dependencies**: 2.1

### Story 2.3: Preferential rates by agreement
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev + DBA
**Technical Notes**: EU, USA, TR, AGADIR, UK, AELE. This is the feature that visibly pays for the subscription (UX principle 3).
**Dependencies**: 2.1

### Story 2.4: Nightly cron + staleness alert
**Priority**: Must | **Size**: S | **Specialist**: DevOps
**Dependencies**: 2.2

---

## Epic 3: Duty Engine
The correctness core.

### Story 3.1: Pure landed-cost calculator
**Priority**: Must | **Size**: L | **Specialist**: Backend Dev
**Acceptance Criteria**:
```gherkin
Given the published tariff version as of 2026-09-15
And HS code "8544.42.90.00" has a DI rate of 2.5% and TVA of 20%
When I quote a CIF value of 120000 MAD from origin "CN"
Then the import duty is 3000.00 MAD
And the PFI is 300.00 MAD
And the TVA is 24660.00 MAD
And the total landed cost is 147960.00 MAD
```
**Technical Notes**: `calculateLandedCost(input, tariffSnapshot)` — zero I/O (SDR-4). Decimal arithmetic only, never JS `number` for money. **100% branch coverage, CI-enforced.**
**Dependencies**: 1.1

### Story 3.2: Preferential origin resolution + savings delta
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev
**Acceptance Criteria**:
```gherkin
Given HS code "8544.42.90.00" has an EU preferential DI rate of 0%
When I quote from origin "DE" without claiming a certificate of origin
Then the MFN rate of 2.5% is applied
When I claim a valid certificate of origin
Then the preferential rate of 0% is applied and the saving versus MFN is displayed
```
**Dependencies**: 3.1, 2.3

### Story 3.3: Quote persistence with version + engine pinning
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev
**Acceptance Criteria**:
```gherkin
Given a shipment with a quote created under tariff version V1
And a newer published tariff version V2 exists
When I re-run the shipment against current tariffs
Then a new quote is created referencing V2
And the original quote still references V1 with its original breakdown
```
**Technical Notes**: ADR-3 + ADR-4. Quotes are immutable; no `updated_at` column exists by design.
**Dependencies**: 3.1, 1.3

---

## Epic 4: HS Classification
### Story 4.1: Embedding index over tariff descriptions
**Priority**: Must | **Size**: M | **Specialist**: Backend Dev
**Technical Notes**: pgvector HNSW index (SDR-3). Embeddings regenerate per published tariff version.
**Dependencies**: 2.2

### Story 4.2: Claude classifier with structured output
**Priority**: Must | **Size**: L | **Specialist**: Backend Dev + Security Engineer
**Acceptance Criteria**:
```gherkin
Given the classifier returns a top candidate with confidence below the threshold
When the candidates are displayed
Then no candidate is pre-selected, the RTC path is offered
And manual HS entry remains available
```
**Technical Notes**: claude-sonnet-5. Description is **untrusted data** — delimited, never concatenated into instructions (security baseline STRIDE row). Model output never auto-applies (ADR-5). Validate returned codes exist in our dataset.
**Dependencies**: 4.1

### Story 4.3: Classification cache + rate limiting
**Priority**: Must | **Size**: S | **Specialist**: Backend Dev
**Dependencies**: 4.2

### Story 4.4: Eval set + CI quality gate
**Priority**: Must | **Size**: M | **Specialist**: Test Architect
**Acceptance Criteria**: 100-description eval set; CI fails if top-1 < 75% or top-3 < 90%.
**Dependencies**: 4.2

### Story 4.5: Graceful degradation when Claude is down
**Priority**: Must | **Size**: S | **Specialist**: Frontend Dev
**Acceptance Criteria**:
```gherkin
Given the Claude API is unavailable
When I submit a product description
Then I am shown a manual HS code entry field and can still complete a quote
```
**Dependencies**: 4.2

---

## Epic 5: Workspace UI
### Story 5.1: Design tokens + base components
**Priority**: Must | **Size**: M | **Specialist**: UI Designer + Frontend Dev
**Technical Notes**: Tokens from `docs/ui-diwana.md` §2. BreakdownCard, ConfidenceBadge, SavingsBanner, DisclaimerBand, AsOfStamp.

### Story 5.2: Quote builder screen
**Priority**: Must | **Size**: L | **Specialist**: Frontend Dev
**Technical Notes**: Wireframe in `docs/ux-diwana.md` §4. Every state from §5 implemented — no dead ends.
**Dependencies**: 5.1, 3.2, 4.2

### Story 5.3: Shipments list + detail with quote history
**Priority**: Must | **Size**: L | **Specialist**: Frontend Dev
**Dependencies**: 5.1, 3.3

### Story 5.4: i18n FR / AR-RTL / EN
**Priority**: Must | **Size**: L | **Specialist**: Frontend Dev + Copywriter
**Acceptance Criteria**: Arabic is **fully translated copy**, not mirrored French. Numbers and HS codes stay LTR via `<bdi>`. DoD is a native-reader pass.
**Technical Notes**: The explicit lesson from Bina — RTL plumbing without translation is not done.
**Dependencies**: 5.2

### Story 5.5: PDF + CSV export
**Priority**: Should | **Size**: M | **Specialist**: Frontend Dev
**Dependencies**: 5.3

### Story 5.6: Public calculator + HS SEO pages
**Priority**: Should | **Size**: M | **Specialist**: Frontend Dev + Content Marketer
**Technical Notes**: The acquisition funnel (UX §2). Anonymous quote must survive sign-up.
**Dependencies**: 5.2

---

## Epic 6: Launch Readiness
### Story 6.1: Billing — CMI + Stripe
**Priority**: Must | **Size**: L | **Specialist**: Backend Dev
**Technical Notes**: Webhook signature verification + idempotency.

### Story 6.2: Sentry + PostHog + staleness monitor
**Priority**: Must | **Size**: M | **Specialist**: DevOps
**Technical Notes**: Sentry scrubbing for CIF values. Alert thresholds from `docs/devops-diwana.md` §7.

### Story 6.3: Security review + scan gates green
**Priority**: Must | **Size**: M | **Specialist**: Security Engineer + DevSecOps
**Technical Notes**: Full STRIDE table moved from TODO to Done. Adversarial checklist executed.

### Story 6.4: E2E suite + recorded walkthrough
**Priority**: Must | **Size**: M | **Specialist**: Tester
**Technical Notes**: CTS rule 9 — record to `.recordings/v1.0-[date].webm`.

### Story 6.5: Legal — ToS, disclaimer, CNDP registration
**Priority**: Must | **Size**: M | **Specialist**: PM + Copywriter
**Technical Notes**: Law 09-08 CNDP registration is an administrative task for the user, not a code task. Liability cap + RTC signposting per security baseline §7.

---

## Sprint Allocation
| Sprint | Stories | Estimated Effort |
|---|---|---|
| **Sprint 0** ⛔ | 0.1, 0.2, 0.3, 0.4 | ~3h — GO/NO-GO |
| Sprint 1 | 1.1, 1.2, 1.3 | ~4h |
| Sprint 2 | 2.1, 2.2, 2.3, 2.4 | ~6h |
| Sprint 3 | 3.1, 3.2, 3.3 | ~4h |
| Sprint 4 | 4.1, 4.2, 4.3, 4.4, 4.5 | ~5h |
| Sprint 5 | 5.1, 5.2, 5.3, 5.4, 5.5, 5.6 | ~6h |
| Sprint 6 | 6.1, 6.2, 6.3, 6.4, 6.5 | ~5h |
| | **Total** | **~33h** (Sprint 0 excluded) |

## Traceability
| PRD Req | Decision | Story | Test Scenario |
|---|---|---|---|
| FR-1 | ADR-5 | 4.2 | "Low confidence never auto-selects" |
| FR-2 | ADR-5 | 4.2 | classification_attempts.was_override |
| FR-3 | SDR-4 | 3.1 | "Standard MFN duty on a non-agreement origin" |
| FR-4 | — | 3.2 | "Preferential rate applied only when origin is claimed" |
| FR-5 | ADR-3, ADR-4 | 3.3 | "Re-running a shipment creates a new quote" |
| FR-6 | ADR-3 | 5.3 | quote history |
| FR-7 | — | 5.5 | export |
| FR-8 | ADR-2 | 1.3 | "A user cannot read another organisation's quote" |
| FR-9 | — | 5.1 | DisclaimerBand with as_of |
| NFR-4 | SDR-4 | 3.1 | 100% branch coverage gate |
| NFR-5 | ADR-3, ADR-4 | 3.3 | immutability scenario |
