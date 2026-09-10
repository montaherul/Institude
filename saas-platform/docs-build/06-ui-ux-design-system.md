# 06 — UI/UX Design System

## 1. Product identity

One brand for the platform ("the dashboard"), three product faces. Shell chrome is identical; module areas carry their own iconography + color accent.

- School: indigo `#6366f1` · Rent: teal `#14b8a6` · Transport: amber `#f59e0b` · Core: slate `#0f172a`.

## 2. Layout

```
┌──────────── Top bar: logo · global search · tenant switcher · language · user menu ─────┐
│ sidebar (entitlement-driven) │  breadcrumbs │ module page area (cards/tables/forms)      │
│  Dashboard                    │  … returned module content (lazy-loaded)                │
│  School zone    (if entitled) │                                                         │
│  Rent zone      (if entitled) │                                                       
│  Transport zone (if entitled) │
│  Settings · Billing ◆ Core    │
└───────────────────────────────┴─────────────────────────────────────────────────────────┘
```

## 3. Tokens

```
--primary: linear-gradient(135deg,#6366f1,#22d3ee);  --bg: #f1f5f9; --surface:#fff; --ink:#0f172a;
radius 12–16px; shadow soft; font Inter; spacing 4px scale.
Dark mode: bg #0b1220, surface #111a2e (toggle Light/Dark/System).
Error #ef4444, success #22c55e, warn #f59e0b.  Badges for module+plan, status dots.
```

## 4. Entitlement-driven navigation

- Sidebar sections come from the **entitlements record**, not static config.
- Module sections lazy-load their JS/CSS bundles only when that module is entitled (bundle per module: `school.*`, `rent.*`, `transport.*`).
- Not-permitted menu items never render; server still blocks routes (defense in depth).
- "Plan" chip in top bar reflects current plan + active modules; upsell card in shell when modules available (`plan.modules` ⊃ `entitled.modules`).

## 5. Component library

- **Shell:** TopBar, Sidebar (grouped accordion), Breadcrumbs, CommandK global search, TenantSwitcher, UserMenu.
- **Data:** DataTable (search/sort/page/export CSV), StatCards (KPI tiles), SummaryCards (module-level), EmptyState.
- **Forms:** labeled inputs+errors, Select2-style cascades (class→section, property→unit, route→stop), date/month pickers, switches, file upload preview.
- **Modals:** confirm destructive ("Type environment name to confirm"), receipt/preview drawers, maintenance timeline.
- **Reports:** filter panels → tables → export (CSV/PDF), print stylesheets.
- **Statuses:** chips (active/inactive/due/paid/scheduled/etc.) with color mapping.

## 6. Module skins

| Module | Landing screen | Primary objects |
|---|---|---|
| School | Class dashboard (attendance %, fee due) | students, classes, attendance grid, fee cabinets |
| Rent | Rent dashboard (occupancy %, rent due) | properties/units grid, leases, invoices |
| Transport | Fleet dashboard (trips today, vehicles) | vehicles, routes map, trips, subscribers |

## 7. Onboarding flow (client acquisition)

1. `/pricing` (public) → CTA "Get started"
2. Tenant signup (org name/slug, owner)
3. Pick module checkbox cards (1/2/3) with per-module prices
4. Plan tier + billing period
5. Stripe payment → provision → first-run setup checklist per selected module

## 8. Responsive & a11y

- Reflow to stacked cards < md; sidebar off-canvas; tables horizontal scroll.
- Keyboard nav, focus rings, contrast AA, `aria` on menus/status.
- Loading spinners on async; submit-lock; URL-param filter persistence.

## 9. i18n

- `en` + `bn` json (Bangladesh-first market), locale switcher; module labels translatable; date/number locale-aware.

## 10. Empty & error states

- Empty: illustration + primary CTA (Add first student / Add property / Add vehicle).
- Module not entitled: never encountered in UI (menu hidden); 404 toast if stale URL handled gracefully.