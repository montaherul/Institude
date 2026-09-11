# 06 — UI/UX Design System

## 1. Layouts

### Public site (per-tenant marketing)
- Theme: modern SaaS look — dark `#020617`/white cards, gradients `#60a5fa→#818cf8`, glass blur, rounded-3xl.
- Header: logo, nav (Home, About, Gallery, Academics dropdown, Events, FAQ, Contact), Call-us, Login.
- Sticky translucent header; smooth-scroll anchors for About/Events/FAQ.
- Mobile: toggle menu (off-canvas) mirroring nav.
- Footer: logo + description, socials, Contact Info, Quick Links, Legal links, copyright (year + vendor).
- Pages: Home gallery-carousel hero (banners with CTA buttons), Why-Us cards, About, counters, Faculty(teachers)+Staff, Events cards, Testimonials carousel, FAQ accordion; Gallery page w/ lightbox; Contact page w/ form + embedded Google Map; policy pages; event detail pages.
- (Reference shows placeholder content — seed polished copies.)

### Admin (back office)
- Metronic-style: fixed top bar (logo, search?, language switch EN/بn, user dropdown, logout), collapsible left sidebar with accordion menu groups & bullet items, content area with cards/tables.
- Dark/light/system theme toggle (reference has Light/Dark/System in user menu).
- Toast notifications (success/error) and inline validation messages.

## 2. Brand tokens

```
--primary: #2563eb;  --secondary/indigo: #4f46e5;
--text: #334155; --muted: #99a (slate-400); --bg: #020617;
radius: xl 12px / 2xl 16px / 3xl 24px / full;
shadow: soft (0 10px 40px rgba(2,6,23,.15)), elevated (0 20px 80px rgba(0,0,0,.45));
fonts: Inter (body), system fallback; weights 400/600/800.
gradient: linear(90deg, #60a5fa, #818cf8);  gradient-button: linear(90deg,#2563eb,#4f46e5).
```

Tenant-configurable overrides: primary/secondary/text/sidebar colors (General Settings → Theme) + header/footer/favicon images.

## 3. Component library (admin)

- **Cards** (`.card` + `.card-header` title/action), **Widgets** (KPI tiles: icon + value + label).
- **Tables:** Tabulator server-side grids (AJAX JSON → Controller → Service → SP; search, sort, pagination, export), empty-state ("No data found") illustration.
- **Forms:** labeled inputs, selects w/ Select2, date pickers, file uploads w/ preview, checkbox/switch toggles.
- **Cascading selects:** Class → Section → Group (JS, hitting `/sections-section-group-wise`, `/groups-class-section-wise`).
- **Modals:** confirmations ("Type Institute 1 to confirm"), detail views; delete buttons submit POST forms with the antiforgery token.
- **Print views:** mark sheets, certificates, ID cards, admit cards, barcodes, slips (dedicated print CSS, @media print).
- **Misc:** badges (status), avatars, toasts, breadcrumbs page titles.

## 4. Navigation information architecture

Grouped exactly as reference sidebar (see 05 §list) for parity. Each group header collapses (accordion). Top-level: Dashboard. Footer of sidebar: help/version.

## 5. Responsive

- Admin: content reflows to stacked cards < lg; sidebar off-canvas < lg; tables scroll horizontally.
- Public: hero stacks, grid→single columns, nav→hamburger.

## 6. Accessibility & UX

- Semantic headings hierarchy; labels on all inputs; focus rings; contrast ≥ 4.5:1; keyboard nav for menus.
- Loading spinners on async actions; disable submit while pending.
- Persist filters in URL query params for shareable reports.

## 7. i18n

- `Resources/en.json`, `Resources/bn.json` loaded via `IStringLocalizer` (or .resx); language switch posts to `/language/{en|bn}` (session).
- All UI strings via `@Localizer[...]`/`ITextLocalizer`; dates localized; Bangla numerals option.
- Reference exposes both EN/BN switchers.

## 8. Branding matrix

| Surface | Value |
|---|---|
| Public site title/footer | tenant `site_title`/`tagline` |
| Admin brand | "Mighty School" (fixed product brand) |
| Logos | header light/dark, footer light/dark, favicon from Institute Image Settings |
| Banner text | banner CRUD drives hero slides |

## 9. Payment/settings dynamic UI

General Settings is a tabbed mega-form: General, SEO, Social, Mail, SMS, Payment, App, Theme (+ Zoom/Meet tabs). Tabs validate independently and save via single `general_settings/update` endpoint (reference uses multiple forms on one route).