# Issue Log
<!-- Tracks bugs, errors, blockers with status -->
<!-- Format: ### [YYYY-MM-DD HH:MM] BUG/BLOCKER/ERROR — Title -->


### [2026-09-17 10:05] [BLOCKER] — ADII tariff documents are behind an anti-automation WAF
- Specialist: DevOps/DevSecOps
- Summary: The chapter PDF index is reachable with a browser UA and lists 96 chapter PDFs at stable URLs, but the PDFs themselves are rejected by an F5 WAF with browser UA, Referer and cookies. robots.txt is itself blocked. No open-data tariff file exists on data.gov.ma. Automated ingestion per SDR-1 is not buildable on this source.
- Status: open
- Impact: critical

### [2026-09-17 10:10] [BLOCKER] — Competitive assumption in the PRD is false
- Specialist: Creative Intelligence
- Summary: tariq-ai.com is a direct competitor, not adjacent: its Tariq Customs product does tariff classification, duty calculation and DUM preparation, priced 2400 DH/yr individual and 9600 DH/yr for 5 users, aimed at the same transitaires and importers. diwanassist.ma is a second paid incumbent doing HS browsing, duty rates and import VAT, and its name collides with ours. PRD rated competition Low.
- Status: open
- Impact: critical
