# Newsletter footer polish + Subscribers admin section — design

> Date: 2026-07-23 · Projects: `cozycorner` (footer), `web.admin` (new section)

Two coordinated changes from one request:
1. cozycorner footer — make the Subscribe button match the "View on Amazon" CTA and
   narrow the "Join our newsletter" column by 20% on desktop.
2. web.admin — a new top-level **Subscribers** section (list + CSV export + delete)
   reading the existing `cozycorner.subscribers` table.

No DB migration, no RLS change: the table already exists (migration 0025) and RLS
`Admin manage subscribers` (`is_admin()`) already lets an admin read and delete rows.
Insert stays service-role-only (site form + Turnstile) — we are **not** importing.

## Part A — cozycorner footer

Files: `cozycorner/components/NewsletterForm.tsx`, `cozycorner/components/Footer.module.css`.

1. **Subscribe button size.** The button already reuses the shared `Button`; it renders
   at default `size="md"`, while "View on Amazon" (product page) uses `size="lg"`
   (`padding var(--space-3) var(--space-6)`, `font-size var(--fs-base)`). Fix: pass
   `size="lg"` to the Subscribe `Button`. Stays full-width via `.submit { width: 100% }`.
2. **Column width.** `Footer.module.css .top` grid `grid-template-columns: 1fr minmax(0, 400px)`
   → `1fr minmax(0, 320px)` (400 × 0.8 = 20% narrower). The `max-width: 720px` media query
   that stacks the footer on mobile is unchanged → desktop-only effect.

No logic, data flow, or copy touched. Turnstile → Server Action → `subscribers` insert stays as-is.

## Part B — web.admin Subscribers section

Follows the existing per-section pattern (data module in `src/lib`, page in `src/features`,
TanStack Query, sonner toasts, shadcn UI). English UI text.

### Data — `src/lib/subscribers.ts`
- `type Subscriber = { id: string; email: string; created_at: string }`.
- `listSubscribers(site)` → `getDb(site).from('subscribers').select('id,email,created_at').order('created_at', { ascending: false })`.
- `deleteSubscriber(site, id)` → `.delete().eq('id', id)`.
- Style matches `src/lib/products.ts` (throw on error, return `data ?? []`).

### CSV export — `src/lib/csv.ts` (net-new, generic)
- `toCsv(rows, columns)` builds CSV text with a header row and RFC-4180 escaping
  (wrap in quotes + double any `"` when a value contains `,`, `"`, `\n`, or `\r`).
- `downloadCsv(filename, text)` triggers a client download via a `Blob` + object URL
  (`URL.createObjectURL`, temporary `<a>`, `revokeObjectURL`). No new dependency.
- Has a unit test (`csv.test.ts`) — pure, easy to cover; escaping is the risky part.

### Page — `src/features/subscribers/SubscribersPage.tsx`
- `useQuery(['subscribers', site.slug], () => listSubscribers(site))`.
- **Header row:** `<h1>Subscribers</h1>` + total count (`N` or `N / total` when filtered) +
  right-aligned **Export CSV** button (primary; disabled when list empty).
- **Search:** client-side email filter with `useDeferredValue` (spinner-in-input pattern,
  list dims to 60% while filtering) — same UX as other sections.
- **Table:** the shadcn `Table` component (first use in the app). Columns:
  **Email · Subscribed (`formatDate`) · Actions** (per-row trash icon → confirm → delete).
- **Bulk delete:** a **Select** toggle (shown when list non-empty) enters selection mode:
  a row checkbox column appears and the header row is replaced by a minimal selection bar
  (selected count + Cancel + `BulkDeleteButton`). Reuses `useBulkDelete` and `BulkDeleteButton`
  from `src/features/folders/` (both folder-independent). `SelectionBar` is NOT reused (it is
  folder/move-coupled); the bar here is a small local element.
- **Delete confirm:** shadcn `AlertDialog` — per-row and bulk both confirm ("Delete N
  subscriber(s)? This action cannot be undone."), then invalidate `['subscribers', site.slug]`
  + sonner toast (handled inside `useBulkDelete` / a small per-row mutation).
- **States:** loading → skeleton rows; empty → dashed "No subscribers yet."; filtered-empty →
  "No subscribers match your search."; query error → destructive banner (same as other pages).
- **Export CSV** exports the **full** list (ignores the search box) as
  `cozycorner-subscribers-YYYY-MM-DD.csv` with columns `email,subscribed_at` (created_at ISO).

### Wiring
- `src/components/SiteLayout.tsx` — add `{ to: 'subscribers', label: 'Subscribers', icon: Mail }`
  to `NAV_ITEMS` (placed **last**, after SEO Posts).
- `src/App.tsx` — add `<Route path="subscribers" element={<SubscribersPage />} />` under `/:siteSlug`.

### Multi-site note
The nav item shows for every site (same as all sections), but only `cozycorner` has a
`subscribers` table today. Other sites would surface a load error until they add the table —
acceptable: only cozycorner exists in the registry now.

## Docs to update
- `platform-docs/admin-panel/components.md` — nav map + nav-menu list (§2, §3.5) gain Subscribers.
- `platform-docs/admin-panel/subscribers.md` — new short section doc (data contract, operations, UI).
- `platform-docs/database/schema.md` — note the admin now reads/deletes `subscribers` + CSV export.

## Verification
- cozycorner: `npm run build` + `npm run lint`.
- web.admin: `npm run build` + `npm run lint` + `npm test` (adds `csv.test.ts`).

## Out of scope
CSV import (no admin insert RLS), collecting name/IP (GDPR-minimal table stays email+date),
per-site conditional nav, pagination (all rows fetched at once, matching every other section).
