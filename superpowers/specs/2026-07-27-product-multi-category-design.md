# Spec — Product ↔ many categories (cozycorner + web.admin + DB)

**Date:** 2026-07-27
**Projects:** cozycorner (DB migration + public frontend), web.admin (admin UI). Cross-repo.
**Status:** design, awaiting review
**Supersedes:** the single-category assumption in
`2026-07-27-category-products-page-design.md` (that page stays, but its data layer
changes: add/remove become join-table operations; the list reuses the products-row style;
a "remove from category" action is added). Feature-2 work already on branch
`feat/category-products-page` (rowTo navigation, CategoryProductsPage, back-to-category)
is largely reused with its data layer swapped.

## Goal

A product can belong to **one or more categories**. Model becomes a clean many-to-many
via a `product_categories` join table keyed by `category_id`; the legacy single
`products.category` (text = category name) is migrated into it and then dropped. Both the
public site (cozycorner) and the admin reflect membership; a product appears under every
category it belongs to.

Admin refinements requested alongside the model change:
- **Add product to a category**: the picker lists **all** products (with search), excluding
  those already in this category — since a product can now be in many categories, adding
  it here does not remove it from others.
- **Category page list**: reuse the **same row style as the normal products list**
  (title + brand·? subtitle, **no image thumbnail**), not the current thumbnail rows.
- **Remove from category**: each row gets a remove action that deletes only the
  membership (the product stays); a confirmation modal makes clear it removes the product
  *from this category*, not deletes the product.

## Current model (to be replaced)

- `products.category text` = a single category **name** (`= categories.name`); no FK.
- Rename/delete of a category cascades the name into `products.category` in app code
  (`web.admin/src/lib/categories.ts`).
- **cozycorner** filters by this text everywhere: `lib/products.ts` `fetchProducts`
  (`.eq("category", name)`), `fetchFilterOptions` (`.eq("category", name)` + distinct),
  `fetchRelatedProducts` (`.eq("category", product.category)`); `app/shop/[category]/page.tsx`
  resolves a category by slug then filters products by `category.name`; `lib/types.ts`
  `Product.category: string | null`.
- **web.admin**: `products.category` in `ProductListItem`/`Product`; `ProductsPage` category
  `<Select>` filter (`p.category === name`); `ProductEditPage` single `TaxonomyCombobox`
  category field; `TaxonomyManager` usage count (`p[productField] === name`);
  `CategoryProductsPage` (`p.category === name`, `setProductCategory`).

## Target model

New table (schema `cozycorner`, owned by cozycorner migrations):

```sql
create table cozycorner.product_categories (
  product_id  uuid not null references cozycorner.products(id)   on delete cascade,
  category_id uuid not null references cozycorner.categories(id) on delete cascade,
  primary key (product_id, category_id)
);
create index on cozycorner.product_categories (category_id);
```

- **RLS**: public/anon `select` allowed (catalog is public); insert/update/delete only for
  admin (`is_admin()`), mirroring the existing `products`/`categories` policies. Confirm
  exact policy shape against the live schema during implementation.
- Membership is by **`category_id`** → renaming a category no longer touches products, and
  deleting a category removes its memberships via `on delete cascade` (both simplifications
  vs today's name cascades).

## Migration sequencing (safety-critical)

The public site keeps reading `products.category` until the new frontend is deployed.
Therefore the column is dropped **only after** the new cozycorner frontend is live.

- **Migration A — additive, safe to apply now** (`cozycorner/supabase/migrations/<ts>_product_categories.sql`):
  create the table + index + RLS, then **backfill**:
  ```sql
  insert into cozycorner.product_categories (product_id, category_id)
  select p.id, c.id
  from cozycorner.products p
  join cozycorner.categories c on c.name = p.category
  where p.category is not null
  on conflict do nothing;
  ```
  Does not change `products.category`; the live (old) site is unaffected.
  **Apply via MCP now.** Mirror the file in the repo (workspace rule).
- **Migration B — destructive, apply only AFTER the new frontend is deployed**
  (`cozycorner/supabase/migrations/<ts+1>_drop_products_category.sql`):
  `alter table cozycorner.products drop column category;`
  Write the file now; **do not apply until the user confirms the new cozycorner + admin are
  deployed**. Until then both repos' new code must not depend on the column existing at
  runtime beyond what Migration A guarantees.

During development neither repo's new code reads/writes `products.category`; it uses the
join table exclusively. So once both are deployed together, dropping the column is clean.

## cozycorner frontend changes

`lib/types.ts`
- Remove `Product.category`. If any UI shows a product's categories, add
  `categories?: string[]` populated where needed (see product page below), but do **not**
  add it to the base `Product` select by default (keep list queries lean).

`lib/products.ts`
- `fetchProducts(client, page, filters)` — when `filters.categoryId` is set, filter via the
  join using an inner embed:
  ```ts
  let query = client
    .from("products")
    .select("*, product_categories!inner(category_id)")
    .eq("product_categories.category_id", filters.categoryId)
  ```
  (unfiltered catalog keeps `select("*")`). Keep the deterministic
  `order(created_at desc).order(id desc).range(...)`. Change `ProductFilters.category?: string`
  → `categoryId?: string`.
- `fetchFilterOptions(client, categoryId?)` — brands present among a category's products:
  filter the same way via the join; return the brand list. Categories list for the top-level
  `/shop` filter comes from `categories` (all categories), not from product rows.
- `fetchRelatedProducts(client, product)` — "same category" now means shares any category:
  fetch the product's `category_id`s from `product_categories`, then products in those
  categories (excluding itself), top up with recent others as today.

`app/shop/[category]/page.tsx`
- Resolve category by slug (unchanged), then pass `category.id` to `fetchProducts` /
  `fetchFilterOptions` as `categoryId`.

`lib/categories.ts`, sitemap, search, `CategoryGrid` — category listing/slug logic is
unchanged (categories table untouched). Verify no other reader of `products.category`
remains (`grep`). Product page (`app/product/[slug]` or similar) — if it displays a
category, fetch the product's category names via the join and render all of them.

Follow the cozycorner rules: check `node_modules/next/dist/docs/` before route/query code;
RSC; ISR `revalidate = 60`; update `lib/types.ts` and `platform-docs/database/schema.md`.

## web.admin changes

Data layer (`src/lib/`)
- New `src/lib/productCategories.ts`: `listProductCategoryIds(site)` →
  `Map<productId, categoryId[]>` (or per-product fetch), `setProductCategories(site, productId,
  categoryIds)` (diff-sync: insert missing, delete removed), `addProductToCategory(site,
  productId, categoryId)`, `removeProductFromCategory(site, productId, categoryId)`.
- `src/lib/products.ts`: drop `category` from `Product`/`ProductListItem` and the selects;
  remove `setProductCategory` (replaced by the join helpers). `ProductInput` loses
  `category`.
- `src/lib/categories.ts`: `renameCategory` no longer cascades into products (membership is
  id-based); `deleteCategory` no longer nulls `products.category` (FK cascade handles the
  join). Simplify both.

Product editor (`ProductEditPage.tsx`)
- Replace the single category `TaxonomyCombobox` with a **multi-select** of categories
  (chips + add/remove, or a multi-combobox). Load the product's current `category_id`s;
  on save, `setProductCategories(site, productId, selectedIds)` (in addition to the product
  update). Remove `category` from the product form schema/`toInput`/`toFormValues`.

Products list (`ProductsPage.tsx`)
- Category `<Select>` filter now filters by membership: build a `Map<productId,
  categoryId[]>` and keep a product when the selected category id is in its set. The row
  subtitle showing `category` → show brand only (or the product's category names if cheap).

Category page (`CategoryProductsPage.tsx`) — revise the already-built page:
- **List**: products whose membership includes this `categoryId`. Render with the **same row
  style as `ProductsPage`** (title + subtitle, **no thumbnail**). Extract that row into a
  small shared `ProductRow`/list component reused by both `ProductsPage` and this page (DRY),
  or mirror the markup if extraction is out of scope — prefer extraction.
- **Add product**: `ProductPickerDialog` with `excludeIds` = products already in this
  category (so it lists **all** other products, with search). `onAdd(ids)` →
  `addProductToCategory` for each; invalidate the products/membership queries.
- **Remove from category**: each row has a destructive icon button; clicking opens an
  `AlertDialog` — "Remove *{title}* from *{category}*? The product itself is not deleted and
  stays in its other categories." Confirm → `removeProductFromCategory(site, productId,
  categoryId)`; invalidate.
- Keep the header (Back to categories, name, count, Edit category) and the product-row link
  `…/products/:id?from=category:<id>` (back-to-category still works). Drop the category
  thumbnail-row markup.

Taxonomy (`TaxonomyManager.tsx`, `config.ts`)
- Category usage count now comes from the join, not `p.category === name`. Provide the count
  via a membership map (per-category product counts). `productField` for categories is no
  longer a `products` column — adjust the config/count source (brands still use
  `products.brand`). Keep brands entirely unchanged.

## Query keys / caching (web.admin)
- Reuse `['products', site.slug]` and `['categories', site.slug]`. Add
  `['product-categories', site.slug]` for the membership map. Invalidate the membership key
  (and products where relevant) after add/remove/editor-save.

## Data flow (admin add/remove)
```
Category page (categoryId)
  membership map ['product-categories', site.slug]  ← listProductCategoryIds
  list = products where map[product.id] includes categoryId
  Add:    picker(excludeIds = those already in category) → addProductToCategory(pid, categoryId) → invalidate
  Remove: confirm modal → removeProductFromCategory(pid, categoryId) → invalidate
Product editor
  load category_ids for product → multi-select
  save → setProductCategories(pid, selectedIds) (diff-sync) + product update
```

## Edge cases / risks
- **Migration ordering** (above) — dropping `products.category` before the frontend deploys
  breaks the live site. Migration B is gated on deploy.
- Backfill matches categories by **name**; any `products.category` value with no matching
  `categories.name` row is silently skipped (orphan text). Report the count of skipped rows
  when applying Migration A so the user can reconcile.
- A product with **zero** categories is now valid (previously `null`); the public catalog
  simply won't surface it under any category — acceptable.
- RLS on the join table must allow anon `select` (else the public shop category pages return
  nothing) and restrict writes to admin.
- `on delete cascade` on both FKs: deleting a product or a category cleans memberships
  automatically — verify no app code double-deletes.

## Testing (per platform-docs/methodology/testing.md)
- **web.admin** component: category page lists members via the membership map (no thumbnail,
  products-row style); Add picker excludes current members and lists the rest; Remove opens
  the warning modal and calls `removeProductFromCategory`; product editor multi-select loads
  and diff-syncs `category_id`s; ProductsPage category filter uses membership.
- **web.admin** lib: `setProductCategories` diff-sync (insert missing / delete removed);
  add/remove helpers hit the join table.
- **cozycorner** unit: `fetchProducts`/`fetchFilterOptions`/`fetchRelatedProducts` filter via
  the join by `categoryId`; `/shop/[category]` shows a product that belongs to multiple
  categories under each.
- **cozycorner** e2e (:3000, live Supabase, after Migration A): a multi-category product
  appears on each of its category pages.
- Migration A verified against live schema (backfill counts) before wiring code; Migration B
  applied only post-deploy.

## Out of scope
- Category ordering/nesting; per-product "primary" category; bulk membership editing beyond
  the add-picker and per-row remove.
- Blog preview (Feature 3) — separate spec, paused.

## Rollout order (summary)
1. Apply **Migration A** (create + backfill) via MCP; report skipped-orphan count.
2. Implement cozycorner frontend (join-based reads) + web.admin (join-based writes/UI) on
   their branches; tests green.
3. User merges + deploys both.
4. Apply **Migration B** (drop `products.category`) via MCP.
