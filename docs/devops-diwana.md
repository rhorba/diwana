# DevOps Foundation: Diwana.ma
**Architecture**: docs/architecture-diwana.md
**Security**: docs/security-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: DevOps/DevSecOps

## 1. Environment Strategy
| Environment | Purpose | Deploy Trigger | Database |
|---|---|---|---|
| local | Development | Manual (`pnpm dev`) | Neon branch `dev` or local Docker Postgres+pgvector |
| preview | Per-PR review | Automatic on PR open/update | Neon branch created per PR, dropped on merge |
| production | Live users | Manual promote from `main` | Neon primary |

No separate long-lived staging. Neon branching gives a real database per PR, which is strictly better than one shared staging box that drifts.

## 2. CI Pipeline (GitHub Actions)
```yaml
stages:
  - lint            # eslint + prettier + tsc --noEmit (no `any`, no raw db access)
  - test-unit       # vitest, fails if lib/duty branch coverage < 100%
  - test-integration# testcontainers postgres+pgvector; includes tenant isolation suite
  - test-eval       # classifier eval set; fails if top-1 < 75%
  - security-scan   # semgrep (SAST), trivy (SCA), gitleaks (secrets)
  - build           # next build
  - e2e             # playwright against the preview deployment
  - deploy-preview  # automatic on PR
  - deploy-prod     # manual approval gate on main
```

Coverage gate is enforced in `vitest.config.ts` thresholds, not by a shell grep — the build fails on its own.

**CI monitoring is mandatory** (CTS rule 11): if CI is red after a push, all other work stops until it is green again.

## 3. Infrastructure
- **Hosting**: Vercel — web + cron. No container runtime to maintain (SDR-2).
- **Compute**: serverless functions; Node runtime for anything touching Postgres.
- **Database**: Neon Postgres 16 + pgvector, PITR enabled.
- **Cache / rate limit**: Upstash Redis.
- **Secrets**: Vercel environment variables, scoped per environment. `.env.example` committed; `.env` in `.gitignore` from commit one.
- **Monitoring**: Sentry (errors, with CIF-value scrubbing), PostHog (funnel + activation analytics).

## 4. Security Scanning Gates
| Scanner | Scan Type | Fail Threshold |
|---|---|---|
| Semgrep | SAST — `p/owasp-top-ten` + `p/typescript` | Any critical finding |
| Trivy | SCA — dependency CVEs | Critical CVEs |
| Gitleaks | Secrets detection | Any secret found |
| Custom ESLint rule | Raw `db.` access outside `lib/*/repository.ts` | Any occurrence — this is the tenant-isolation guardrail |

## 5. Environment Variables (CTS rule 10 — collect upfront)
Written to `.env.example`. **Values needed from the user before EXECUTE begins:**

| Variable | Purpose | Blocking for |
|---|---|---|
| `DATABASE_URL` | Neon connection string | Sprint 1 |
| `DIRECT_URL` | Neon direct connection for migrations | Sprint 1 |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk client | Sprint 1 |
| `CLERK_SECRET_KEY` | Clerk server | Sprint 1 |
| `ANTHROPIC_API_KEY` | HS classification (claude-sonnet-5) | Sprint 4 |
| `UPSTASH_REDIS_REST_URL` | Rate limiting + classify cache | Sprint 4 |
| `UPSTASH_REDIS_REST_TOKEN` | idem | Sprint 4 |
| `CRON_SECRET` | Authenticates the ingestion cron endpoint | Sprint 2 |
| `SENTRY_DSN` | Error monitoring | Sprint 6 |
| `NEXT_PUBLIC_POSTHOG_KEY` | Product analytics | Sprint 6 |
| `CMI_MERCHANT_ID` / `CMI_STORE_KEY` | Moroccan card billing | Sprint 6 |
| `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` | International billing | Sprint 6 |

Nothing here is needed to start Sprint 0 — the spike touches only public ADII endpoints.

## 6. Ingestion Cron
```
Schedule: 0 3 * * *  (03:00 Africa/Casablanca)
Endpoint: POST /api/cron/ingest-tariffs
Auth:     Bearer CRON_SECRET, constant-time comparison
Behaviour:
  - chunk by HS chapter, resume via cursor (function time limit)
  - write an unpublished tariff_version
  - diff against last published version
  - auto-publish only if changed lines ≤ 5%
  - otherwise raise a review task and alert via Sentry
```

## 7. Monitoring Baseline
| Signal | Tool | Alert Threshold |
|---|---|---|
| Errors | Sentry | Error rate > 5/min, or any error in `lib/duty/` at all |
| Ingestion health | Sentry cron monitor | No successful run in 48h |
| Tariff staleness | Custom check | `as_of` older than 7 days (PRD success metric) |
| Latency | Vercel Analytics | Calc p99 > 300ms, classify p99 > 4s |
| Uptime | Better Stack | 99.5% SLO breach |
| Classifier quality | PostHog | Override rate > 25% (implies top-1 below target in the wild) |

## 8. Backup & Recovery
- Neon PITR: RPO 5 minutes, RTO 4 hours (system design NFRs).
- The tariff dataset is reproducible by re-running ingestion, but published versions are **not** regenerable identically — they are the audit trail. They are covered by PITR and must never be truncated.
- Monthly restore drill into a Neon branch. A backup that has never been restored is not a backup.
