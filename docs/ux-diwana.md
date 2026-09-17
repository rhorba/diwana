# UX Foundation: Diwana.ma
**PRD Reference**: docs/prd-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: UX Designer

## 1. User Personas (minimal — YAGNI)
| Persona | Role | Goal | Pain Point |
|---|---|---|---|
| **Karim** | Owner of an SME importing electronics from China and Turkey | Know the true cost before signing a PO | Finds out the duty rate after the goods are already on a ship |
| **Salma** | Transitaire / customs broker, 3-person firm | Produce a client cost quote fast enough to win the job | Rebuilds the same spreadsheet for every enquiry, unpaid |
| **Youssef** | Finance manager at a mid-size importer | Reconcile estimates against the actual DUM | Has no record of what was estimated or why |

## 2. Information Architecture / Site Map
```
[Diwana]
├── (public)
│   ├── /                      landing + free single-shipment calculator
│   ├── /calculateur           the funnel: calculate once, no account
│   └── /tarif/[hs]            SEO page per HS chapter/code
├── (auth)
│   ├── /sign-in, /sign-up
│   └── /org/select            org picker when user belongs to several
└── (app)
    ├── /dashboard             recent shipments + spend summary
    ├── /shipments             list, filter, compare
    │   ├── /new               create shipment
    │   └── /[id]              detail + quote history + export
    ├── /quotes/new            standalone quote (not yet attached to a shipment)
    ├── /tariff                HS code explorer
    └── /settings              members, roles, billing, language
```

The free `/calculateur` is deliberately part of the public tree. Per the PRD risk table, the calculator is commoditised — it is our acquisition funnel, not the product. The product begins at `/shipments`.

## 3. Core User Flows (top 3 journeys)

### Flow 1: First landed-cost calculation (anonymous → activated)
```
[Landing] → [Describe product in own words]
                     ↓
          [AI suggests HS codes + confidence]
                     ↓
          confidence ≥ threshold? ──No──> [Show candidates, force explicit pick]
                     │ Yes                        ↓
                     ↓                   [Link: request an RTC from ADII]
          [Pre-select top code, still editable]
                     ↓
          [Enter CIF value + origin country]
                     ↓
          [Breakdown: CIF → DI → PFI → TVA → Total]
                     ↓
          [Preferential rate available for this origin?] ──Yes──> [Banner: "You may save X MAD with a certificate of origin"]
                     ↓
          [Save this shipment] → [Sign-up prompt] → [Account created, quote preserved]
```

### Flow 2: Recurring importer logs a shipment (the retention loop)
```
[Dashboard] → [New shipment] → [Reference + supplier + origin]
                                        ↓
                          [Add line item → classify → quote]
                                        ↓
                          [Repeat for each line item]
                                        ↓
                          [Shipment total] → [Export PDF for finance]
                                        ↓
                          [Later: "Re-run against current tariffs"]
                                        ↓
                          [New quote created; old one kept side-by-side for comparison]
```

### Flow 3: Broker produces a client quote under time pressure
```
[Quotes/new] → [Paste product description] → [Classify] → [CIF + origin]
                                                              ↓
                                              [Breakdown in under 60 seconds]
                                                              ↓
                                              [Export branded PDF] → [Send to client]
```

## 4. Key Screen Wireframes (text-based)

### Screen: Calculator / Quote builder (the core screen)
```
┌──────────────────────────────────────────────────────┐
│ Diwana            Shipments  Tariff  Settings   [FR▾]│
├──────────────────────────────────────────────────────┤
│  1. What are you importing?                          │
│  ┌────────────────────────────────────────────────┐  │
│  │ e.g. "câbles USB-C en cuivre pour téléphone"   │  │
│  └────────────────────────────────────────────────┘  │
│                                    [ Classifier → ]  │
│                                                      │
│  Suggested codes                                     │
│  ● 8544.42.90.00   Câbles avec connecteurs    92% ✓  │
│  ○ 8544.49.00.00   Autres conducteurs         61%    │
│  ○ 8517.62.00.00   Appareils de transmission  40%    │
│           [ Not sure? Request a binding ruling (RTC) ]│
│                                                      │
│  2. Value & origin                                   │
│  CIF value [ 120 000 ] [MAD▾]   Origin [ Turkey ▾ ]  │
│                                                      │
│  ┌── Breakdown ─────────────────────────────────┐    │
│  │ CIF value                        120 000,00  │    │
│  │ Import duty (DI)      2,5% ⓘ       3 000,00  │    │
│  │   └ preferential, Turkey agreement           │    │
│  │ PFI                  0,25%           300,00  │    │
│  │ TVA                    20%        24 660,00  │    │
│  │ ───────────────────────────────────────────  │    │
│  │ Total landed cost                147 960,00  │    │
│  │ Duty & taxes                      27 960,00  │    │
│  └──────────────────────────────────────────────┘    │
│  💡 Without a certificate of origin: 156 360,00 MAD  │
│     You save 8 400,00 MAD by claiming it.            │
│                                                      │
│  ⚠ Indicative estimate based on the ADII tariff of   │
│    2026-09-15. Not a binding ADII ruling.            │
│                                                      │
│         [ Save to shipment ]   [ Export PDF ]        │
└──────────────────────────────────────────────────────┘
```

### Screen: Shipments list
```
┌──────────────────────────────────────────────────────┐
│ Shipments                          [ + New shipment ]│
├──────────────────────────────────────────────────────┤
│ Ref        Supplier      Origin  Lines  Total    ⋮   │
│ IMP-0042   Shenzhen Ltd  CN        4    412 300  ⋮   │
│ IMP-0041   Arçelik       TR        1    147 960  ⋮   │
│ IMP-0040   Bosch GmbH    DE        7    883 120  ⋮   │
│                                                      │
│ (empty state: "No shipments yet. Your first quote    │
│  takes about a minute." [ Create one ])              │
└──────────────────────────────────────────────────────┘
```

## 5. Screen States
| Screen | Empty State | Loading | Error | Success |
|---|---|---|---|---|
| Calculator | Prompt with a worked example placeholder | Skeleton on the candidate list; button shows spinner | "Classification unavailable — enter the HS code manually" with a manual field revealed | Breakdown animates in, total emphasised |
| Candidate list | "No confident match. Enter an HS code or request an RTC." | Three skeleton rows | Same as above, never a dead end | Top candidate pre-selected but focusable |
| Shipments | "No shipments yet. Your first quote takes about a minute." + CTA | Table skeleton, 5 rows | Retry banner, list stays visible if cached | New row highlights briefly |
| Shipment detail | "No quotes on this shipment yet." | Skeleton breakdown card | Inline error, previously saved quotes still readable | Toast + quote appended to history |
| Export | — | Button disabled, "Preparing PDF…" | "Export failed, try again" — never a blank download | File downloads, toast confirms |
| Tariff explorer | "Search an HS code or describe a product." | Skeleton rows | "Search unavailable" | Results with the `as_of` date shown |

## 6. UX Principles for this product
1. **Never dead-end on classification.** Every low-confidence path offers manual entry plus the RTC link. The user must always be able to finish.
2. **Show the `as_of` date on every number.** Trust in this product is trust in the data's freshness.
3. **Surface the preferential saving unprompted.** It is the single moment where the product visibly pays for itself.
4. **Arabic is a first-class language, not a mirrored afterthought.** Real translated copy — the specific gap flagged in the Bina project.
