# Risk Log
<!-- Tracks risks identified and their mitigations -->
<!-- Format: ### [YYYY-MM-DD HH:MM] SECURITY/PERFORMANCE/DEPENDENCY — Title -->


### [2026-09-17 09:45] [RISK] — ADII publishes no open tariff API
- Specialist: DevOps/DevSecOps
- Summary: Verified 2026-09-17. Tariff data must come from the ADIL assistant and douane.gov.ma/adil/Tarif_pdf.asp via scraping and PDF parsing. This is the single largest technical risk to the product.
- Status: open
- Impact: critical
- Mitigation: Sprint 0 Epic 0 is a hard GO/NO-GO gate before any product code. Fallbacks: licensed dataset, manual curation of top 500 HS lines, or pivot to idea A/E in the pool.

### [2026-09-17 09:45] [RISK] — Wrong duty quoted creates user financial loss and liability
- Specialist: Security Engineer
- Summary: A wrong HS code or stale rate could cost a customer real money.
- Status: open
- Impact: critical
- Mitigation: FR-9 disclaimer on every quote surface, as_of stamping, AI never auto-applies a code (ADR-5), no filing on the user behalf, capped-liability ToS.

### [2026-09-17 09:45] [RISK] — Free calculators commoditise the core feature
- Specialist: Creative Intelligence
- Summary: macalculatriceenligne.com offers a free duty simulator; tariq-ai.com covers Moroccan fiscal and customs law Q&A.
- Status: open
- Impact: medium
- Mitigation: the calculator is the funnel, not the product. Moat is the shipment workspace, quote history and preferential-origin logic.

### [2026-09-17 10:05] [RISK] — Two risks moved from open to REALIZED
- Specialist: Orchestrator
- Summary: The ADII-data risk and the commoditisation/competition risk both materialised during Sprint 0, within three hours of being written down. The scraping risk is worse than forecast (active WAF, not just absent API) and the competition risk is far worse (two paid incumbents, one direct).
- Status: realized
- Impact: critical
