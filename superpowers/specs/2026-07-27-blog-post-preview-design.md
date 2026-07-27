# Spec — Blog post preview via preview token (cozycorner + web.admin + DB)

**Date:** 2026-07-27
**Projects:** cozycorner (frontend + DB migration), web.admin (admin UI). Spans 3
concerns: DB schema, public site, admin panel.
**Status:** design, awaiting implementation plan

## Goal

Let an editor view a blog post on the **real frontend** before publishing it, via a
shareable link, **without** the post appearing in the blog grid. Publishing works
exactly as today; only publishing puts a post in the grid.

## The constraint that shapes this

A draft (`is_published = false`) is unreadable by the public site:

- cozycorner uses the **publishable/anon** Supabase key. RLS grants anon only published
  rows, and `lib/posts.ts` additionally applies `.eq("is_published", true)` on the post
  (`fetchPostBySlug`, lines 41–53) **and** on every section (`fetchPostSections`, lines
  101–118).
- So a plain `/blog/<slug>` URL for a draft is a 404, no matter what the client does.

Therefore a draft preview **must bypass RLS for that one request**, server-side. We do
this with a per-post secret token and cozycorner's existing service-role client — **not**
by changing the visibility model. `is_published`, the grid query, and
`generateStaticParams` are left untouched, so nothing leaks into the grid.

Why a DB token (not an HMAC signature from the admin): web.admin is a client-side Vite
SPA and **must never hold a signing secret or service-role key** (project rule). A
random token stored on the row is a capability the admin simply **reads** via its
existing admin RLS; nothing sensitive ships in the admin bundle, and access is revocable
by regenerating the token.

## Current state

- **DB / cozycorner** owns schema via `cozycorner/supabase/migrations/` (single source
  of truth for the whole workspace). `posts` has `is_published`, `published_at`, `slug`
  (DB-trigger generated), `post_type`.
- **cozycorner**
  - `app/blog/page.tsx` → grid via `fetchPosts` (`.eq is_published` + `.eq post_type
    'blog'`).
  - `app/blog/[slug]/page.tsx` → `revalidate = 60`, `generateStaticParams` from
    `fetchAllPostSlugs`, post via `fetchPostBySlug`, sections via `fetchPostSections`;
    `notFound()` when null.
  - `lib/posts.ts` — the fetchers above (all filter `is_published = true`).
  - `lib/supabase/` — anon/publishable clients; **`lib/supabase/admin.ts`** holds a
    **service-role** client used today only by the newsletter action (server-only).
  - `lib/types.ts` — `Post` type; `next.config.ts` image `remotePatterns`.
  - Env: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` (public), plus the
    service-role secret consumed by `admin.ts` (never `NEXT_PUBLIC`).
- **web.admin**
  - `PostEditPage.tsx` — editor; publish state is the `is_published` `Switch`
    (lines ~350–363); `slug` shown read-only (line ~416).
  - `src/lib/posts.ts` — `listPosts` selects `id,title,published_at,is_published,
    folder_id`; `getPost` fetches post + sections.
  - `src/config/sites.ts` — `SiteConfig` (slug/label/projectUrl/anonKey/schema/bucket).
    **No frontend base URL anywhere in web.admin today.**
  - Project rule: **no service-role key, ever**; all access under admin session + RLS.

## Design

### 1. DB migration (in cozycorner/supabase/migrations/)

Add to `posts`:

```sql
alter table cozycorner.posts
  add column preview_token uuid not null default gen_random_uuid();
```

- Backfill is automatic via the default for existing rows.
- **RLS:** admin (authenticated, `is_admin()`) can already read the full row incl.
  `preview_token`. Anon must **not** be able to read `preview_token` (it would leak the
  capability). Confirm the anon `select` policy does not expose it, or restrict the
  column — the preview read happens with the **service-role** client, so anon never
  needs this column. (Verify current column-level grants during implementation.)
- After applying: sync `cozycorner/lib/types.ts` (`Post.preview_token`) and update
  `platform-docs/database/schema.md`. Mirror the migration file per workspace rule.

### 2. cozycorner preview route

Add a preview render path that reuses the live page's rendering but lifts the
post-level publish gate, reading with the **service-role** admin client and verifying
the token.

- **Route shape:** a dedicated segment is cleaner than a query param on the live route
  (keeps ISR/`generateStaticParams` on `/blog/[slug]` untouched and keeps preview
  dynamic). Proposed: `app/preview/blog/[slug]/page.tsx` with
  `export const dynamic = 'force-dynamic'` and `searchParams` carrying `token`
  (`/preview/blog/<slug>?token=<uuid>`).
- **Fetch:** new server-only helpers in `lib/posts.ts` (or a `lib/preview.ts`) using the
  `admin.ts` service-role client:
  - `fetchPostForPreview(adminClient, slug, token)` — select the post by slug **without**
    the `is_published` filter; return it **only if** `row.preview_token === token`, else
    `notFound()`.
  - Section rendering shows **all sections regardless of their `is_published`** so the
    author previews the full post, including sections still being staged. This is a
    preview-only relaxation: the live `/blog/[slug]` page keeps filtering sections by
    `is_published`. Implement with a preview-specific section fetch that drops the
    `.eq("is_published", true)` on both `post_text_sections` and
    `post_product_sections` (otherwise identical to `fetchPostSections`).
- **Isolation & safety:** the route is `noindex` (robots meta) and excluded from
  sitemap; it never affects the grid or static params. Invalid/missing token behaves like
  the live route (`notFound`).
- Follow the cozycorner Next 16 rule: check `node_modules/next/dist/docs/` before writing
  route code; content pages are RSC.

### 3. web.admin — preview link + frontend URL config

- **Config:** add an optional **`frontendUrl`** to `SiteConfig` in `src/config/sites.ts`
  (e.g. cozycorner prod URL). This is admin config, not DB.
- **Read the token:** extend the admin post fetch (`src/lib/posts.ts` `getPost` and/or a
  small `getPostPreviewToken`) to select `preview_token` (admin RLS already permits it).
  Add `preview_token` to the admin `Post` type.
- **UI:** in `PostEditPage.tsx`, add a **"Preview" / "Copy preview link"** control that
  builds `${site.frontendUrl}/preview/blog/${slug}?token=${preview_token}` and
  opens/copies it. Show it whenever a slug + token exist (i.e. saved posts); most useful
  for drafts but harmless for published. If `frontendUrl` is unset for a site, hide the
  control.
- No service-role key and no signing in web.admin — it only reads a value and builds a
  URL.

## Data flow

```
Editor (web.admin, admin session)
  reads posts.preview_token via admin RLS
  builds  {frontendUrl}/preview/blog/{slug}?token={uuid}
        │  (shareable link)
        ▼
cozycorner /preview/blog/[slug]  (force-dynamic, noindex)
  service-role admin client:
    SELECT post by slug (no is_published filter)
    if preview_token != token  -> notFound()
    else render (sections mirror live)
Grid / generateStaticParams / live /blog/[slug]  — unchanged, anon key, is_published gate
```

## Files affected

**cozycorner**
- `supabase/migrations/<new>.sql` — add `preview_token`.
- `lib/types.ts` — `Post.preview_token`.
- `lib/posts.ts` (or new `lib/preview.ts`) — `fetchPostForPreview` (+ section fetch for
  preview) using the service-role client.
- **New** `app/preview/blog/[slug]/page.tsx` — dynamic, noindex, token-checked render;
  reuse existing section components.
- `app/robots.ts` / `app/sitemap.ts` — ensure `/preview` is excluded/noindex.

**web.admin**
- `src/config/sites.ts` — add `frontendUrl` to `SiteConfig` + cozycorner value.
- `src/lib/posts.ts` — select/expose `preview_token`; `Post` type field.
- `src/features/posts/PostEditPage.tsx` — "Preview / Copy preview link" control.

**platform-docs**
- `database/schema.md` — document `preview_token`.
- `admin-panel/blog.md` and `sites/cozycorner.md` — document the preview flow.

## Edge cases / risks

- **Anyone with the link sees the draft** (intended — sharing with a client). Revoke by
  regenerating `preview_token`. A "regenerate preview link" button is **deferred** (not
  in this scope) — the column exists, so it can be added later without migration.
- Token must **not** be exposed to anon (column grants / rely on service-role-only read).
- Preview route must stay out of ISR/static params and search indexing.
- Preview shows **all** sections (ignores per-section `is_published`); the live page
  still filters them. A section fetched for preview may reference products/images not
  yet finalized — acceptable for a preview.
- Cross-repo change: DB migration and `lib/types.ts` sync are cozycorner's
  responsibility; web.admin only reads the column.

## Testing (per platform-docs/methodology/testing.md)

- **cozycorner** unit: `fetchPostForPreview` returns a draft when token matches; returns
  null/notFound on wrong/absent token; grid fetchers unchanged (still exclude drafts).
- **cozycorner** e2e (:3000, live Supabase): open preview URL for a draft → renders;
  wrong token → 404; draft absent from `/blog` grid.
- **web.admin** component: preview control builds the correct URL from
  `frontendUrl`/`slug`/`preview_token`; hidden when `frontendUrl` unset.
- Migration verified against live schema before frontend wiring.

## Out of scope

- Changing the publish model / grid semantics (untouched).
- Next.js draft-mode cookies (token route chosen instead).
- Previewing SEO posts specifically (same mechanism would extend, but scope is blog).
- In-admin rendered preview (explicitly not chosen).
