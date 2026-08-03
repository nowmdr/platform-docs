---
name: avocado-content-ops
description: Operate the Avocado Kiss recipe magazine's content through the Supabase connector — add or edit recipes, categories, tags, blog posts, Curated Shop products / shop categories / shop curation (Editors' picks, Pairs well with, Related reading), curate the home page (home_slots / Editor's Picks), edit footer & page SEO, and read the current state. Use for any "add a recipe", "add a shop product", "put this in Editors' picks", "put this on the home page", "edit this category", "tag this recipe", "what recipes/products do we have" request against the Avocado Kiss database (schema avocado_kiss). Teaches the project's philosophy and data conventions and enforces hard guardrails against structural / destructive changes. The live database schema (checked via the connector) is the source of truth; this skill is the operating manual. This is the AVOCADO KISS site, not CozyCorner — if a request is about the cozy-home-goods catalog, use the cozycorner-content-ops skill instead.
---

# Avocado Kiss — Content Operations (Supabase connector)

You are a claude.ai agent with the **Supabase connector** attached to the
project **base-one**. This skill is your operating manual for running the
**Avocado Kiss** recipe magazine's content from natural-language requests:
"add this recipe", "feature it on the home page", "rename this category", "tag
this Quick + Vegetarian", "write the footer newsletter blurb".

Avocado Kiss and CozyCorner share one Supabase project but live in **separate
schemas**. This skill covers **`avocado_kiss` only**. For the cozy-home-goods
affiliate catalog, use the **`cozycorner-content-ops`** skill instead. If a
request is ambiguous about which site it targets, **ask first**.

## 0. The one golden rule

**This skill teaches philosophy and conventions. The live database is the
source of truth for exact structure.**

The schema evolves (new columns, new content types — `posts` is still a
skeleton; the **Curated Shop** — `products` + `shop_categories` + three shop
curation tables — shipped in migration `0007`). So:

- **Before writing anything, inspect the real table** through the connector
  (`list_tables` with schema `avocado_kiss`, or `select * from
  information_schema.columns where table_schema='avocado_kiss' and
  table_name='<t>'`, or a `select … limit 1`). Compose the write from what the
  table actually looks like *now*.
- Use this document for **intent, relationships, gotchas, and guardrails** —
  not as a frozen column list.
- **If the live schema contradicts this document, trust the database**, do the
  right thing, and then tell the user the skill is out of date (see §10). (The
  repo's `lib/types.ts` also lags the DB in places — e.g. it types
  `home_slots.recipe_id` as non-null and omits `post_id`, but the live table is
  polymorphic. Trust the DB.)

## 1. Project philosophy (mental model)

Avocado Kiss is a **culinary recipe magazine** (English UI): a curated home
page, recipe pages, and category archives. Stack: **Next.js 16 (App Router,
RSC) + Supabase + Vercel**. Almost everything a visitor sees is **content
stored in the database** and rendered on the fly.

Things that matter for how you operate:

- **Edits are near-live.** Content pages are statically rendered with ISR
  (`revalidate = 60s`), so a change you make in the DB appears on the site
  within about a minute **without any redeploy**. What you write is real and
  public quickly — that is exactly why the guardrails below exist.
- **The admin panel does not exist yet.** Editing Avocado Kiss content is
  **Phase B** of `web.admin` and is **not built**. Right now this connector is
  the *only* way to edit the site's content — which means there are **no app
  safety rails**; you must supply them yourself (§5, §6).
- **Draft → review → publish.** New content should be created as a **draft**
  (`is_published = false`) so a human can review it before it goes public.
  ⚠️ **Publish-default gotcha:** `recipes.is_published` defaults to `false`
  (safe), **but `posts.is_published` and `home_slots.is_published` default to
  `true`** — a new post or home-slot goes **live immediately** unless you
  explicitly set `is_published = false`. Always set it explicitly.
  ⚠️ **The Curated Shop has NO draft state at all.** None of the five shop tables
  (`products`, `shop_categories`, `shop_editors_picks`, `product_pairings`,
  `product_reading`) has an `is_published` column — they are public-read. **A
  product or curation row is live the instant you insert it.** Tell the user that
  before you create shop content, and confirm per §6.
- **The home page is curated, not queried.** It is assembled from `home_slots`
  rows, not from "latest N recipes". Putting a recipe on the home page = adding
  a `home_slots` row (§3, §7).
- **Multisite.** One Supabase project (**base-one**) hosts several sites, one
  Postgres **schema per site**. Avocado Kiss lives in schema `avocado_kiss`.
  CozyCorner (`cozycorner`) shares the project — never operate across the
  schema boundary in one request.

## 2. Connection facts

| Thing | Value |
|---|---|
| Supabase project | **base-one** |
| Project ref | `zwrkphynupdubevzwdzy` |
| Project URL | `https://zwrkphynupdubevzwdzy.supabase.co` |
| Avocado Kiss schema | `avocado_kiss` (all content tables live here) |
| Avocado Kiss storage bucket | `avocado-kiss-photos` (flat keys, public read) |
| Shared SQL helpers | `public.slugify`, `public.set_updated_at`, `public.is_admin` (already exist project-wide — don't recreate) |

> ⚠️ **The connector runs with elevated privileges and bypasses RLS.** The
> site's Row-Level Security protects the public site, but it does **not**
> restrain you. That means *you can delete or corrupt anything*. The guardrails
> in §5 are the only safety net — treat them as hard rules, not suggestions.

## 3. Data relationship map (how the content connects)

These conventions are stable even when individual columns change. Internalize
them — they are how the data stays consistent.

- **All Avocado Kiss content is in schema `avocado_kiss`.** Always qualify
  tables with the schema.
- **Two different link styles coexist — know which is which:**
  - **Categories link by NAME** (like CozyCorner). `recipes.category` (a single
    text field) equals a `categories.name`. There is **no FK** — the join is a
    string match. Renaming a category must also update every matching
    `recipes.category` value (you own that cascade; there is no admin app to do
    it for you).
  - **Tags link by FK.** `recipe_tags`/`post_tags` are join tables referencing
    real `tags.id` (foreign keys, `on delete cascade`). To tag a recipe, insert
    `(recipe_id, tag_id, position)` rows — never a text field. Query real
    `tags.id` first; never invent ids.
  - **Shop products link to their category by SLUG** (a third style — don't
    confuse it with recipes). `products.category` (single text field) equals a
    `shop_categories.slug` — **by slug, not by name**. No FK — a string match. Set
    it to an existing `shop_categories.slug` or the product shows under no
    category. (`shop_categories.name` is only the display label — never the key.)
- **Categories ≠ tags** (they look similar but are different sensibilities):
  - **Categories** are the recipe's *section* (Breakfast, Seasonal, Seafood…),
    shown on cards and driving `/category/[slug]` archives and header nav.
  - **Tags** are *cross-content labels* shared by all content types — magazine
    labels (LIFE, COMMUNITY) and dish descriptors (Vegetarian, Quick, Comfort
    Food…). They render as `TAG | TAG` in Editor's Picks and on category-page
    cards.
- **A recipe is self-contained — no body-section tables.** Unlike CozyCorner
  posts, a recipe's `ingredients` and `steps` are **inline `text[]` arrays** on
  the `recipes` row. Array order = display order (steps render as 01, 02, …).
  There is no separate ingredients/steps table.
- **Images are relative flat keys or external URLs.** `hero_image_path` (the
  only image field in v1 — used both as card cover and recipe hero) and
  `pages.og_image_path` store either a bare key inside the `avocado-kiss-photos`
  bucket (e.g. `recipe-zucchini.jpg`, no folders) **or** a full external URL.
  Never store an absolute URL of our own Storage — it breaks on project moves.
  An empty/broken path falls back to a branded placeholder on the site, so a
  recipe without a photo is fine. `media.path` links to image fields **by
  string match** (no FK).
- **The database owns `slug`, `created_at`, `updated_at`.** Triggers generate
  `slug` from `title`/`name` (via `public.slugify`) and maintain timestamps.
  **Never send these on insert.** **Exception — the Curated Shop tables have no
  such triggers.** `products` and `shop_categories` have **no slug trigger and no
  `updated_at` column**; `slug` is `NOT NULL UNIQUE` with no default, so you
  **must supply it yourself** — compute it explicitly (`select
  public.slugify('Speckled Dinner Plate')`) and check it isn't already taken
  before inserting. (The shop curation tables have no slug at all.)
- **Empty optional fields are `null`, never empty strings.** Empty SEO/eyebrow/
  description fields intentionally fall back to code defaults.
- **Publication gating.** `is_published` controls visibility; the public site
  reads only published rows (see the default gotcha in §1).
- **Singletons and fixed tables.** `footer_settings` is a single-row
  "column-per-field" table — **update only**, never insert/delete. `pages` is a
  fixed set (currently just `home`) — **update only**, no create/delete.

## 4. Verify the live schema before you write

For any table you're about to modify:

1. Look at the actual columns, defaults, and CHECK constraints via the
   connector (e.g. `home_slots.slot` has a CHECK enumerating valid slot names;
   `home_slots` has `check (num_nonnulls(recipe_id, post_id) = 1)`).
2. Build your insert/update from that reality, applying the conventions in §3.
3. If something in this doc no longer matches, follow the DB and flag it (§10).

This is not optional — it's how the skill stays correct as the schema grows.

## 5. Guardrails — protection against damage (HARD rules)

You must **refuse** these, explain why, and offer the safe alternative:

- ❌ **No structural / DDL changes.** No `CREATE`/`ALTER`/`DROP TABLE`, no
  `CREATE`/`DROP POLICY`, no `GRANT`/`REVOKE`, no changing RLS or Auth, no
  creating/altering functions, triggers, or extensions, no migrations.
  → *"Schema changes go through migrations in the `avocado.kiss` repository
  (`supabase/migrations/`), not through the connector. I can't do that here."*
  In particular, adding a new `home_slots.slot` value or a new content type is a
  **migration**, not a content edit.
- ❌ **Don't touch admin/infra tables.** `admin_folders` is admin-panel
  plumbing (Phase B) — leave it alone. Leave Auth users, storage policies, and
  any table outside `avocado_kiss` alone.
- ❌ **`media` is not a free-write table.** It's the media manager's record of
  files that were actually uploaded to the bucket. Read it to find image keys;
  **don't fabricate `media` rows** for files that don't exist in Storage.
- ❌ **No mass or unscoped mutations.** Refuse any `DELETE`/`UPDATE` that lacks
  a `WHERE` targeting specific, named rows. Refuse "delete everything",
  `TRUNCATE`, or "unpublish all recipes". A destructive op must name exactly
  what it hits, and you confirm it first (§6).
- ✅ **Every write requires explicit confirmation** — no insert, update, or
  delete without the preview-and-confirm cycle in §6.
- ✅ **One schema at a time.** This skill is `avocado_kiss`. If a request could
  mean CozyCorner, ask before proceeding.
- ✅ **New content is a draft** (`is_published = false`) unless told to publish
  — and remember the default gotcha (§1): you must set it explicitly for
  `posts` and `home_slots`.

When in doubt, **prefer reading over writing**, and ask.

## 6. Preview-before-write (mandatory cycle for every mutation)

1. **Restate the intent** in plain language ("You want a new recipe 'X' in
   category 'Seasonal', as a draft — right?").
2. **Resolve dependencies with SELECTs first**: real `recipes.id` /
   `posts.id` / `tags.id` / `products.id` values, existing `categories.name` /
   `shop_categories.slug` (so you match, not duplicate), existing slugs/titles
   (avoid duplicates — and for shop, compute the new `slug` and check it's free),
   `media.path` for images, and — for home-page or shop-curation work — the
   current rows in the target `slot` or curation table (and the category's
   `item_count` if you're adding/removing a product).
3. **Show the preview**: target `schema.table`, the operation, and the exact
   field values (or the SQL / connector call you'll run). For a home-page
   placement, show the `home_slots` row (slot, position, recipe_id/post_id,
   eyebrow/description overrides, is_published). For tagging, show each
   `recipe_tags` row.
4. **Wait for an explicit "yes."** Do not execute on assumption.
5. **Execute** — insert the parent, capture its `id`, then insert children
   (tag joins, home_slots rows) referencing it.
6. **Report back**: what was created/changed, the slug (generated for recipes,
   supplied by you for shop), draft vs published (and for shop: **live now — no
   draft**), and where to see it (public URL pattern, e.g. `/recipes/<slug>`,
   `/category/<slug>`, `/product/<slug>`, `/shop/<category-slug>`, `/shop`, home
   page `/`).

## 7. Entity operations reference

Field lists here are the *intent*; confirm exact columns live (§4). All tables
are in schema `avocado_kiss`.

- **recipes** — the core content. Key fields: `title`, `excerpt` (always fill
  it — it feeds card descriptions and `seo_description` fallback), `category`
  (= a real `categories.name`, single value), `author_name` ("by Mark P."),
  `time_label` ("45 minutes" — a ready-made string, not a number),
  `servings_label` ("4 servings"), `hero_image_path` (§3 contract),
  `ingredients` (`text[]`, order = list order), `steps` (`text[]`, order = step
  numbers), `is_published` (defaults false), `published_at`, optional
  `seo_title`/`seo_description`. Don't send `slug`. New recipe with an unknown
  category → confirm whether to also create that `categories` row. Tagging is a
  separate step via `recipe_tags` (below).
- **categories** — header nav + `/category/[slug]` archives. `name` (the join
  key, unique), `position` (nav order), optional `seo_title`/`seo_description`
  (empty → auto-formula from `name`). There is **no category image field** in
  v1. Renaming a category must cascade to every `recipes.category` string.
- **tags** — the cross-content taxonomy (`name` unique, `position`). Magazine
  labels (LIFE, COMMUNITY) and dish descriptors (Vegetarian, Quick, …). Don't
  send `slug`. To attach tags to a recipe, insert **recipe_tags** rows.
- **recipe_tags / post_tags** — join tables: `(recipe_id|post_id, tag_id)` PK +
  `position` (tag display order). FK to `tags.id` with `on delete cascade`.
  This is how tagging works — never a text field on the recipe.
- **home_slots** — the curated home page. One row = "show this item in this
  slot, in this order." Fields: `slot` (CHECK-enumerated: `hero`, `grid_large`,
  `grid_medium`, `grid_list`, `wide`, `mosaic_banner`, `pick`, `pick_feature`),
  `position` (order within the slot), **`recipe_id` XOR `post_id`** (exactly one
  — CHECK `num_nonnulls=1`), `eyebrow` (label override; empty → recipe's
  category), `description` (override; empty → item's excerpt), `is_published`
  (defaults **true** — set false for staged placements). `eyebrow_secondary` is
  **deprecated** (v2 moved Editor's Picks labels to the item's tags) — don't use
  it. Slot roles & capacities (validated by the future admin; the site just
  renders what's there):

  | Slot | Capacity | Role |
  |---|---|---|
  | `hero` | 3 | Hero carousel slides |
  | `grid_large` | 1 | Mosaic feature card |
  | `grid_medium` | 2 | Mosaic medium cards |
  | `grid_list` | 5 | Mosaic small photo tiles |
  | `wide` | 2 | Mosaic wide cards |
  | `mosaic_banner` | 1 | Mosaic full-width banner |
  | `pick` | 3 | Editor's Picks numbered list (recipe or post) |
  | `pick_feature` | 1 | Editor's Picks sticky feature card (recipe or post) |

  The mosaic uses **recipe-only** slots (`grid_*`, `wide`, `mosaic_banner`);
  `pick`/`pick_feature` feed Editor's Picks and may point to a recipe **or** a
  post. To feature something: pick the slot, `select` current rows to choose a
  `position`, insert one row with exactly one of `recipe_id`/`post_id`.
- **posts** — blog articles: the **second content type, still a skeleton**
  (0 rows; final fields/design TBD, route `/archive/[slug]` in progress).
  Fields today: `title`, `slug` (don't send), `excerpt`, `hero_image_path`,
  `is_published` (defaults **true**), `published_at`. Usable for Editor's Picks
  placements now; treat as provisional and flag if asked to do more.
- **Curated Shop** (5 tables, migration `0007`) — the affiliate storefront:
  routes `/shop`, `/shop/[category]`, `/product/[slug]`; products link out to
  where you buy them. It does **not** follow every recipe convention — the two
  shop-wide rules from §1/§3 apply to all five tables: **(a) no draft state** (no
  `is_published` anywhere — a row is live on insert), and **(b) you supply
  `slug`** on `products`/`shop_categories` (no trigger). Details per entity:
  - **products** — the storefront items. Columns: `slug` (`NOT NULL UNIQUE` — you
    supply it via `slugify`), `title` (NOT NULL), `price` (`numeric`, NOT NULL — a
    bare number like `32.00`, no currency symbol; the site formats it),
    `referral_url` (NOT NULL — the outbound "Buy from …" link), `brand` (nullable
    — card eyebrow, e.g. "Clay & Co."), `description` (nullable — product-page
    body), `image_path` (nullable — same flat-key/external-URL contract as recipe
    `hero_image_path`, bucket `avocado-kiss-photos`; empty → placeholder),
    `category` (nullable text = a `shop_categories.slug` — **by slug, not name**,
    §3). No `is_published`, no `updated_at`. Adding/removing a product → **update
    the category's `item_count`** (see below).
  - **shop_categories** — the storefront sections (`/shop/[slug]`). Columns:
    `slug` (`NOT NULL UNIQUE` — you supply it), `name` (NOT NULL — display, e.g.
    "Ceramics & Table"), `position` (grid/order), `item_count` (NOT NULL —
    ⚠️ see cascade), optional `hero_eyebrow`/`hero_title`/`hero_description`/
    `hero_image_path` (category-page hero; empty → code fallbacks), optional
    `seo_title`/`seo_description`.
  - **⚠️ `shop_categories.item_count` is a STORED counter you keep in sync.** It
    is a plain column (no trigger); the hub `/shop` prints it as "N items". After
    any add/remove of a product in a category, set that category's `item_count` to
    the real number: `select count(*) from avocado_kiss.products where category =
    '<slug>'`. This cascade is yours to own — like renaming a category cascades to
    `recipes.category`. (Moving it to a trigger is a future migration, not a
    connector edit.)
  - **shop_editors_picks** — the "Editors' picks this month" strip on `/shop`.
    Row = `(product_id, position)`. **`UNIQUE(product_id)`** — a product can be a
    pick at most once. To feature: `select` current picks for a free `position`,
    then insert one row referencing a real `products.id`.
  - **product_pairings** — "Pairs well with" on a product page. Row =
    `(product_id, paired_product_id, position)`. FK both → `products.id`;
    **`UNIQUE(product_id, paired_product_id)`** + **`CHECK(product_id <>
    paired_product_id)`** (no self-pair, no dupes). **Directional** — A→B does not
    imply B→A; insert both rows if you want it mutual.
  - **product_reading** — "Related reading" on a product page: links a product to
    **real recipes**. Row = `(product_id, recipe_id, position)`. FK →
    `products.id` and **`recipes.id`**; `UNIQUE(product_id, recipe_id)`. Point it
    only at **published** recipes — the site embeds `recipes!inner` filtered to
    `is_published = true`, so an unpublished recipe silently drops out. Resolve
    real `recipes.id` with a SELECT first; never invent ids.
- **pages** — SEO for fixed routes. Currently `home` and `shop` rows.
  **Update-only** (`seo_title`, `seo_description`, `og_image_path`); no
  create/delete. (Unlike CozyCorner there are no `hero_sections`/body/markdown
  fields here.)
- **footer_settings** — singleton (one row), **update-only**. Footer text
  (`tagline`, `copyright`, `made_with`), social URLs (`instagram_url`,
  `pinterest_url`, `telegram_url`, `rss_url`), and the decorative newsletter
  block text (`newsletter_eyebrow`, `newsletter_title`, `newsletter_text`).
  Empty field → that element isn't rendered. The newsletter form writes nowhere
  — there is **no `subscribers` table** in v1.
- **media** — upload metadata (§5). Read it to find image keys; don't fabricate
  rows.

### Recipe-writing quality notes

- **`excerpt` is load-bearing** — it's the card blurb *and* the SEO description
  fallback. Always write a real one.
- **`ingredients`/`steps` order matters** — the array position is the on-page
  order; steps become 01, 02, … Keep each array element one ingredient / one
  step.
- **`time_label`/`servings_label` are display strings**, not numbers — write
  them as they should read ("45 minutes", "Serves 4").
- **SEO**: set `seo_title` (≤ ~60 chars, front-loaded keyword) and
  `seo_description` (~120–155 chars) when the recipe matters for search; leave
  `null` to fall back to `title`/`excerpt`. Write for people — honest, concrete,
  no keyword stuffing.
- Create as a **draft** (`is_published = false`) unless told otherwise.

## 8. Orientation recipes (read-only, to understand current state)

Use these to answer "what do we have" and to ground writes:

- Catalog overview: `select title, category, is_published, slug from
  avocado_kiss.recipes order by published_at desc limit 50;`
- Taxonomy: `select name, position from avocado_kiss.categories order by
  position;` · `select name, position from avocado_kiss.tags order by position,
  name;`
- A recipe's tags: `select t.name from avocado_kiss.recipe_tags rt join
  avocado_kiss.tags t on t.id = rt.tag_id where rt.recipe_id = '<id>' order by
  rt.position;`
- Current home page: `select slot, position, recipe_id, post_id, eyebrow,
  is_published from avocado_kiss.home_slots order by slot, position;`
- Posts: `select title, is_published, published_at, slug from
  avocado_kiss.posts order by published_at desc limit 50;`
- Shop catalog: `select p.title, p.brand, p.price, p.category, p.slug from
  avocado_kiss.products p order by p.created_at desc limit 50;`
- Shop categories + live counts (spot `item_count` drift): `select c.slug,
  c.name, c.position, c.item_count, (select count(*) from avocado_kiss.products p
  where p.category = c.slug) as actual from avocado_kiss.shop_categories c order
  by c.position;`
- Current Editors' picks: `select ep.position, p.title from
  avocado_kiss.shop_editors_picks ep join avocado_kiss.products p on p.id =
  ep.product_id order by ep.position;`
- A product's pairings / related reading: `select pr.position, x.title from
  avocado_kiss.product_pairings pr join avocado_kiss.products x on x.id =
  pr.paired_product_id where pr.product_id = '<id>' order by pr.position;` ·
  `select r.title from avocado_kiss.product_reading rd join avocado_kiss.recipes
  r on r.id = rd.recipe_id where rd.product_id = '<id>' order by rd.position;`
- Find an image key: `select path, original_name from avocado_kiss.media where
  original_name ilike '%zucchini%';`

Start most sessions by reading the relevant slice so you match existing data
instead of duplicating it.

## 9. Sibling skill

For the CozyCorner cozy-home-goods catalog (products, brands, post sections),
use the **`cozycorner-content-ops`** skill — it covers a different schema with
different conventions (name-based brand/category links, post section tables).
Don't cross the streams: this skill is `avocado_kiss` only.

## 10. Keep this skill in sync

This document is philosophy-level, so it should not churn on every column. But
when the project changes, the skill must change with it:

- If you (or the user) change **conventions, guardrails, entity contracts, or
  workflows**, update this skill in the same piece of work.
- If you notice the **live schema contradicts this document** while working,
  finish the task using the database as truth, then **tell the user** and offer
  to update the relevant section here.
- New content type going live (`posts` fields finalized, `product` shipped),
  new slot, new protected table → reflect it in §3/§5/§7.

A skill that drifts from reality is worse than no skill — it teaches wrong
assumptions. Keeping it current is part of the job.
