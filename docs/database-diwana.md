# Database Design: Diwana.ma
**Architecture Reference**: docs/architecture-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: DBA

## 1. Database Selection
- **Engine**: PostgreSQL 16 with the `pgvector` extension.
- **Rationale**: YAGNI default, and it does three jobs at once here — relational tariff data, JSONB breakdown storage, and vector search for HS classification (SDR-3). No second datastore to operate.
- **Hosting**: Neon (serverless, branching for preview environments, PITR).
- **ORM**: Drizzle. Raw SQL only for the ingestion diff aggregation.

## 2. Entity-Relationship Model
```
organizations --1:N--> shipments --1:N--> quotes
quotes --N:1--> tariff_versions
tariff_versions --1:N--> tariff_lines
tariff_lines --1:N--> preferential_rates
quotes --1:1--> classification_attempts (nullable)
organizations --1:N--> memberships
```

## 3. Schema Design
```sql
-- Tenancy -------------------------------------------------------------
CREATE TABLE organizations (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_org_id  TEXT NOT NULL UNIQUE,
  name          TEXT NOT NULL,
  plan          TEXT NOT NULL DEFAULT 'trial',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE memberships (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id         UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  clerk_user_id  TEXT NOT NULL,
  role           TEXT NOT NULL CHECK (role IN ('owner','member','viewer')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (org_id, clerk_user_id)
);

-- Tariff reference data (append-only, not tenant-scoped) ---------------
CREATE TABLE tariff_versions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  as_of        DATE NOT NULL,
  source       TEXT NOT NULL,              -- 'adil-scrape' | 'manual' | 'licensed'
  line_count   INTEGER NOT NULL,
  diff_summary JSONB,                      -- {added, removed, changed}
  published    BOOLEAN NOT NULL DEFAULT FALSE,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (as_of, source)
);

CREATE TABLE tariff_lines (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version_id  UUID NOT NULL REFERENCES tariff_versions(id) ON DELETE CASCADE,
  hs_code     TEXT NOT NULL,               -- 10-digit Moroccan nomenclature
  description_fr TEXT NOT NULL,
  description_ar TEXT,
  di_rate     NUMERIC(5,2) NOT NULL,       -- import duty %, 2.5 - 30
  tva_rate    NUMERIC(5,2) NOT NULL DEFAULT 20.00,
  pfi_rate    NUMERIC(5,4) NOT NULL DEFAULT 0.0025,
  unit        TEXT,
  embedding   VECTOR(1024),                -- description embedding for search
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (version_id, hs_code)
);

CREATE TABLE preferential_rates (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tariff_line_id UUID NOT NULL REFERENCES tariff_lines(id) ON DELETE CASCADE,
  agreement      TEXT NOT NULL,            -- 'EU' | 'USA' | 'TR' | 'AGADIR' | 'UK' | 'AELE'
  di_rate        NUMERIC(5,2) NOT NULL,
  UNIQUE (tariff_line_id, agreement)
);

-- Tenant-owned operational data ---------------------------------------
CREATE TABLE shipments (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  reference   TEXT NOT NULL,
  supplier    TEXT,
  origin_country CHAR(2) NOT NULL,         -- ISO 3166-1 alpha-2
  status      TEXT NOT NULL DEFAULT 'draft',
  created_by  TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE quotes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id            UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  shipment_id       UUID REFERENCES shipments(id) ON DELETE CASCADE,
  tariff_version_id UUID NOT NULL REFERENCES tariff_versions(id),
  engine_version    TEXT NOT NULL,         -- ADR-4: pins the calculation code
  hs_code           TEXT NOT NULL,
  origin_country    CHAR(2) NOT NULL,
  currency          CHAR(3) NOT NULL,
  fx_rate           NUMERIC(12,6) NOT NULL,
  cif_value_mad     NUMERIC(14,2) NOT NULL,
  preferential_claimed BOOLEAN NOT NULL DEFAULT FALSE,
  breakdown         JSONB NOT NULL,        -- ADR-4: full materialised result
  total_mad         NUMERIC(14,2) NOT NULL,
  created_by        TEXT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
  -- deliberately NO updated_at: quotes are immutable audit records
);

CREATE TABLE classification_attempts (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id       UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  quote_id     UUID UNIQUE REFERENCES quotes(id) ON DELETE SET NULL,
  description  TEXT NOT NULL,
  lang         CHAR(2) NOT NULL,
  candidates   JSONB NOT NULL,             -- [{hs_code, confidence, reason}]
  chosen_code  TEXT,
  was_override BOOLEAN NOT NULL DEFAULT FALSE,
  model        TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 4. Index Strategy
| Table | Index Name | Columns | Query Pattern |
|---|---|---|---|
| memberships | idx_memberships_user | (clerk_user_id) | Resolve a user's orgs at auth time |
| tariff_lines | idx_tariff_lines_lookup | (version_id, hs_code) | Primary duty lookup — covered by the UNIQUE |
| tariff_lines | idx_tariff_lines_hs_prefix | (hs_code text_pattern_ops) | HS prefix search "8703%" |
| tariff_lines | idx_tariff_lines_embedding | HNSW (embedding vector_cosine_ops) | Semantic HS candidate search |
| preferential_rates | idx_pref_line_agreement | (tariff_line_id, agreement) | Preferential rate resolution — covered by UNIQUE |
| shipments | idx_shipments_org | (org_id, created_at DESC) | Tenant shipment list, newest first |
| quotes | idx_quotes_org | (org_id, created_at DESC) | Tenant quote history |
| quotes | idx_quotes_shipment | (shipment_id) | Quotes belonging to a shipment |
| classification_attempts | idx_class_org | (org_id, created_at DESC) | Override analysis for prompt tuning |

No index on `tariff_versions.published` — the table holds tens of rows.

## 5. Migration Plan
| Migration File | Description | Reversible |
|---|---|---|
| 0001_extensions.sql | Enable pgcrypto, vector | Yes |
| 0002_tenancy.sql | organizations, memberships | Yes |
| 0003_tariff.sql | tariff_versions, tariff_lines, preferential_rates | Yes |
| 0004_operational.sql | shipments, quotes, classification_attempts | Yes |
| 0005_indexes.sql | All indexes from section 4 | Yes |

## 6. Access Patterns
| Use Case | Query Pattern | Index Coverage |
|---|---|---|
| Duty lookup for a quote | SELECT by (version_id, hs_code) | idx_tariff_lines_lookup |
| Preferential rate resolution | SELECT by (tariff_line_id, agreement) | idx_pref_line_agreement |
| HS candidate search | ORDER BY embedding <=> query LIMIT 10 | idx_tariff_lines_embedding |
| HS code autocomplete | WHERE hs_code LIKE '8703%' | idx_tariff_lines_hs_prefix |
| Shipment list | WHERE org_id = ? ORDER BY created_at DESC | idx_shipments_org |
| Quote history | WHERE org_id = ? ORDER BY created_at DESC | idx_quotes_org |

## 7. Sensitive Data
- **Columns requiring encryption**: none beyond Neon's at-rest default. Per the security baseline, the sensitive unit is the whole shipment row, so column encryption would break queryability for no real gain.
- **Row-level security**: **not** enabled in Postgres. Isolation is enforced in the repository layer (ADR-2) because Clerk identity lives in the app, not in a Postgres role. This is a deliberate trade-off and it makes the repository layer security-critical — hence the lint ban plus the isolation integration test in the test strategy.
- **Hard delete on org deletion**: `ON DELETE CASCADE` from `organizations` covers every tenant table.
