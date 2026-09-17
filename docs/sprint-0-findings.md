# Sprint 0 — Data Feasibility Spike: Findings
**Epic**: 0 (GO/NO-GO gate) | **Date**: 2026-09-17 | **Author**: DevOps + Backend Dev + Creative Intelligence

## Verdict: ⛔ NO-GO on Diwana.ma as specified

Both blocking stories failed. The gate fired exactly as designed, before any product code was written.

---

## Story 0.1 — Probe the ADII/ADIL source: **FAIL**

### What we found
| Target | Result |
|---|---|
| `douane.gov.ma/robots.txt` | **Blocked by WAF** — returns a 246-byte "Request Rejected" page (F5 ASM, support ID issued) |
| `douane.gov.ma/adil/Tarif_pdf.asp` | Reachable with a browser User-Agent. 52 KB HTML index listing **96 chapter PDFs** at stable URLs `/tarif/pdf/NN.pdf` |
| `douane.gov.ma/tarif/pdf/85.pdf` | **Blocked by WAF** — rejected with browser UA, with `Referer`, and with a cookie jar |
| Fetch from datacenter IPs | **Blocked entirely** — every request rejected regardless of headers |
| `data.gov.ma` open tariff file | **Does not exist** — no downloadable open-data nomenclature published |

### What this means
The tariff index is publicly browsable, but **the documents themselves sit behind an anti-automation WAF**. The architecture in SDR-1 — a nightly cron that scrapes and re-ingests — cannot be built on this source.

We could not even read `robots.txt` to learn the site's stated crawl policy, because the WAF blocks that too. In the absence of a stated policy, a WAF that rejects every non-interactive client is itself the policy.

**We stopped here deliberately.** Getting past this would mean browser-fingerprint spoofing, residential proxy rotation, or similar evasion. That is the wrong foundation for a business whose entire value proposition is regulatory trustworthiness, and it would put the product one IP-ban away from total failure. Not pursued.

### Legitimate paths that remain
1. **Ask the ADII.** Request the tariff dataset and its reuse terms directly. Costs an email and some patience; it is the only path that ends with a defensible data supply. Morocco's Digital Morocco 2030 programme and the $250M World Bank digital transformation loan both push toward open public data — the timing is unusually favourable for this ask.
2. **License a commercial dataset.** Global tariff data vendors cover Morocco. Costs money; removes the risk entirely.
3. **Manually curate the top 500 HS lines.** Covers the bulk of real SME import volume. Cheap to start, expensive to keep current, and "current" is a stated PRD success metric (≤ 7 days staleness).

---

## Story 0.2 — Parse one full HS chapter: **BLOCKED**

Could not obtain a chapter PDF, so parse accuracy could not be measured. The acceptance criterion (≥ 98% extraction accuracy on `hs_code`, `description`, `di_rate`, `tva_rate`) is **unverified**, not failed — we never got to test it.

One structural fact was confirmed from secondary sources: the Moroccan tariff code is **10 digits** — 6 from the Harmonized System, 4 Morocco-specific. The `tariff_lines.hs_code TEXT` design in the database doc is correct.

---

## Story 0.3 — Competitive teardown: **FAIL — the PRD's core assumption was wrong**

The PRD rated competition "Low". That rating does not survive contact with the market.

### tariq-ai.com — a direct competitor, not an adjacent one
Earlier in this session it was characterised as a fiscal/legal Q&A tool. That was wrong. Its own product page describes **Tariq Customs** as handling *"tariff classification, duty calculation, and customs documentation (DUM preparation, duty liquidation)"*.

| | Tariq | Diwana (planned) |
|---|---|---|
| HS classification | ✅ core feature | ✅ planned |
| Duty calculation | ✅ core feature | ✅ planned |
| DUM preparation | ✅ | ❌ explicitly out of scope |
| Cited sources | ✅ a stated differentiator | ⚠️ we planned confidence scores, not citations |
| Shipment workspace | ❌ | ✅ **our only real gap-fill** |
| Pricing | 2,400 DH/yr individual; 9,600 DH/yr for 5 users | not set |
| Target users | transitaires, accountants, law firms, import/export | the same people |

They also ship as a Claude and ChatGPT connector, which is a distribution channel we had not considered.

### diwanassist.ma — a second incumbent, and a naming collision
Offers HS tree browsing, duty rates and import VAT calculation, on a paid plan. Carries the same "not an official government platform" disclaimer we had designed for FR-9.

**The name is a problem.** "Diwana" against an existing "DiwanAssist" in the identical category invites confusion and a trademark argument we would lose time on.

### What survives
The **shipment workspace with reproducible quote history** is still genuinely unoccupied. Neither competitor manages shipments over time. But that reduces Diwana from "empty lane, high revenue ceiling" to "a feature gap in a market with two established paid incumbents" — a materially worse bet than the one that was approved.

---

## Story 0.4 — Five importer/transitaire discovery conversations: **NOT DONE**

User-owned; requires real conversations. **Given the competitive findings, this is now the highest-value remaining action**: the question is no longer "is this a problem?" but "are Tariq and DiwanAssist failing these people in a way a workspace would fix?"

---

## Impact on approved documents
These docs are now partially invalidated and must be revised before any implementation:

| Doc | What is now wrong |
|---|---|
| `prd-diwana.md` | §7 competition risk rated "H / Medium" — should be realized and critical. §6 assumes scrapeable data. Product name is contested. |
| `system-design-diwana.md` | **SDR-1 is void** — nightly scrape ingestion is not buildable on this source |
| `architecture-diwana.md` | §8 risk "ADII source unavailable" has materialized |
| `database-diwana.md` | Still valid — `tariff_versions.source` already anticipates `'manual'` and `'licensed'` |
| `stories-diwana.md` | Epic 2 (tariff pipeline) must be rewritten around whichever data path is chosen |

`security-`, `ux-`, `ui-`, `test-strategy-` and `devops-` remain valid as written.

---

## Recommendation
Do not proceed to Sprint 1. The decision belongs to the user, and there are three honest options:

1. **Pivot** to idea **A (Da3m.ma)** or **E (Fatoura.ma)** from `Desktop/idea-pool-sept-2026.md`. Diwana's foundation docs stay as a reusable template — none of the 10 documents were wasted effort.
2. **Park Diwana pending the ADII data request.** Send the email, pursue idea A meanwhile, revisit if the data path opens.
3. **Proceed anyway, repositioned** — shipment-ops-first, licensed or curated data, new name. Viable, but it is a different and harder product than the one approved this morning, and it should be re-entered at BRAINSTORM rather than continued from PLAN.
