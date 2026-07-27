# Spec — Category as a products page (web.admin)

**Date:** 2026-07-27
**Project:** web.admin (only)
**Status:** design, awaiting implementation plan

## Goal

Change what clicking a category does. Today a click opens the **edit-category** modal.
Instead:

1. Clicking a category **navigates to a new route** — a page listing the products that
   belong to that category.
2. On that page you can **add a product to the category** and **edit the category
   itself** (via a button that opens the existing edit dialog).
3. Clicking a product on that page opens the product edit page; its back link reads
   **"Back to category"** and returns to the category page (instead of "Back to
   products").

We use a real route (not a modal) specifically so returning from the product editor to
the exact category is trivial.

## Current state

- Category ↔ product relationship is **denormalized by name**: `products.category`
  (text) equals `categories.name`. No join table, no `category_id`.
  - `src/lib/products.ts:13` — `category: string | null; // = categories.name`.
  - "Products in category X" = `products.filter(p => p.category === X.name)`
    (already done client-side in `ProductsPage.tsx:94–99`).
  - Rename/delete of a category cascades the name into `products` in app code
    (`src/lib/categories.ts` `renameCategory`/`updateCategory`/`deleteCategory`).
- Category list + click: `src/features/taxonomy/TaxonomyManager.tsx` — rows are a
  `<ul>`; when a config `editor` exists (categories do), the row label is a `<button>`
  with `onClick={() => openEditor(item.id)}` (line ~311). `openEditor` sets
  `editorId`/`editorOpen` and renders `CategoryEditDialog` (lines ~398–400). Row label
  shows a thumbnail + name + live count `usageCount(name)` (lines ~55–56).
- Config: `src/features/taxonomy/config.ts` — `categoryConfig` supplies
  `editor: CategoryEditDialog` (line ~73), `productField: "category"`.
- Edit dialog: `src/features/taxonomy/CategoryEditDialog.tsx` — Radix `Dialog`, props
  `{ site, id, open, onOpenChange }`, `id === null` = create.
- Routing: `src/App.tsx` — under `/:siteSlug` (via `SiteLayout` outlet):
  `categories` → `CategoriesPage`; `products` / `products/new` / `products/:productId`
  → products screens.
- Product editor: `src/features/products/ProductEditPage.tsx` — reads `productId` from
  params; two hardcoded **"Back to products"** links → `/${site.slug}/products`
  (lines ~129–135 for the loading/error state, ~270–274 within the form).
- `updateProduct(site, id, input)` requires a **full `ProductInput`**
  (`Omit<Product,"id"|"created_at"|"slug">`), so assigning only a category needs a
  narrow helper.

## Design

### New route: category detail page

Add under `/:siteSlug` (App.tsx, alongside `categories`):

```
categories/:categorySlug   ->  <CategoryProductsPage />
```

Use the category **slug** (categories already have a `slug` column) for a stable,
readable URL. `CategoriesPage` rows link here instead of opening the modal.

`CategoryProductsPage` responsibilities:

- Resolve the category by slug (`listCategories` / a `getCategoryBySlug`); 404-style
  empty state if not found.
- Header: category name + thumbnail, an **"Edit category"** button (opens the existing
  `CategoryEditDialog` with this category's id), and a back link to
  `/${site.slug}/categories`.
- **Products list**: `listProducts(site)` filtered by `p.category === category.name`
  (reuse the existing query key `['products', site.slug]`). Each product row/card links
  to `/${site.slug}/products/:productId?from=category:<categorySlug>` (see back-link
  design below).
- **Add product to category**: a picker (reuse the product-picker pattern; blog already
  has `ProductPickerDialog` in `src/features/posts/`) listing **only products with no
  category** (`category === null`); selecting one (or several) sets their `category` to
  `category.name`. Products already in another category are **not** shown here — moving a
  product between categories is done from the product editor. This keeps the screen
  non-destructive (you can never silently pull a product out of another category).

### Category list behavior change

In `TaxonomyManager.tsx` (categories only — brands keep current behavior since they have
no `editor`): the row becomes a navigation to the category page rather than
`openEditor`. Cleanest without over-generalizing the shared manager: add an optional
`config.onItemClick`/`rowHref(item)` to the taxonomy config so `categoryConfig` provides
a link builder (`/:siteSlug/categories/:slug`), while brands stay as-is. The
`CategoryEditDialog` is no longer opened from the row — it is opened from the category
page's "Edit category" button. (Keep `CategoryEditDialog` and its create-mode entry
point for the "New category" action, which stays where it is.)

### "Add product to category" data path

`updateProduct` needs a full payload, so add a narrow helper in `src/lib/products.ts`:

```ts
export async function setProductCategory(site, id, category: string | null): Promise<void>
// getDb(site).from("products").update({ category }).eq("id", id)
```

Invalidate `['products', site.slug]` (and the taxonomy count) after assignment. This
matches the name-based model and the cascade helpers in `categories.ts`.

### "Back to category" on the product editor

`ProductEditPage` must know it was reached from a category page and where to return.
Carry the origin in a **query param** (survives refresh, unlike router state):

- Link into the editor: `/${site.slug}/products/:productId?from=category:<categorySlug>`.
- In `ProductEditPage`, read `from`. If it is `category:<slug>`, both back links render
  **"Back to category"** → `/${site.slug}/categories/<slug>`; otherwise the current
  **"Back to products"** → `/${site.slug}/products` (default, unchanged).
- On save/cancel navigation, preserve the same return target when `from` is present.

## Files affected (web.admin)

- `src/App.tsx` — add route `categories/:categorySlug`.
- **New** `src/features/taxonomy/CategoryProductsPage.tsx` — the category detail page.
- `src/features/taxonomy/TaxonomyManager.tsx` — category rows navigate instead of
  `openEditor`; keep brands unchanged.
- `src/features/taxonomy/config.ts` — add a per-config row-navigation hook; keep
  `CategoryEditDialog` wired for create + the page's Edit button.
- `src/lib/products.ts` — add `setProductCategory` (and a `getCategoryBySlug` in
  `src/lib/categories.ts` if not present).
- `src/features/products/ProductEditPage.tsx` — context-aware back link from `from`
  query param (2 link sites + save/cancel nav).
- Reuse: `CategoryEditDialog.tsx`, a product picker (pattern from
  `src/features/posts/ProductPickerDialog.tsx`).

## Edge cases

- Category renamed while on the page → products' `category` cascades (existing behavior);
  slug may change → page should resolve by the current slug (navigate/refresh handles it).
- Product removed from category = set `category = null` (optional "remove from category"
  action on a row; include if cheap, else defer).
- A product belongs to only one category (single text field). The add-picker only offers
  uncategorized products, so this screen never overwrites an existing category; moving a
  product between categories happens in the product editor.
- Direct navigation to an unknown `categorySlug` → empty/not-found state.
- `from` param absent or unrecognized → default "Back to products" (no regression for
  the normal products flow).

## Testing (per platform-docs/methodology/testing.md)

- Component: `CategoryProductsPage` lists only products whose `category` matches; "Edit
  category" opens the dialog; "add product" calls `setProductCategory` and invalidates
  the products query.
- Component: `ProductEditPage` renders "Back to category" → correct href when
  `?from=category:<slug>`; renders "Back to products" otherwise.
- Component: category row navigates to the route (does not open the edit modal); brand
  row behavior unchanged.
- e2e optional / propose: categories → open a category → add a product → open the
  product → "Back to category" returns to the same page.

## Out of scope

- Introducing a real `category_id` FK / many-to-many (stays name-based).
- Changing brand behavior.
- Bulk category reassignment beyond the add-to-category picker.
