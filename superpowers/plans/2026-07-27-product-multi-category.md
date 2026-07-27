# Product Multi-Category (M2M) Implementation Plan

> **For agentic workers:** Use superpowers:subagent-driven-development or executing-plans. Steps use `- [ ]`.

**Goal:** Move product↔category from a single `products.category` text to a `product_categories(product_id, category_id)` join; admin edits multiple categories per product; the public site shows a product under every category it belongs to.

**Status of DB:** Migration A (create join + backfill 50 rows) is **already applied** to the live DB (`zwrkphynupdubevzwdzy`, schema `cozycorner`) and mirrored as `cozycorner/supabase/migrations/0027_product_categories.sql`. Migration B (`0028_drop_products_category.sql`, `drop column category`) is written but **applied last**, after both repos' code is done.

**Spec:** `platform-docs/superpowers/specs/2026-07-27-product-multi-category-design.md`

**Branches:** web.admin → `feat/category-products-page` (continues Feature 2). cozycorner → `feat/product-multi-category`. Commits + push allowed (site is in test mode). Verify each project with `npm test` + `npm run build` + `npm run lint`.

**Ordering:** do web.admin Part 1 (lib) first, then the admin UI, then cozycorner, then apply Migration B. Both repos must stop referencing `products.category` before B.

---

## PART A — web.admin (branch feat/category-products-page)

### Task A1: `productCategories` data layer

**Files:** Create `src/lib/productCategories.ts`; Test `src/lib/productCategories.test.ts` (optional — these are thin DB passthroughs like products.ts; cover via component tests instead if a supabase mock is heavy).

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

// Membership map: productId -> categoryId[]. Одна лёгкая выборка всей join-таблицы
// (каталог небольшой), как fetchFilterOptions на фронте.
export async function listProductCategories(
  site: SiteConfig,
): Promise<Map<string, string[]>> {
  const { data, error } = await getDb(site)
    .from("product_categories")
    .select("product_id,category_id");
  if (error) throw error;
  const map = new Map<string, string[]>();
  for (const row of (data ?? []) as { product_id: string; category_id: string }[]) {
    const list = map.get(row.product_id) ?? [];
    list.push(row.category_id);
    map.set(row.product_id, list);
  }
  return map;
}

// Категории одного товара (для редактора).
export async function getProductCategoryIds(
  site: SiteConfig,
  productId: string,
): Promise<string[]> {
  const { data, error } = await getDb(site)
    .from("product_categories")
    .select("category_id")
    .eq("product_id", productId);
  if (error) throw error;
  return (data ?? []).map((r) => (r as { category_id: string }).category_id);
}

export async function addProductToCategory(
  site: SiteConfig,
  productId: string,
  categoryId: string,
): Promise<void> {
  const { error } = await getDb(site)
    .from("product_categories")
    .upsert({ product_id: productId, category_id: categoryId });
  if (error) throw error;
}

export async function removeProductFromCategory(
  site: SiteConfig,
  productId: string,
  categoryId: string,
): Promise<void> {
  const { error } = await getDb(site)
    .from("product_categories")
    .delete()
    .eq("product_id", productId)
    .eq("category_id", categoryId);
  if (error) throw error;
}

// Diff-sync набора категорий товара (редактор): вставить недостающие, удалить лишние.
export async function setProductCategories(
  site: SiteConfig,
  productId: string,
  categoryIds: string[],
): Promise<void> {
  const db = getDb(site);
  const current = await getProductCategoryIds(site, productId);
  const next = new Set(categoryIds);
  const cur = new Set(current);
  const toAdd = [...next].filter((id) => !cur.has(id));
  const toRemove = [...cur].filter((id) => !next.has(id));
  if (toAdd.length) {
    const { error } = await db
      .from("product_categories")
      .upsert(toAdd.map((category_id) => ({ product_id: productId, category_id })));
    if (error) throw error;
  }
  if (toRemove.length) {
    const { error } = await db
      .from("product_categories")
      .delete()
      .eq("product_id", productId)
      .in("category_id", toRemove);
    if (error) throw error;
  }
}
```

Query key for the membership map: `['product-categories', site.slug]`.

- [ ] Write the file, `npm run build` (clean), commit: `feat(products): product_categories data layer (M2M membership)`.

### Task A2: drop `category` from products lib

**Files:** `src/lib/products.ts`.
- Remove `category: string | null` from `Product` and from the `ProductListItem` Pick and the `.select(...)` string (both `listProducts` and any select). `ProductInput = Omit<Product,"id"|"created_at"|"slug">` then automatically loses `category`.
- Delete `setProductCategory` (added in Feature 2 — replaced by the join helpers).
- `npm run build` will now flag every consumer of `.category` (ProductsPage, ProductEditPage, TaxonomyManager, CategoryProductsPage, ProductPickerDialog). Those are fixed in later tasks — expect the build to be red until Task A6. To keep commits green, do Tasks A2–A6 as ONE commit (they are interdependent through the removed field).

### Task A3: product editor — categories chips (multi-select)

**Files:** `src/features/products/ProductEditPage.tsx`, and a new small `src/features/products/CategoryChipsField.tsx`.

- Remove `category` from `productSchema`, `toInput`, `toFormValues` (the single combobox).
- Add a chips field: a component that shows selected categories as removable chips plus an "Add category" combobox listing categories not yet selected. Reuse `listCategories` and the existing `TaxonomyCombobox`/`Command` primitives where possible; if simpler, use a shadcn `Popover` + `Command` multi-list. The field's value is `string[]` of category ids, held in local state in the form component (not RHF, since it's not a product column) — load initial via `getProductCategoryIds(site, product.id)` when editing (`enabled: !isNew`).
- On save: after the product create/update succeeds, call `setProductCategories(site, productId, selectedIds)` (for a newly created product use the returned `created.id`). Invalidate `['product-categories', site.slug]`. The unsaved-changes guard (`useBlocker`) should also treat a change in the chips set as dirty — track a `categoriesDirty` boolean and OR it into the blocker condition.

Because this component is nontrivial, the implementer may read `TaxonomyCombobox.tsx` and the existing category field markup first, then build the chips UI consistent with the repo. Keep UI text English.

### Task A4: products list filter by membership

**Files:** `src/features/products/ProductsPage.tsx`.
- Load the membership map: `useQuery(['product-categories', site.slug], () => listProductCategories(site))`.
- Category `<Select>` options: from `listCategories` use `c.id` as value + `c.name` as label (was `c.name` for both). Keep an `ALL` sentinel.
- Filter predicate: replace `category === ALL || p.category === category` with `category === ALL || (membership.get(p.id) ?? []).includes(category)`.
- Row subtitle currently `[product.brand, product.category].filter(Boolean).join(' · ')` → show brand only (category is gone from the row). Same in `src/features/posts/ProductPickerDialog.tsx` (its subtitle uses `p.category`) — change to brand only.

### Task A5: CategoryProductsPage — revise for M2M + remove-from-category + products-row style

**Files:** `src/features/taxonomy/CategoryProductsPage.tsx`, its test, and extract a shared row `src/features/products/ProductListRow.tsx`.
- Extract the product row markup used by `ProductsPage` (the `<Link to={id}>` with title + brand subtitle, **no thumbnail**) into `ProductListRow` and reuse it in both `ProductsPage` and `CategoryProductsPage` (DRY). It takes `{ to: string; title: string; subtitle?: string; trailing?: ReactNode }`.
- List: products whose membership includes this `categoryId` (from `listProductCategories`). Render each as `ProductListRow` with `to={`/${site.slug}/products/${p.id}?from=category:${categoryId}`}` and a trailing **remove** button.
- Remove-from-category: a destructive icon button → `AlertDialog`: title "Remove \"{title}\" from \"{category.name}\"?", body "The product is not deleted and stays in its other categories." Confirm → `removeProductFromCategory(site, p.id, categoryId)`; invalidate `['product-categories', site.slug]`.
- Add product: `ProductPickerDialog` `excludeIds` = product ids already in this category (from membership). `onAdd(ids)` → `addProductToCategory` for each; invalidate.
- Remove the old category-thumbnail row markup and the `setProductCategory` usage. Update the test: membership is mocked via `listProductCategories`; assert the list, the add flow (`addProductToCategory`), and the remove flow (`removeProductFromCategory`).

### Task A6: taxonomy usage counts + categories cascade cleanup

**Files:** `src/features/taxonomy/TaxonomyManager.tsx`, `src/features/taxonomy/config.ts`, `src/lib/categories.ts`.
- `TaxonomyManager` usage count for categories: instead of `products.filter(p => p[productField] === name)`, use a membership count. Load `listProductCategories(site)` when the config is a category; count products whose membership includes `item.id`. Simplest: add an optional `usageCountSource` to the config, or compute a `Map<categoryId, count>` in the manager when `config.kind === 'category'`. Brands keep the `products.brand` name count. Keep it clean and typed.
- `src/lib/categories.ts`: `renameCategory` — drop the `products` name-cascade (membership is id-based; renaming only updates `categories.name`). `deleteCategory` — drop the `products` null-cascade (FK `on delete cascade` on `product_categories` handles it). `updateCategory` — drop its name-cascade into products too.

- [ ] After A2–A6: `npm test`, `npm run build`, `npm run lint` all green. Commit A2–A6 together: `feat(products): product↔category many-to-many (chips editor, membership filter, category page, counts)`.

---

## PART B — cozycorner (branch feat/product-multi-category)

Read `node_modules/next/dist/docs/` before route/query code. RSC; ISR `revalidate=60`.

### Task B1: types

**Files:** `lib/types.ts`. Remove `category` from `Product`. (Add `categories?: string[]` only if the product page displays categories — see B3.)

### Task B2: join-based product queries

**Files:** `lib/products.ts`.
- `ProductFilters`: `category?: string` → `categoryId?: string`.
- `fetchProducts`: when `categoryId` set, use the inner-join embed:
  ```ts
  let query = filters?.categoryId
    ? client.from("products").select("*, product_categories!inner(category_id)")
        .eq("product_categories.category_id", filters.categoryId)
    : client.from("products").select("*");
  ```
  Keep brand/price filters + deterministic order + range. (When embedding, the returned rows include a `product_categories` array — either omit it from the mapped `Product` or strip it; do not add it to `Product` type.)
- `fetchFilterOptions(client, categoryId?)`: brands within a category. When `categoryId` set, query products via the same inner embed and collect distinct brands; categories for the top-level `/shop` filter come from the `categories` table (a separate `fetchCategories`/existing loader), not from product rows.
- `fetchRelatedProducts(client, product)`: get the product's category ids (`select category_id from product_categories where product_id = product.id`); if any, fetch products in those categories (`product_categories!inner`, `.in("product_categories.category_id", ids)`, `.neq("id", product.id)`, dedup) then top up with recent others to `limit`.

### Task B3: shop category page + any category display

**Files:** `app/shop/[category]/page.tsx` (+ grep for other readers of `product.category`).
- Resolve category by slug (unchanged) → pass `category.id` as `categoryId` to `fetchProducts`/`fetchFilterOptions`.
- `grep -rn "\.category\b" app lib components` — fix every remaining reader (product card, product page, search). If the product page shows its category, fetch names via the join and render all.

- [ ] `npm test`, `npm run build` (TS + prerender), `npm run lint` green. Commit: `feat(shop): products belong to many categories (join-based catalog queries)`. Push both branches.

---

## PART C — finalize DB

- [ ] After both repos are committed/pushed (and, ideally, deployed — site is in test mode so temporary breakage is acceptable), apply Migration B via MCP: `alter table cozycorner.products drop column if exists category;` (mirrors `0028_drop_products_category.sql`). Verify `products` no longer has `category` and both apps still build/run.
- [ ] Update `platform-docs/database/schema.md`: document `product_categories` and the removal of `products.category`.

---

## Self-Review
- Model change covered end to end: join table (done, A1), admin writes (A3/A5), admin reads/filters/counts (A4/A5/A6), simplified cascades (A6), frontend reads (B2/B3), column drop (C). ✓
- Requested admin refinements: add-picker shows all non-member products (A5), products-row style w/o thumbnail via shared `ProductListRow` (A4/A5), remove-from-category with warning modal (A5). ✓
- Migration safety: B applied last (C), both repos stop reading `products.category` before it. ✓
- No dangling `products.category` readers: enforced by A2 (build breaks until fixed) and the B3 grep. ✓
