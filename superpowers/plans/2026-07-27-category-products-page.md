# Category Products Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make clicking a category open a dedicated page listing that category's products (instead of the edit modal); on that page you can edit the category, add uncategorized products to it, and open a product whose "Back" link returns to the category.

**Architecture:** A new route `categories/:categoryId` renders `CategoryProductsPage`. The shared `TaxonomyManager` gets an optional `rowTo` config hook so category rows become navigation links (brands keep their current behavior). Category ↔ product relationship stays denormalized-by-name (`products.category === categories.name`); adding a product sets its `category` via a new narrow `setProductCategory` helper. The product editor reads a `?from=category:<id>` query param to swap its back link to "Back to category".

**Tech Stack:** Vite + React + TS, react-router-dom (data router), TanStack Query, shadcn/ui, Tailwind v4, Vitest + RTL. Project: `web.admin` only. No DB migration.

**Spec:** `platform-docs/superpowers/specs/2026-07-27-category-products-page-design.md`

**Deviation from spec (intentional):** the spec suggested routing by category **slug**; we route by **id** because the shared `TaxonomyItem` type carries no slug (only categories have one) and `getCategory(site, id)` already exists. Id keeps the shared type clean.

**Repo rules:** UI text English; empty optional fields → null; reuse shared query keys (`['products', site.slug]`, `['categories', site.slug]`) — never fork a key for the same data. Do not push; local commits on the working branch are authorized.

---

## File Structure

- **Modify** `src/features/taxonomy/config.ts` — add optional `rowTo?: (item: TaxonomyItem) => string` to `TaxonomyConfig`; set it on `categoryConfig`.
- **Modify** `src/features/taxonomy/TaxonomyManager.tsx` — when `config.rowTo` is set, render the row label as a react-router `<Link>` instead of the modal-opening `<button>`.
- **Create** `src/features/taxonomy/TaxonomyManager.test.tsx` — verify a `rowTo` config renders rows as links (and a no-`rowTo`/`editor` config renders a modal-opening button).
- **Modify** `src/lib/products.ts` — add `setProductCategory(site, id, category)`.
- **Create** `src/features/taxonomy/CategoryProductsPage.tsx` — the category detail page (products list + Edit category + Add product).
- **Create** `src/features/taxonomy/CategoryProductsPage.test.tsx` — page behavior.
- **Modify** `src/App.tsx` — add route `categories/:categoryId`.
- **Modify** `src/features/products/ProductEditPage.tsx` — context-aware back link/label + back-aware delete navigation, driven by `?from=category:<id>`.
- **Modify** `src/features/products/ProductEditPage.test.tsx` if it exists, else **Create** it — verify the back link switches to "Back to category".

---

## Task 1: TaxonomyManager row navigation via `config.rowTo`

**Files:**
- Modify: `src/features/taxonomy/config.ts`
- Modify: `src/features/taxonomy/TaxonomyManager.tsx`
- Test: `src/features/taxonomy/TaxonomyManager.test.tsx`

- [ ] **Step 1: Write the failing test**

Create `src/features/taxonomy/TaxonomyManager.test.tsx`:

```tsx
import { describe, expect, it, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import type { TaxonomyConfig, TaxonomyItem } from './config'
import { TaxonomyManager } from './TaxonomyManager'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
// TaxonomyManager fetches products for usage counts — stub to empty.
vi.mock('@/lib/products', () => ({ listProducts: vi.fn().mockResolvedValue([]) }))

const site: SiteConfig = {
  slug: 'cozycorner',
  label: 'CozyCorner',
  projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test',
  schema: 'cozycorner',
  bucket: 'cozycorner-photos',
}

const items: TaxonomyItem[] = [
  { id: 'cat-1', name: 'Vases', position: 0, image_path: null },
]

function makeConfig(overrides: Partial<TaxonomyConfig> = {}): TaxonomyConfig {
  return {
    kind: 'category',
    noun: 'category',
    plural: 'categories',
    title: 'Categories',
    hasImage: true,
    productField: 'category',
    queryKey: () => ['categories', site.slug],
    list: async () => items,
    create: async () => items[0],
    rename: async () => {},
    remove: async () => {},
    reorder: async () => {},
    ...overrides,
  }
}

function renderManager(config: TaxonomyConfig) {
  const qc = new QueryClient()
  render(
    <QueryClientProvider client={qc}>
      <MemoryRouter initialEntries={['/cozycorner/categories']}>
        <TaxonomyManager site={site} config={config} />
      </MemoryRouter>
    </QueryClientProvider>,
  )
}

describe('<TaxonomyManager /> row navigation', () => {
  it('renders the row as a link when config.rowTo is set', async () => {
    renderManager(makeConfig({ rowTo: (item) => item.id }))
    const link = await screen.findByRole('link', { name: /Vases/ })
    expect(link).toHaveAttribute('href', '/cozycorner/categories/cat-1')
  })
})
```

- [ ] **Step 2: Run test — expect FAIL**

Run: `npm test -- src/features/taxonomy/TaxonomyManager.test.tsx`
Expected: FAIL — `rowTo` is not a valid `TaxonomyConfig` property yet (type error) and/or the row renders as a `<button>`, so `getByRole('link')` finds nothing.

- [ ] **Step 3: Add `rowTo` to the config type + categoryConfig**

In `src/features/taxonomy/config.ts`, add this field to the `TaxonomyConfig` type (right after the `editor?: ...` block, before the closing `}`):

```ts
  // Относительный путь для клика по ряду (категория → страница её товаров).
  // Задан — ряд это ссылка (навигация); не задан — поведение по editor/inline.
  rowTo?: (item: TaxonomyItem) => string;
```

Then add to `categoryConfig` (e.g. right after `editor: CategoryEditDialog,`):

```ts
  rowTo: (item) => item.id,
```

Leave `brandConfig` unchanged (no `rowTo`).

- [ ] **Step 4: Render the row as a Link when `rowTo` is set**

In `src/features/taxonomy/TaxonomyManager.tsx`:

Add the router import near the top (with the other imports):

```tsx
import { Link } from 'react-router-dom'
```

Replace the row-label branch (currently the `{Editor ? (<button ...>{label}</button>) : (<div ...>{label}</div>)}` block inside the `<li>`) with a three-way choice:

```tsx
                {config.rowTo ? (
                  <Link
                    to={config.rowTo(item)}
                    className="flex min-w-0 flex-1 items-center gap-3 rounded-md px-3 py-2 text-left transition-colors hover:bg-accent/50"
                  >
                    {label}
                  </Link>
                ) : Editor ? (
                  <button
                    type="button"
                    onClick={() => openEditor(item.id)}
                    className="flex min-w-0 flex-1 items-center gap-3 rounded-md px-3 py-2 text-left transition-colors hover:bg-accent/50"
                  >
                    {label}
                  </button>
                ) : (
                  <div className="flex min-w-0 flex-1 items-center gap-3 px-3 py-2">{label}</div>
                )}
```

Do NOT remove the `openEditor`/`editorOpen`/`editorId` state or the bottom `{Editor && (<Editor .../>)}` render — the New-category button (`onNew` → `openEditor(null)`) still uses the modal in create mode. Category rows simply no longer open it; the per-category edit entry now lives on the category page (Task 2).

- [ ] **Step 5: Run test — expect PASS**

Run: `npm test -- src/features/taxonomy/TaxonomyManager.test.tsx`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/features/taxonomy/config.ts src/features/taxonomy/TaxonomyManager.tsx src/features/taxonomy/TaxonomyManager.test.tsx
git commit -m "feat(taxonomy): category rows navigate via config.rowTo"
```

---

## Task 2: `setProductCategory` helper + CategoryProductsPage + route

**Files:**
- Modify: `src/lib/products.ts`
- Create: `src/features/taxonomy/CategoryProductsPage.tsx`
- Create: `src/features/taxonomy/CategoryProductsPage.test.tsx`
- Modify: `src/App.tsx`

- [ ] **Step 1: Add the `setProductCategory` helper**

In `src/lib/products.ts`, add after `updateProduct`:

```ts
// Точечная смена категории товара (экран «товары категории»): updateProduct требует
// полный ProductInput, а здесь меняем только одно поле по имени категории.
export async function setProductCategory(
  site: SiteConfig,
  id: string,
  category: string | null,
): Promise<void> {
  const { error } = await getDb(site).from("products").update({ category }).eq("id", id);
  if (error) throw error;
}
```

- [ ] **Step 2: Write the failing page test**

Create `src/features/taxonomy/CategoryProductsPage.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { fireEvent, render, screen, waitFor, within } from '@testing-library/react'
import { MemoryRouter, Outlet, Route, Routes } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import { getCategory } from '@/lib/categories'
import { listProducts, setProductCategory } from '@/lib/products'
import { CategoryProductsPage } from './CategoryProductsPage'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
vi.mock('@/lib/categories', () => ({ getCategory: vi.fn() }))
vi.mock('@/lib/products', () => ({
  listProducts: vi.fn(),
  setProductCategory: vi.fn().mockResolvedValue(undefined),
}))
// The page renders CategoryEditDialog + ProductPickerDialog; keep them light.
vi.mock('./CategoryEditDialog', () => ({ CategoryEditDialog: () => null }))

const site: SiteConfig = {
  slug: 'cozycorner',
  label: 'CozyCorner',
  projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test',
  schema: 'cozycorner',
  bucket: 'cozycorner-photos',
}

const category = {
  id: 'cat-1',
  name: 'Vases',
  slug: 'vases',
  image_path: null,
  position: 0,
  hero_title: null,
  hero_description: null,
  seo_title: null,
  seo_description: null,
}

const products = [
  { id: 'p1', title: 'Blue Vase', created_at: '', brand: null, category: 'Vases', image_path: null, folder_id: null },
  { id: 'p2', title: 'Red Lamp', created_at: '', brand: null, category: 'Lamps', image_path: null, folder_id: null },
  { id: 'p3', title: 'Free Agent', created_at: '', brand: null, category: null, image_path: null, folder_id: null },
]

function renderPage() {
  const qc = new QueryClient()
  render(
    <QueryClientProvider client={qc}>
      <MemoryRouter initialEntries={['/cozycorner/categories/cat-1']}>
        <Routes>
          <Route element={<Outlet context={site} />}>
            <Route path="/:siteSlug/categories/:categoryId" element={<CategoryProductsPage />} />
            <Route path="/:siteSlug/products/:productId" element={<div>product editor</div>} />
          </Route>
        </Routes>
      </MemoryRouter>
    </QueryClientProvider>,
  )
}

describe('<CategoryProductsPage />', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    vi.mocked(getCategory).mockResolvedValue(category as never)
    vi.mocked(listProducts).mockResolvedValue(products as never)
  })

  it('lists only products whose category matches, with a link carrying ?from=category', async () => {
    renderPage()
    const link = await screen.findByRole('link', { name: /Blue Vase/ })
    expect(link).toHaveAttribute('href', '/cozycorner/products/p1?from=category:cat-1')
    expect(screen.queryByText('Red Lamp')).not.toBeInTheDocument()
    expect(screen.queryByText('Free Agent')).not.toBeInTheDocument()
  })

  it('adds an uncategorized product to the category via setProductCategory', async () => {
    renderPage()
    await screen.findByText('Blue Vase')
    fireEvent.click(screen.getByRole('button', { name: /Add product/ }))
    // ProductPickerDialog is the real component; it lists only non-excluded products.
    const dialog = await screen.findByRole('dialog')
    fireEvent.click(await within(dialog).findByText('Free Agent'))
    fireEvent.click(within(dialog).getByRole('button', { name: /Add 1 product/ }))
    await waitFor(() => expect(setProductCategory).toHaveBeenCalledWith(site, 'p3', 'Vases'))
  })
})
```

- [ ] **Step 3: Run test — expect FAIL**

Run: `npm test -- src/features/taxonomy/CategoryProductsPage.test.tsx`
Expected: FAIL — `./CategoryProductsPage` does not exist.

- [ ] **Step 4: Create `CategoryProductsPage.tsx`**

Create `src/features/taxonomy/CategoryProductsPage.tsx`:

```tsx
import { useState } from 'react'
import { Link, useOutletContext, useParams } from 'react-router-dom'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { ArrowLeft, ImageOff, Pencil, Plus } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { getCategory } from '@/lib/categories'
import { listProducts, setProductCategory } from '@/lib/products'
import { resolveImageUrl } from '@/lib/images'
import { humanizeError } from '@/lib/errors'
import { Button } from '@/components/ui/button'
import { Skeleton } from '@/components/ui/skeleton'
import { CategoryEditDialog } from './CategoryEditDialog'
import { ProductPickerDialog } from '@/features/posts/ProductPickerDialog'

// Страница товаров одной категории (роут categories/:categoryId). Связь категория↔товар
// денормализована по имени: products.category === category.name. Здесь: список товаров
// категории, правка категории (переиспользуем CategoryEditDialog) и добавление товара
// (пикер показывает только товары БЕЗ категории — экран ничего не «уводит» из чужих
// категорий). Клик по товару → его правка с ?from=category:<id>, откуда «Back to
// category» вернёт сюда.
export function CategoryProductsPage() {
  const site = useOutletContext<SiteConfig>()
  const { categoryId } = useParams()
  const queryClient = useQueryClient()

  const [editOpen, setEditOpen] = useState(false)
  const [pickerOpen, setPickerOpen] = useState(false)

  const { data: category, isPending: catPending, error: catError } = useQuery({
    queryKey: ['categories', site.slug, categoryId],
    queryFn: () => getCategory(site, categoryId!),
    enabled: Boolean(categoryId),
  })

  const { data: products, error: prodError } = useQuery({
    queryKey: ['products', site.slug],
    queryFn: () => listProducts(site),
  })

  const inCategory = (products ?? []).filter((p) => category && p.category === category.name)
  // Пикер «добавить товар» показывает только товары без категории:
  // исключаем все, у кого категория уже задана (в т.ч. эта же категория).
  const excludeIds = (products ?? []).filter((p) => p.category !== null).map((p) => p.id)

  const add = useMutation({
    mutationFn: async (ids: string[]) => {
      for (const id of ids) await setProductCategory(site, id, category!.name)
    },
    onSuccess: (_, ids) => {
      queryClient.invalidateQueries({ queryKey: ['products', site.slug] })
      queryClient.invalidateQueries({ queryKey: ['categories', site.slug] })
      toast.success(ids.length === 1 ? 'Product added' : `${ids.length} products added`)
    },
    onError: (e) => toast.error(humanizeError(e, 'Failed to add product')),
  })

  const backToCategories = (
    <Link
      to={`/${site.slug}/categories`}
      className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
    >
      <ArrowLeft className="size-4" />
      Back to categories
    </Link>
  )

  if (catError || prodError) {
    return (
      <div className="flex flex-col gap-4">
        {backToCategories}
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          Failed to load: {(catError ?? prodError)?.message}
        </p>
      </div>
    )
  }

  if (catPending) {
    return (
      <div className="flex flex-col gap-4">
        {backToCategories}
        <Skeleton className="h-10 w-64 rounded-md" />
      </div>
    )
  }

  if (!category) {
    return (
      <div className="flex flex-col gap-4">
        {backToCategories}
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          Category not found.
        </p>
      </div>
    )
  }

  return (
    <div className="flex flex-col gap-4">
      {backToCategories}

      <div className="flex items-center gap-3">
        <span className="flex size-9 shrink-0 items-center justify-center overflow-hidden rounded border bg-muted/30">
          {category.image_path ? (
            <img
              src={resolveImageUrl(site, category.image_path)}
              alt=""
              className="size-full object-contain"
            />
          ) : (
            <ImageOff className="size-4 text-muted-foreground" />
          )}
        </span>
        <h1 className="font-heading text-lg font-medium">{category.name}</h1>
        <span className="text-sm text-muted-foreground">
          {inCategory.length} product{inCategory.length === 1 ? '' : 's'}
        </span>
        <div className="ml-auto flex items-center gap-2">
          <Button variant="outline" onClick={() => setEditOpen(true)}>
            <Pencil />
            Edit category
          </Button>
          <Button onClick={() => setPickerOpen(true)}>
            <Plus />
            Add product
          </Button>
        </div>
      </div>

      {inCategory.length === 0 ? (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          No products in this category yet.
        </p>
      ) : (
        <ul className="flex max-w-2xl flex-col">
          {inCategory.map((p) => (
            <li key={p.id} className="flex items-center gap-1">
              <Link
                to={`/${site.slug}/products/${p.id}?from=category:${category.id}`}
                className="flex min-w-0 flex-1 items-center gap-3 rounded-md px-3 py-2 transition-colors hover:bg-accent/50"
              >
                {p.image_path ? (
                  <img
                    src={resolveImageUrl(site, p.image_path)}
                    alt=""
                    loading="lazy"
                    className="size-9 shrink-0 rounded border object-cover"
                  />
                ) : (
                  <span className="flex size-9 shrink-0 items-center justify-center rounded border bg-muted/30">
                    <ImageOff className="size-4 text-muted-foreground" />
                  </span>
                )}
                <span className="min-w-0 flex-1 truncate text-sm font-medium">{p.title}</span>
                {p.brand && (
                  <span className="shrink-0 text-xs text-muted-foreground">{p.brand}</span>
                )}
              </Link>
            </li>
          ))}
        </ul>
      )}

      <CategoryEditDialog site={site} id={category.id} open={editOpen} onOpenChange={setEditOpen} />
      <ProductPickerDialog
        site={site}
        open={pickerOpen}
        onOpenChange={setPickerOpen}
        excludeIds={excludeIds}
        onAdd={(ids) => add.mutate(ids)}
      />
    </div>
  )
}
```

- [ ] **Step 5: Add the route**

In `src/App.tsx`, add the import with the other feature imports:

```tsx
import { CategoryProductsPage } from '@/features/taxonomy/CategoryProductsPage'
```

Add the route immediately after the existing `<Route path="categories" element={<CategoriesPage />} />` line:

```tsx
            <Route path="categories/:categoryId" element={<CategoryProductsPage />} />
```

- [ ] **Step 6: Run the page test — expect PASS**

Run: `npm test -- src/features/taxonomy/CategoryProductsPage.test.tsx`
Expected: PASS (2 tests). If the picker's Add-button label assertion fails, open `src/features/posts/ProductPickerDialog.tsx` and match the exact button text (it renders `Add {n} product(s)`); adjust the test's button name regex to match, do not change the dialog.

- [ ] **Step 7: Build + full suite + lint**

Run: `npm run build` (expect clean), `npm test` (expect all pass), `npm run lint` (only the 2 known shadcn warnings).

- [ ] **Step 8: Commit**

```bash
git add src/lib/products.ts src/features/taxonomy/CategoryProductsPage.tsx src/features/taxonomy/CategoryProductsPage.test.tsx src/App.tsx
git commit -m "feat(taxonomy): category products page (list, edit, add uncategorized product)"
```

---

## Task 3: "Back to category" in the product editor

**Files:**
- Modify: `src/features/products/ProductEditPage.tsx`
- Test: `src/features/products/ProductEditPage.test.tsx` (create if absent)

- [ ] **Step 1: Write the failing test**

Create (or add to) `src/features/products/ProductEditPage.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { MemoryRouter, Outlet, Route, Routes } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import { getProduct } from '@/lib/products'
import { ProductEditPage } from './ProductEditPage'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
vi.mock('@/lib/products', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/lib/products')>()
  return { ...actual, getProduct: vi.fn() }
})

const site: SiteConfig = {
  slug: 'cozycorner',
  label: 'CozyCorner',
  projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test',
  schema: 'cozycorner',
  bucket: 'cozycorner-photos',
}

const product = {
  id: 'p1',
  created_at: '',
  title: 'Blue Vase',
  price: 10,
  image_path: null,
  referral_url: 'https://example.com',
  brand: null,
  category: 'Vases',
  slug: 'blue-vase',
  description: null,
  image_style: 'cutout' as const,
  seo_title: null,
  seo_description: null,
}

function renderAt(url: string) {
  const qc = new QueryClient()
  render(
    <QueryClientProvider client={qc}>
      <MemoryRouter initialEntries={[url]}>
        <Routes>
          <Route element={<Outlet context={site} />}>
            <Route path="/:siteSlug/products/:productId" element={<ProductEditPage />} />
          </Route>
        </Routes>
      </MemoryRouter>
    </QueryClientProvider>,
  )
}

describe('<ProductEditPage /> back link', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    vi.mocked(getProduct).mockResolvedValue(product as never)
  })

  it('shows "Back to products" by default', async () => {
    renderAt('/cozycorner/products/p1')
    const link = await screen.findByRole('link', { name: /Back to products/ })
    expect(link).toHaveAttribute('href', '/cozycorner/products')
  })

  it('shows "Back to category" when arrived from a category', async () => {
    renderAt('/cozycorner/products/p1?from=category:cat-1')
    const link = await screen.findByRole('link', { name: /Back to category/ })
    expect(link).toHaveAttribute('href', '/cozycorner/categories/cat-1')
  })
})
```

- [ ] **Step 2: Run test — expect the second test to FAIL**

Run: `npm test -- src/features/products/ProductEditPage.test.tsx`
Expected: the "Back to category" test fails (page always renders "Back to products").

- [ ] **Step 3: Make the back link context-aware**

In `src/features/products/ProductEditPage.tsx`:

Add `useSearchParams` to the existing `react-router-dom` import (it currently imports `Link, useBlocker, useNavigate, useOutletContext, useParams`):

```tsx
import {
  Link,
  useBlocker,
  useNavigate,
  useOutletContext,
  useParams,
  useSearchParams,
} from 'react-router-dom'
```

In the `ProductEditPage` wrapper component (the one with `useParams`), compute the back target from the `from` param, right after `const { productId } = useParams()`:

```tsx
  const [searchParams] = useSearchParams()
  const from = searchParams.get('from')
  const backCategoryId = from?.startsWith('category:') ? from.slice('category:'.length) : null
  const backTo = backCategoryId
    ? `/${site.slug}/categories/${backCategoryId}`
    : `/${site.slug}/products`
  const backLabel = backCategoryId ? 'Back to category' : 'Back to products'
```

Update the wrapper's loading/error back link (the `<Link to={`/${site.slug}/products`}>…Back to products</Link>` shown when `!showForm`) to use the computed values:

```tsx
        <Link
          to={backTo}
          className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          {backLabel}
        </Link>
```

Pass the values into the form. Change the `<ProductForm site={site} product={product ?? null} />` render to:

```tsx
        <ProductForm site={site} product={product ?? null} backTo={backTo} backLabel={backLabel} />
```

Extend `ProductForm`'s props type and signature to accept them:

```tsx
function ProductForm({
  site,
  product,
  backTo,
  backLabel,
}: {
  site: SiteConfig
  product: Product | null
  backTo: string
  backLabel: string
}) {
```

Update the form's sticky back link (currently `<Link to={`/${site.slug}/products`}>…Back to products</Link>`) to:

```tsx
        <Link
          to={backTo}
          className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          {backLabel}
        </Link>
```

Update the delete-success navigation so deleting a product returns to where you came from — change `navigate(`/${site.slug}/products`, { replace: true })` inside the `remove` mutation's `onSuccess` to:

```tsx
      navigate(backTo, { replace: true })
```

Leave the create-success navigation (`navigate(`/${site.slug}/products/${created.id}`, ...)`) as-is: creating a new product is not reached from a category page.

- [ ] **Step 4: Run test — expect PASS**

Run: `npm test -- src/features/products/ProductEditPage.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Build + full suite + lint**

Run: `npm run build` (clean), `npm test` (all pass), `npm run lint` (only 2 known warnings).

- [ ] **Step 6: Commit**

```bash
git add src/features/products/ProductEditPage.tsx src/features/products/ProductEditPage.test.tsx
git commit -m "feat(products): context-aware Back to category link from category page"
```

---

## Task 4: Full verification

**Files:** none (verification only).

- [ ] **Step 1:** `npm test` — all pass (new files: TaxonomyManager, CategoryProductsPage, ProductEditPage tests).
- [ ] **Step 2:** `npm run lint` — only the 2 known shadcn warnings; delete any newly-unused imports.
- [ ] **Step 3:** `npm run build` — succeeds.
- [ ] **Step 4: Manual check** (`npm run dev`, needs admin login):
  - Categories list → click a category → lands on its products page (URL `…/categories/<id>`), not the edit modal.
  - "Edit category" opens the existing dialog; saving a rename updates the page title and keeps the product list correct.
  - "Add product" lists only products with no category; adding one makes it appear in the list and disappear from the picker.
  - Click a product → product editor shows "Back to category"; clicking it returns to the same category page. Deleting the product also returns to the category page.
  - A product opened normally from Products still shows "Back to products".
  - "New category" (from the Categories list) still opens the create modal; brands are unchanged (still inline rename + delete, no navigation).
- [ ] **Step 5 (optional):** propose Playwright e2e (web.admin, port 5173): category → open → add product → open product → Back to category.

---

## Self-Review

**Spec coverage:**
- Click category → products page route → Task 1 (rowTo→Link) + Task 2 (route + page). ✓
- Products filtered by name (`p.category === category.name`) → Task 2 `inCategory`. ✓
- Edit category from the page → Task 2 `CategoryEditDialog` via "Edit category". ✓
- Add uncategorized product (`setProductCategory`, picker shows only `category === null`) → Task 2 `excludeIds` + `add` mutation. ✓
- Product click → editor with "Back to category" via `?from=category:<id>` → Task 2 links + Task 3 parsing. ✓
- Back defaults to "Back to products" otherwise → Task 3 default branch. ✓
- Single-category model unchanged; move-between-categories only via product editor → picker excludes categorized products (Task 2). ✓
- Brands unchanged → `rowTo` only on `categoryConfig` (Task 1). ✓

**Placeholder scan:** none — all steps carry complete code and exact commands.

**Type consistency:** `setProductCategory(site, id, category)` signature identical in `products.ts` (Task 2 Step 1) and its call in `CategoryProductsPage`/test. `rowTo?: (item: TaxonomyItem) => string` added in Task 1 and consumed in `TaxonomyManager`/test. `ProductForm` gains `backTo: string`, `backLabel: string` (Task 3), passed from the wrapper. `getCategory` returns `CategoryFull` (has `id`, `name`, `image_path`) — used for the list filter and links. `ProductListItem` fields (`id`, `title`, `brand`, `category`, `image_path`) match the page's usage.
