# Security Baseline: Diwana.ma
**Architecture Reference**: docs/architecture-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: Security Engineer

## 1. Threat Model (5-Minute)
- **What are we building?** A multi-tenant SaaS where Moroccan importers compute customs landed cost on shipments, storing commercially sensitive supplier and pricing data.
- **Who would attack it?** Primarily a **competitor of one of our customers** — knowing a rival's supplier, origin country, unit CIF value and volumes is direct commercial intelligence. Secondarily opportunistic credential-stuffing and scrapers who want our tariff dataset.
- **Worst outcome?** Cross-tenant data leak exposing one importer's supplier pricing to another. This is worse than downtime and worse than losing our own data — it ends the business.

## 2. STRIDE Analysis (top risks only)
| Threat | Component | Mitigation | Status |
|---|---|---|---|
| Spoofing | Auth | Clerk-managed identity, MFA available, no self-rolled sessions | TODO |
| Spoofing | Cron endpoint | `CRON_SECRET` bearer check, constant-time compare, reject non-Vercel source | TODO |
| Tampering | Quote records | Quotes are immutable after creation; re-runs create new rows | TODO |
| Tampering | Tariff dataset | Versions append-only; ingestion refuses auto-publish when > 5% of lines change | TODO |
| Repudiation | Billing + quote history | `created_by` and `created_at` on every quote; webhook events logged raw | TODO |
| **Info Disclosure** | **All tenant tables** | **`org_id` predicate injected in the repository layer; lint bans raw db access; integration test proves isolation** | **TODO** |
| Info Disclosure | Error responses | Generic errors to client; full detail to Sentry only | TODO |
| DoS | Claude classify endpoint | Upstash rate limit per org and per user; cache identical descriptions | TODO |
| DoS | Tariff search | Indexed queries only, result cap, pagination required | TODO |
| Elevation of Privilege | Org roles | Role checked server-side on every mutation, never trusted from the client | TODO |
| **Prompt injection** | **Classifier** | **Product description treated as untrusted data, wrapped in delimiters, structured output enforced, model output never auto-applied (ADR-5)** | **TODO** |

## 3. Authentication Strategy
- **Type**: Clerk-hosted sessions with org-scoped JWT. No custom auth code.
- **MFA**: Optional per user, **required for the `owner` role** — owners control billing and can invite members.
- **Password policy**: delegated to Clerk (breach-list check enabled).
- **Session management**: Clerk defaults — HttpOnly, Secure, SameSite=Lax. Note: Vercel terminates TLS, but unlike the Bina/Auth.js case we are not self-hosting the auth layer, so the secure-cookie-prefix mismatch documented in that project does not apply here.

## 4. Authorization Model
- **Pattern**: RBAC plus mandatory tenant scoping. Both checks must pass; neither substitutes for the other.
- **Roles defined**:
  - `owner` — billing, member management, delete shipments, everything below
  - `member` — create/read/export quotes and shipments
  - `viewer` — read and export only, no writes
- **Resource-level checks**: yes. Every read and write is scoped by `org_id` at the query layer. A `viewer` who guesses another org's quote UUID gets a 404, not a 403 — we do not confirm existence across tenants.

## 5. Data Protection
- **PII fields**: user email and name (held by Clerk, not mirrored into our DB beyond `clerk_user_id`). Commercial data — supplier name, CIF value, origin — is **confidential but not personal**; it still gets the strictest handling under our threat model.
- **Encryption at rest**: Neon default (AES-256). No additional column encryption in v1 — the sensitive data is the entire shipment row, so column-level encryption would buy nothing while breaking query ability.
- **Encryption in transit**: HTTPS enforced, HSTS with preload.
- **Secrets management**: Vercel environment variables. `.env.example` committed, `.env` never. Gitleaks in CI.
- **Data retention**: quotes retained indefinitely for auditability (NFR-5). Org deletion triggers a hard delete of all org-scoped rows within 30 days.

## 6. Security Requirements for Dev Team
- [ ] All inputs validated server-side with Zod, including every server action argument
- [ ] Every tenant query goes through the repository layer — no raw `db.select()` in route handlers, server actions or components
- [ ] Output encoded for context; no `dangerouslySetInnerHTML` on any model or user output
- [ ] Product descriptions sent to Claude are wrapped as untrusted data and never concatenated into instructions
- [ ] No secrets in code, logs, or error messages; Sentry scrubbing configured for CIF values
- [ ] HTTPS only, security headers set (CSP, HSTS, X-Content-Type-Options, Referrer-Policy)
- [ ] Dependencies scanned in CI (Trivy SCA), code scanned (Semgrep SAST), secrets scanned (Gitleaks)
- [ ] Rate limits on `/api/v1/classify` and `/api/v1/tariff/search` before launch, not after

## 7. Legal / Compliance Posture
Not a classic compliance regime, but material to risk:
- **Law 09-08** (Moroccan personal data protection, CNDP) applies to user account data. Registration with the CNDP is required before public launch — flag for the user, it is an administrative task, not a code task.
- We are **not** a customs broker and must never present as one. Every quote carries the disclaimer required by FR-9: indicative estimate, not a binding ADII ruling.
- ToS must cap liability for duty estimates and direct users to the RTC process for binding determinations.
