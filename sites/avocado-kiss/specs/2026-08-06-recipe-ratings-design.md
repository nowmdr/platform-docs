# Recipe Ratings — Design Spec

**Project:** avocado.kiss
**Date:** 2026-08-06
**Status:** Approved for planning
**Owner spec:** this document is the source of truth for the feature; the live
DB schema (via migrations) and `lib/types.ts` must be kept in sync with it.

---

## 1. Goal

Let anonymous visitors rate a recipe (1–5 stars) and show an aggregate rating on
the recipe page. Two surfaces:

1. **Read-only badge** under the recipe title — `4.3 ★★★★☆ 3 Ratings`.
2. **Interactive rating section** at the end of the recipe — a 5-star picker; on
   submit it thanks the user and shows the fresh aggregate.

The **displayed** average is floored so it never drops below a per-recipe minimum
(default **4.1**). The **real** votes are always recorded truthfully; the floor
only affects presentation. Admins (web.admin, Phase B — not built here) must be
able to seed and adjust the numbers later, so the data model is admin-friendly.

## 2. Non-goals

- No user accounts, no login, no comments/reviews text — a numeric star only.
- No web.admin UI in this change. We only ship the DB columns/tables and the
  public site behavior so the future admin can drive them.
- No perfect anti-fraud. localStorage dedup is a UX aid; bot spam is mitigated by
  a Cloudflare Turnstile challenge, not eliminated.
- Rating is **only** on recipe detail pages — not on cards, category grids, home.

## 3. Rating math

Each recipe carries two independent buckets:

| Bucket | Columns | Written by |
|---|---|---|
| Real votes | `ratings_count`, `ratings_sum` | the site (via the rate endpoint) — read-only to admins |
| Admin baseline (seed) | `seed_count`, `seed_sum` | admin only (Phase B) |

Plus a per-recipe floor `min_display_rating` (default `4.1`, admin-adjustable).

**Displayed values** (computed in the DB as generated columns so the site and the
future admin share one formula):

```
display_count = seed_count + ratings_count
display_avg   = CASE
                  WHEN display_count = 0 THEN min_display_rating
                  ELSE GREATEST(
                    min_display_rating,
                    ROUND((seed_sum + ratings_sum)::numeric / display_count, 1)
                  )
                END
```

- A user's own vote **does** move `display_avg` upward once the true weighted
  average exceeds the floor; below the floor it is pinned to `min_display_rating`.
- `seed_sum` is a sum, not an average: a seed of "3 ratings averaging 4.7" is
  stored as `seed_count = 3`, `seed_sum = 14` (≈4.67 → displays 4.7). Admin tools
  will expose this as count + average and convert.
- `display_avg` is `numeric(2,1)` in effect (one decimal). The badge renders it to
  one decimal (`4.3`). Stars may render a fractional fill (see §7).
- With `display_count = 0` the badge shows `4.1` and `0 Ratings`. This is
  intentional and expected to be avoided in practice by admins seeding a recipe.

## 4. Data model — migration `0012_recipe_ratings.sql`

Schema: `avocado_kiss`. Follow the existing migration style (idempotent
`if not exists`, `set search_path = ''` in functions, RLS conventions).

### 4.1 Columns added to `avocado_kiss.recipes`

```sql
alter table avocado_kiss.recipes
  add column if not exists ratings_count      integer      not null default 0,
  add column if not exists ratings_sum        integer      not null default 0,
  add column if not exists seed_count         integer      not null default 0,
  add column if not exists seed_sum           integer      not null default 0,
  add column if not exists min_display_rating numeric(2,1) not null default 4.1;

-- Generated (stored) display columns — single source of the display formula.
alter table avocado_kiss.recipes
  add column if not exists display_count integer
    generated always as (seed_count + ratings_count) stored,
  add column if not exists display_avg numeric(2,1)
    generated always as (
      case when (seed_count + ratings_count) = 0 then min_display_rating
      else greatest(
        min_display_rating,
        round((seed_sum + ratings_sum)::numeric / (seed_count + ratings_count), 1)
      ) end
    ) stored;

alter table avocado_kiss.recipes
  add constraint recipes_min_display_rating_range
    check (min_display_rating >= 1 and min_display_rating <= 5);
```

> Note: `display_avg`/`display_count` reference only base columns, so stored
> generated columns are valid. If the Postgres version rejects the nested
> expression in a stored generated column, fall back to a view
> `avocado_kiss.recipes_with_rating` exposing the same two computed fields and
> point the data loader at it — the site contract (`Recipe.display_avg`,
> `Recipe.display_count`) stays identical either way. Decide at implementation
> time based on what the DB accepts; document the choice in the migration.

### 4.2 Votes table

```sql
create table if not exists avocado_kiss.recipe_ratings (
  id         uuid primary key default gen_random_uuid(),
  recipe_id  uuid not null references avocado_kiss.recipes(id) on delete cascade,
  value      smallint not null check (value between 1 and 5),
  client_id  text,                       -- localStorage UUID, for reference/analytics
  created_at timestamptz not null default now()
);
create index if not exists recipe_ratings_recipe_idx
  on avocado_kiss.recipe_ratings (recipe_id, created_at desc);
```

There is **no** unique constraint on `(recipe_id, client_id)` — per the chosen
design, dedup is a client-side (localStorage) UX aid, not a server guarantee.
The table exists for analytics, audit, and future recompute in admin.

### 4.3 RLS

- `recipe_ratings`: enable RLS. **No** public read policy, **no** anon
  insert/update policy. The only writer is the rate RPC (security definer). Admin
  read/write follows the existing `is_admin()` whitelist pattern (add
  `recipe_ratings` to the admin-write loop and add an admin select policy).
- The new `recipes` columns inherit the table's existing RLS (public reads
  published rows — the display columns come along for free).

### 4.4 Write RPC — `avocado_kiss.rate_recipe`

```sql
create or replace function avocado_kiss.rate_recipe(
  p_slug text, p_value integer, p_client_id text
) returns table (display_avg numeric, display_count integer)
language plpgsql security definer set search_path = '' as $$
declare v_id uuid;
begin
  if p_value < 1 or p_value > 5 then
    raise exception 'value out of range';
  end if;
  select id into v_id from avocado_kiss.recipes
    where slug = p_slug and is_published = true;
  if v_id is null then
    raise exception 'recipe not found';
  end if;
  insert into avocado_kiss.recipe_ratings (recipe_id, value, client_id)
    values (v_id, p_value, p_client_id);
  update avocado_kiss.recipes
    set ratings_count = ratings_count + 1,
        ratings_sum   = ratings_sum + p_value
    where id = v_id;
  return query
    select r.display_avg, r.display_count
    from avocado_kiss.recipes r where r.id = v_id;
end $$;
```

- `EXECUTE` granted **only** to the `service_role` (NOT `anon`, NOT
  `authenticated`) — the anon key is public, so exposing this RPC to `anon` would
  let bots bypass Turnstile. The Route Handler calls it with the service-role key.

## 5. Write path — Next.js Route Handler + Turnstile

New file `app/api/recipes/[slug]/rate/route.ts` — the **first mutation** in this
repo (the app has been read-only until now).

Flow:

```
Browser (RatingSection, 'use client')
  → renders invisible Cloudflare Turnstile widget → obtains token
  → POST /api/recipes/[slug]/rate  { token, value, clientId }
Route Handler (server, Node runtime):
  1. Parse + validate body: value ∈ 1..5, token present.
  2. Verify token: POST challenges.cloudflare.com/turnstile/v0/siteverify
     with { secret: TURNSTILE_SECRET_KEY, response: token, remoteip }.
     On failure → 403.
  3. Create a service-role Supabase client (SUPABASE_SECRET_KEY, schema avocado_kiss).
  4. supabase.rpc('rate_recipe', { p_slug: slug, p_value: value, p_client_id: clientId })
  5. Return 200 { displayAvg, displayCount } (or 4xx with a safe error).
```

- Route Handler config: `export const runtime = 'nodejs'` (needs the secret key,
  never Edge with public env). It is a dynamic route — not part of SSG/ISR.
- **Before writing it, read the Route Handlers guide in
  `node_modules/next/dist/docs/`** — this Next.js version differs from training
  data (per AGENTS.md). In Next 16, `params` is a `Promise` (see the recipe page).
- Rate endpoint must NOT trust `value`/`count` from the client for display — it
  returns the DB-computed `display_avg`/`display_count`.

### 5.1 Config / env changes

| Env var | Scope | Purpose |
|---|---|---|
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | public | Turnstile widget on the client |
| `TURNSTILE_SECRET_KEY` | server-only (Vercel) | siteverify |
| `SUPABASE_SECRET_KEY` | server-only (Vercel) | service-role writes from the Route Handler |

**Policy change:** this introduces a server-only Supabase secret into the repo's
deployment, relaxing the current AGENTS.md rule "publishable only — never
`sb_secret_…`". Update:
- `avocado.kiss/AGENTS.md` env line → note the secret is allowed **server-only**
  (Route Handlers), never `NEXT_PUBLIC_`, never shipped to the browser.
- `.env.example` → add the three new vars with placeholder values + a comment that
  the secret keys are server-only.
- `platform-docs/methodology/coding-standards.md` if it restates the key rule.

Provider: **Cloudflare Turnstile** (free, invisible/managed, no PII). hCaptcha is
a drop-in alternative if preferred later — same verify-token shape.

## 6. Client dedup (localStorage)

Helper `lib/rating-client.ts` (browser-only):

- `avk_client_id` — a UUID generated once per browser (`crypto.randomUUID()`),
  persisted; sent as `clientId` with every vote (stored in `recipe_ratings`).
- `avk_rated_<recipeId>` — set to the value the user picked after a successful
  vote. On mount, `RatingSection` reads it: if present, render the "You rated N★"
  state and hide the picker.
- This is defeatable (clearing storage, incognito) — acceptable; server-side bot
  spam is handled by Turnstile.
- SSR-safe: guard all `window`/`localStorage` access; the section renders the
  picker state during SSR and reconciles on mount (avoid hydration mismatch by
  gating the "already rated" view behind a mounted flag).

## 7. UI

### 7.1 `RatingBadge` (RSC, presentational)

- Input: `avg: number`, `count: number`.
- Renders `4.3` + a 5-star row with fractional fill for the decimal + `N Ratings`
  (`1 Rating` singular). Fractional fill via a CSS width-clip overlay (a filled
  star row clipped to `avg/5 * 100%` over an empty row) — no half-star assets.
- Placement: inside the recipe `<header>`, directly under `<h1>` and above/near
  the `.meta` (Serves / Cook time) row, matching the reference screenshot.
- New `StarIcon` in `components/icons/` (filled + outline variants, or one path +
  CSS fill), consistent with existing icon components.

### 7.2 `RatingSection` (`'use client'`)

- Placement: a new `<section>` appended after the Method section, inside
  `<article>` (before it closes).
- States:
  - **idle**: heading ("Rate this recipe"), 5 empty stars with hover preview +
    keyboard support (radio-group semantics, arrow keys, Enter/Space).
  - **submitting**: disabled stars, spinner/label.
  - **done**: "Thanks! You rated N★" + the fresh `display_avg` / `display_count`.
  - **already-rated** (from localStorage on mount): same as done, using the stored
    value; no new request.
  - **error**: inline message, picker re-enabled to retry.
- On click: set `avk_rated_<id>`, POST to the rate endpoint with the Turnstile
  token + `clientId`, then on 200 update the visible aggregate from the response.
  On Turnstile/verify failure show the error state and do NOT persist the flag.
- Styling: CSS Module, design tokens only (no hardcoded colors/sizes — new value →
  new token in `globals.css`). Match the page's editorial tone.
- Accessibility: the picker is a labeled radiogroup; the badge stars are
  `aria-hidden` with an adjacent visually-hidden text label
  (`"Rated 4.3 out of 5, 3 ratings"`).

### 7.3 SEO — structured data

Add `Recipe` JSON-LD `aggregateRating` to the recipe page metadata using the
**displayed** values (`display_avg`, `display_count`) — only when
`display_count > 0`. Keep it consistent with what's shown on screen (Google
requires the visible rating to match the markup). Wire it into the existing
`generateMetadata`/page (a `<script type="application/ld+json">` in the page body
is fine for a Server Component).

## 8. Data layer (`lib`)

- Extend the `Recipe` type in `lib/types.ts` with the new fields:
  `ratings_count`, `ratings_sum`, `seed_count`, `seed_sum`,
  `min_display_rating`, `display_count`, `display_avg` (numbers).
- `fetchRecipeBySlug` uses `select("*")`, so the new columns arrive automatically.
  Verify the generated columns are returned by `select *`; if a view fallback
  (§4.1) is used, point the loader at the view.
- Add a small pure helper if any client formatting is shared
  (`formatRatingCount(n)` → `"1 Rating"` / `"N Ratings"`). Keep display math in the
  DB — the client only formats.

## 9. Admin compatibility (Phase B — not built now)

The design keeps everything admin-drivable without any admin code in this change:

- `seed_count`, `seed_sum`, `min_display_rating` are plain editable columns on
  `recipes`; the future admin edits them via its authenticated `is_admin()` write
  path (already the pattern for all recipe fields).
- `ratings_count` / `ratings_sum` are read-only aggregates the admin can display;
  `recipe_ratings` gives per-vote history for analytics / recompute.
- `display_avg` / `display_count` are computed in the DB, so the admin preview and
  the live site always agree.

## 10. Testing (per `platform-docs/methodology/testing.md`)

Unit / component (Vitest + RTL) — always:

- Rating math helper / DB formula expectations: floor pinning (avg below floor →
  floor), avg above floor passes through, `display_count = 0` → floor + 0, seed +
  real combine correctly, one-decimal rounding, singular "1 Rating".
- `RatingBadge`: renders avg, count, fractional star fill, singular/plural, a11y
  label.
- `RatingSection`: idle → submitting → done transition (mock fetch), "already
  rated" from seeded localStorage skips the request, error state on failed
  response, keyboard selection. Mock `crypto.randomUUID`, `localStorage`, and the
  Turnstile widget.
- Route Handler: mock Turnstile verify + service client — 200 on valid, 403 on bad
  token, 400 on out-of-range value, 404 on unknown/unpublished slug. Assert it
  never exposes the secret and returns DB-computed values.

E2E (Playwright, :3100, live Supabase) — propose but only build if the user asks
(interactive user flow): open a recipe, submit a rating, see the thank-you +
updated count; reload → "already rated" persists. Requires a Turnstile test key
(Cloudflare provides always-pass test keys) and cleanup of test votes.

## 11. Edge cases & decisions

- **Double submit / race**: disable the picker while submitting; the localStorage
  flag is set optimistically but only *confirmed* (kept) on 200 — on error it is
  cleared so the user can retry.
- **JS disabled / Turnstile blocked**: the section degrades to the read-only
  aggregate (picker requires JS + a token; without a token the POST is rejected).
- **Value tampering**: server clamps to 1..5 and ignores any client-sent
  aggregate; display always comes from the DB.
- **ISR staleness**: the badge is baked at build/ISR (`revalidate = 60`), so a
  brand-new vote by another user may lag up to 60s on the badge. The voting user
  sees their fresh number immediately from the POST response. Acceptable.
- **min_display_rating** stored as `numeric(2,1)`, constrained 1..5.

## 12. File change list

New:
- `avocado.kiss/supabase/migrations/0012_recipe_ratings.sql`
- `avocado.kiss/app/api/recipes/[slug]/rate/route.ts`
- `avocado.kiss/components/RatingBadge.tsx` + `.module.css`
- `avocado.kiss/components/RatingSection.tsx` + `.module.css`
- `avocado.kiss/components/icons/StarIcon.tsx`
- `avocado.kiss/lib/rating-client.ts`
- Turnstile loader (small `'use client'` helper or inline in `RatingSection`).

Changed:
- `avocado.kiss/lib/types.ts` — extend `Recipe`.
- `avocado.kiss/app/recipes/[slug]/page.tsx` — render `RatingBadge` in header,
  `RatingSection` at the end, add JSON-LD.
- `avocado.kiss/lib/supabase/` — add a server service-role client factory (new
  file, e.g. `lib/supabase/service.ts`) used only by the Route Handler.
- `avocado.kiss/.env.example`, `avocado.kiss/AGENTS.md` (env policy note).
- `platform-docs/database/schema.md` — document new columns/table/RPC.
- `platform-docs/sites/avocado-kiss.md` — document the ratings feature.
- `avocado.kiss/app/globals.css` — any new design tokens used by the components.

(`app/sitemap.ts` needs no change — no new public page route; the rate endpoint is
an API route, not a page.)

---

## 13. Update 2026-08-07 — as-shipped changes

These decisions were made during implementation and supersede the earlier text
where noted. The shipped code and the `platform-docs` site/schema docs reflect
this section.

### 13.1 Displayed rating COUNT — social-proof baseline (supersedes §3 for count)

§3 said the displayed count is `display_count` (`seed_count + ratings_count`). As
shipped, the **on-page count** is instead computed by `socialProofCount(recipeId,
realCount)` in `avocado.kiss/lib/rating.ts`:

- While real ratings are `<= REAL_COUNT_THRESHOLD` (100): show a **stable
  pseudo-random integer in 1..500**, derived deterministically from `recipe.id`
  (FNV-style hash). Same id → same number across renders/reloads, so it never
  flickers and never reads "0 Ratings".
- Once real ratings are `> 100`: show the real `display_count`.
- Applies to both `RatingBadge` (page computes it) and `RatingSection` (recomputes
  from the POST response). It is a display-only number; nothing is stored in the DB.

The DB `seed_count`/`seed_sum` columns remain (admin baseline for the **average**
and available for Phase B), but the count shown to visitors now comes from
`socialProofCount`, not from `seed_count`.

The displayed **average** is unchanged: `display_avg` with the 4.1 floor.

### 13.2 JSON-LD stays truthful

`aggregateRating` JSON-LD uses the **real** `display_count`/`display_avg` (only
when `display_count > 0`), never the social-proof number — publishing fabricated
review counts in structured data violates Google's guidelines. The on-page
social-proof count and the JSON-LD count therefore intentionally differ before a
recipe has real votes (JSON-LD is simply absent then).

### 13.3 Turnstile hardening

`components/Turnstile.tsx` (dedicated `'use client'` wrapper): the site key is
`.trim()`-ed (a stray space/newline triggers Cloudflare's "Invalid input for
parameter sitekey"); adds `error-callback`/`expired-callback` that clear the token
and log the CF error code; guards against double-render and removes the widget on
unmount. `RatingSection` blocks submit until a token exists, showing an
"unverified" message instead of firing a doomed request.

### 13.4 Star picker interaction fix

Hover flicker fixed by moving `onMouseLeave` from each star to the picker
container (so moving between stars no longer flashes the fill to zero); short
`color`/`transform` transitions and a `focus-visible` ring added. Arrow-key
roving-focus within the radiogroup remains a noted follow-up (Tab + Enter/Space
work today).

### 13.5 Spacing

Dropped the duplicate `.header { margin-top }` on the recipe page to tighten the
gap between the site header/nav and the recipe content.

### 13.6 Badge placement, submit loader, hero, layout ownership

Supersedes §1 and §7.1 where they say the badge sits *under* the title:

- **Badge above the title.** `RatingBadge` now renders **above** the `<h1>`, just
  under the category eyebrow (`.ratingTop` wrapper), not below the title.
- **Submit loader + floating status.** While a vote is in flight the picked stars
  stay lit and a spinner + "Sending your rating…" shows. The
  submitting/error/unverified message is an **absolutely-positioned tooltip pill
  below the stars** (inside a `position: relative` `.pickerWrap`) so it never
  reflows the star row. The caret is a bordered rotated square so the pill's top
  border stays continuous.
- **Hero image shorter.** `.image` aspect-ratio `16/9` (mobile) / `2/1` (≥768px)
  with `max-height: 60vh` so it never dominates on any viewport.
- **Bottom-gap ownership.** `.article` has **no** `padding-bottom`; the trailing
  `RatingSection` owns the page's bottom gap via a symmetric `padding-block`
  (`1.25rem`, no breakpoint override), so its top and bottom spacing are equal and
  identical on mobile/desktop. Do not re-add `padding-bottom` to `.article` — it
  would double the bottom gap.
