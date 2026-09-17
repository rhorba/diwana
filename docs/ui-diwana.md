# UI Foundation: Diwana.ma
**UX Reference**: docs/ux-diwana.md
**Version**: 1.0 | **Date**: 2026-09-17 | **Author**: UI Designer

## 1. Design Approach
- **Strategy**: Tailwind CSS + shadcn/ui, with a custom token layer on top.
- **Rationale**: YAGNI says use the framework. But this product's whole job is making people trust a number, so the numeric presentation — tabular figures, the breakdown card, the savings banner — gets deliberate custom treatment rather than default component styling. Everything else stays stock shadcn.
- **Anti-default posture**: no purple-gradient SaaS look. Diwana should read like a customs document that happens to be well designed — dense, precise, calm. The reference point is a well-set financial statement, not a startup landing page.

## 2. Design Tokens
```css
:root {
  /* Colors — "ink and stamp": paper ground, deep navy ink, one hot accent */
  --color-primary:      #1B3A57;  /* deep customs-navy, headers + primary CTA */
  --color-primary-fg:   #FFFFFF;
  --color-accent:       #C8501E;  /* terracotta stamp — savings, alerts, emphasis */
  --color-background:   #FAF8F4;  /* warm paper, not clinical white */
  --color-surface:      #FFFFFF;
  --color-border:       #E2DCD1;
  --color-success:      #2D6A4F;
  --color-error:        #A02C2C;
  --color-warning:      #B67A1E;  /* the disclaimer band */
  --color-text:         #1A1A18;
  --color-text-muted:   #6B6862;

  /* Typography */
  --font-sans:    "Inter", system-ui, sans-serif;
  --font-arabic:  "IBM Plex Sans Arabic", "Noto Sans Arabic", sans-serif;
  --font-mono:    "JetBrains Mono", ui-monospace, monospace; /* HS codes + figures */

  --font-size-xs:  0.75rem;   --font-size-sm: 0.875rem;
  --font-size-md:  1rem;      --font-size-lg: 1.25rem;
  --font-size-xl:  1.75rem;   --font-size-2xl: 2.5rem;

  /* Spacing — 4px base */
  --spacing-xs: 0.25rem;  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;     --spacing-lg: 1.5rem;  --spacing-xl: 2.5rem;

  --radius: 6px;          /* restrained; this is a document, not a toy */
}
```

**Dark mode**: tokens redefined under `@media (prefers-color-scheme: dark)` and `[data-theme="dark"]`. Paper ground becomes `#16161A`, ink inverts, terracotta lightens to `#E0703F` to hold contrast.

**Numerals**: all monetary figures use `font-variant-numeric: tabular-nums` so columns align. HS codes are set in mono — they are identifiers, not prose.

## 3. Component Inventory
| Component | Reuse Existing | Build New | Notes |
|---|---|---|---|
| Button, Input, Select, Dialog, Toast, Table, Skeleton, Tabs | shadcn/ui | No | Stock, retokenised |
| Combobox (HS search) | shadcn Command | No | Async, debounced |
| **BreakdownCard** | No | **Yes** | The product's signature component: line items, rules, emphasised total, tabular-nums |
| **ConfidenceBadge** | No | **Yes** | % + colour band; below threshold it renders as a warning, never as a quiet grey |
| **SavingsBanner** | No | **Yes** | Terracotta accent; the preferential-origin moment |
| **DisclaimerBand** | No | **Yes** | Persistent, warning-toned, carries the `as_of` date — appears on every quote surface |
| **AsOfStamp** | No | **Yes** | Small mono date chip, reused inline anywhere a rate appears |
| LanguageSwitcher | No | Yes | FR / AR / EN, sets `dir` on `<html>` |

## 4. Responsive Breakpoints
| Breakpoint | Width | Layout Notes |
|---|---|---|
| Mobile | < 768px | Single column; breakdown becomes a stacked definition list, not a squeezed table; sticky total bar at the bottom |
| Tablet | 768–1024px | Two column: inputs left, breakdown right |
| Desktop | > 1024px | Three zones: nav rail, input column, sticky breakdown panel |

Shipments table on mobile collapses to cards — a horizontally scrolling financial table is unusable on a phone.

## 5. RTL / Arabic Strategy
This is called out separately because Bina got it mechanically right and substantively wrong.
- `dir="rtl"` on `<html>` for `ar`, with **fully translated copy** — no French strings leaking through. The DoD for any Arabic screen is a native-reader pass, not a layout screenshot.
- Logical CSS properties throughout (`margin-inline-start`, never `margin-left`).
- **Numbers and HS codes stay LTR** inside RTL text — customs figures are read left-to-right even in Arabic documents. Wrap them in `<bdi>`.
- Arabic font stack is separate from Latin; Inter has no Arabic coverage.

## 6. Accessibility Baseline
- Colour contrast AA minimum: navy `#1B3A57` on paper `#FAF8F4` = 11.8:1; terracotta `#C8501E` on paper = 4.7:1 — passes for normal text.
- **Never colour alone**: confidence badges pair colour with a numeric percentage and a text label; the savings banner carries an icon and words.
- Visible focus indicators on all interactive elements; 2px accent outline with offset.
- Semantic HTML first — the breakdown is a real `<dl>` or `<table>`, so screen readers announce label/value pairs correctly.
- Every form input has a real `<label>`; the classify field gets a described-by hint with the worked example.
