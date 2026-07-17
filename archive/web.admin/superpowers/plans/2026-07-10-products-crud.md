# Products CRUD (v1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Раздел Products: список товаров одной колонкой (только названия) → детальная страница с формой всех полей → редактирование, создание, удаление. Спека: `docs/superpowers/specs/2026-07-10-products-crud-design.md`.

**Architecture:** Слой данных `src/lib/products.ts` (per-site через `getDb(site)`), страницы `src/features/products/{ProductsPage,ProductEditPage}.tsx`, маршруты в `App.tsx` внутри `SiteLayout` (пункт меню Products уже есть). TanStack Query + инвалидация `['products', site.slug]`, формы RHF + Zod v4 + Field, тосты sonner.

**Tech Stack:** Vite 8 + React 19 + TS, shadcn/ui (radix-nova), @supabase/supabase-js, react-router-dom, TanStack Query, RHF + Zod v4.

**Отступления от стандартного процесса (осознанные):**
- В репозитории нет тест-раннера (шаблон без vitest; так строился и media-менеджер). TDD-шаги заменены проверками `npm run build` + `npm run lint` + ручной e2e в конце — паттерн проекта.
- По CLAUDE.md коммитить можно только с явного разрешения пользователя. Шаги «Commit» выполнять ТОЛЬКО после того, как разрешение получено в чате; иначе пропустить и сообщить пользователю.

---

### Task 1: shadcn-компоненты select и textarea

**Files:**
- Create: `src/components/ui/select.tsx` (генерирует CLI)
- Create: `src/components/ui/textarea.tsx` (генерирует CLI)

- [ ] **Step 1: Добавить компоненты через shadcn CLI**

Run: `npx shadcn@latest add select textarea`
Expected: файлы `src/components/ui/select.tsx` и `src/components/ui/textarea.tsx` созданы (конфиг `components.json` уже настроен, style radix-nova).

- [ ] **Step 2: Проверить сборку**

Run: `npm run build`
Expected: успех, без ошибок TS.

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/components/ui/select.tsx src/components/ui/textarea.tsx package.json package-lock.json
git commit -m "chore: add shadcn select and textarea components"
```

---

### Task 2: Слой данных `src/lib/products.ts`

**Files:**
- Create: `src/lib/products.ts`

- [ ] **Step 1: Написать модуль целиком**

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

// Строка products (спека §8). slug автогенерируется БД из title, updated_at нет.
export type Product = {
  id: string;
  created_at: string;
  title: string;
  price: number;
  image_path: string | null;
  referral_url: string;
  brand: string | null;
  category: string | null; // = categories.name
  slug: string;
  description: string | null;
  image_style: "photo" | "cutout";
  seo_title: string | null;
  seo_description: string | null;
};

// Строка списка — только то, что нужно колонке названий.
export type ProductListItem = Pick<Product, "id" | "title" | "created_at">;

// Payload create/update: без id/created_at/slug (slug генерирует БД).
export type ProductInput = Omit<Product, "id" | "created_at" | "slug">;

export async function listProducts(site: SiteConfig): Promise<ProductListItem[]> {
  const { data, error } = await getDb(site)
    .from("products")
    .select("id,title,created_at")
    .order("created_at", { ascending: false });
  if (error) throw error;
  return (data ?? []) as ProductListItem[];
}

export async function getProduct(site: SiteConfig, id: string): Promise<Product | null> {
  const { data, error } = await getDb(site)
    .from("products")
    .select("*")
    .eq("id", id)
    .maybeSingle();
  if (error) throw error;
  return data as Product | null;
}

export async function createProduct(site: SiteConfig, input: ProductInput): Promise<Product> {
  const { data, error } = await getDb(site)
    .from("products")
    .insert(input)
    .select()
    .single();
  if (error) throw error;
  return data as Product;
}

export async function updateProduct(
  site: SiteConfig,
  id: string,
  input: ProductInput,
): Promise<void> {
  const { error } = await getDb(site).from("products").update(input).eq("id", id);
  if (error) throw error;
}

export async function deleteProduct(site: SiteConfig, id: string): Promise<void> {
  const { error } = await getDb(site).from("products").delete().eq("id", id);
  if (error) throw error;
}

// Имена категорий для select (порядок как в каталоге сайта).
export async function listCategoryNames(site: SiteConfig): Promise<string[]> {
  const { data, error } = await getDb(site)
    .from("categories")
    .select("name")
    .order("position", { ascending: true });
  if (error) throw error;
  return ((data ?? []) as { name: string }[]).map((r) => r.name);
}
```

- [ ] **Step 2: Проверить сборку и линт**

Run: `npm run build && npm run lint`
Expected: успех (2 известных warning oxlint в shadcn-файлах — игнорируем).

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/lib/products.ts
git commit -m "feat: add products data layer (list/get/create/update/delete)"
```

---

### Task 3: Список `ProductsPage` + маршрут

**Files:**
- Create: `src/features/products/ProductsPage.tsx`
- Modify: `src/App.tsx` (заменить заглушку products)

- [ ] **Step 1: Написать ProductsPage**

```tsx
import { Link, useOutletContext } from 'react-router-dom'
import { useQuery } from '@tanstack/react-query'
import { Plus } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listProducts } from '@/lib/products'
import { Button } from '@/components/ui/button'
import { Skeleton } from '@/components/ui/skeleton'

// Список товаров: одна колонка строк-ссылок, только названия (спека v1).
export function ProductsPage() {
  const site = useOutletContext<SiteConfig>()

  const { data: products, isPending, error } = useQuery({
    queryKey: ['products', site.slug],
    queryFn: () => listProducts(site),
  })

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">Products</h1>
        {products && (
          <span className="text-sm text-muted-foreground">{products.length}</span>
        )}
        <Button asChild className="ml-auto">
          <Link to="new">
            <Plus />
            New product
          </Link>
        </Button>
      </div>

      {error && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          Failed to load products: {error.message}
        </p>
      )}

      {isPending && !error && (
        <div className="flex flex-col gap-1">
          {Array.from({ length: 8 }, (_, i) => (
            <Skeleton key={i} className="h-9 w-full rounded-md" />
          ))}
        </div>
      )}

      {products &&
        (products.length === 0 ? (
          <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
            No products yet.
          </p>
        ) : (
          <ul className="flex flex-col">
            {products.map((product) => (
              <li key={product.id}>
                <Link
                  to={product.id}
                  className="block rounded-md px-3 py-2 text-sm font-medium transition-colors hover:bg-accent/50"
                >
                  {product.title}
                </Link>
              </li>
            ))}
          </ul>
        ))}
    </div>
  )
}
```

- [ ] **Step 2: Подключить маршрут в App.tsx**

В `src/App.tsx` добавить импорт и заменить строку заглушки:

```tsx
import { ProductsPage } from '@/features/products/ProductsPage'
```

Вместо `<Route path="products" element={<ComingSoonPage section="Products" />} />`:

```tsx
<Route path="products" element={<ProductsPage />} />
```

- [ ] **Step 3: Проверить сборку и линт**

Run: `npm run build && npm run lint`
Expected: успех.

- [ ] **Step 4: Commit (только с разрешения пользователя)**

```bash
git add src/features/products/ProductsPage.tsx src/App.tsx
git commit -m "feat: products list page (single-column titles)"
```

---

### Task 4: Детальная страница `ProductEditPage` (edit + new + delete)

**Files:**
- Create: `src/features/products/ProductEditPage.tsx`
- Modify: `src/App.tsx` (маршруты `products/new` и `products/:productId`)

- [ ] **Step 1: Написать ProductEditPage**

Одна страница на оба режима: без `productId` в params — создание. Форма выделена в
`ProductForm`, монтируется только когда данные загружены (defaultValues из product).
Поля формы — строки (RHF), конвертация в `ProductInput` (число/null) — в `toInput`.
Radix Select не принимает пустое value у Item — для «без категории» сентинел `__none__`.

```tsx
import { Controller, useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Link, useNavigate, useOutletContext, useParams } from 'react-router-dom'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { ArrowLeft, ImageOff, Trash2 } from 'lucide-react'
import { toast } from 'sonner'
import { useState } from 'react'
import type { SiteConfig } from '@/config/sites'
import {
  createProduct,
  deleteProduct,
  getProduct,
  listCategoryNames,
  updateProduct,
  type Product,
  type ProductInput,
} from '@/lib/products'
import { resolveImageUrl } from '@/lib/images'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Field, FieldError, FieldGroup, FieldLabel } from '@/components/ui/field'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from '@/components/ui/alert-dialog'

const NONE = '__none__' // radix SelectItem не принимает пустое value

const productSchema = z.object({
  title: z.string().trim().min(1, 'Title is required'),
  price: z
    .string()
    .trim()
    .min(1, 'Price is required')
    .refine(
      (v) => !Number.isNaN(Number(v)) && Number(v) >= 0,
      'Must be a non-negative number',
    ),
  referral_url: z.url('Must be a valid URL'),
  brand: z.string(),
  category: z.string(),
  description: z.string(),
  image_path: z.string().trim(),
  image_style: z.enum(['photo', 'cutout']),
  seo_title: z.string(),
  seo_description: z.string(),
})

type ProductFormValues = z.infer<typeof productSchema>

// Пустые optional-поля пишем как null (контракт спеки), price — числом.
function toInput(values: ProductFormValues): ProductInput {
  const orNull = (v: string) => (v.trim() ? v.trim() : null)
  return {
    title: values.title.trim(),
    price: Number(values.price),
    referral_url: values.referral_url.trim(),
    brand: orNull(values.brand),
    category: values.category === NONE ? null : values.category,
    description: orNull(values.description),
    image_path: orNull(values.image_path),
    image_style: values.image_style,
    seo_title: orNull(values.seo_title),
    seo_description: orNull(values.seo_description),
  }
}

function toFormValues(product: Product | null): ProductFormValues {
  return {
    title: product?.title ?? '',
    price: product ? String(product.price) : '',
    referral_url: product?.referral_url ?? '',
    brand: product?.brand ?? '',
    category: product?.category ?? NONE,
    description: product?.description ?? '',
    image_path: product?.image_path ?? '',
    image_style: product?.image_style ?? 'photo',
    seo_title: product?.seo_title ?? '',
    seo_description: product?.seo_description ?? '',
  }
}

export function ProductEditPage() {
  const site = useOutletContext<SiteConfig>()
  const { productId } = useParams()
  const isNew = !productId

  const { data: product, isPending: productPending, error: productError } = useQuery({
    queryKey: ['products', site.slug, productId],
    queryFn: () => getProduct(site, productId!),
    enabled: !isNew,
  })

  const { data: categories, error: categoriesError } = useQuery({
    queryKey: ['categories', site.slug],
    queryFn: () => listCategoryNames(site),
  })

  const error = productError ?? categoriesError
  const loading = (!isNew && productPending) || categories === undefined

  return (
    <div className="flex flex-col gap-4">
      <Link
        to={`/${site.slug}/products`}
        className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
      >
        <ArrowLeft className="size-4" />
        Back to products
      </Link>

      {error && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          Failed to load: {error.message}
        </p>
      )}

      {!error && loading && <p className="text-sm text-muted-foreground">Loading…</p>}

      {!error && !loading && !isNew && !product && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          Product not found.
        </p>
      )}

      {!error && !loading && (isNew || product) && (
        <ProductForm
          site={site}
          product={product ?? null}
          categories={categories ?? []}
        />
      )}
    </div>
  )
}

function ProductForm({
  site,
  product,
  categories,
}: {
  site: SiteConfig
  product: Product | null
  categories: string[]
}) {
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  const isNew = product === null

  const form = useForm<ProductFormValues>({
    resolver: zodResolver(productSchema),
    defaultValues: toFormValues(product),
  })
  const { errors, isSubmitting } = form.formState
  const imagePath = form.watch('image_path').trim()
  const [imageBroken, setImageBroken] = useState(false)

  const save = useMutation({
    mutationFn: async (values: ProductFormValues) => {
      const input = toInput(values)
      if (isNew) return createProduct(site, input)
      await updateProduct(site, product.id, input)
      return null
    },
    onSuccess: (created) => {
      queryClient.invalidateQueries({ queryKey: ['products', site.slug] })
      toast.success(isNew ? 'Product created' : 'Product saved')
      if (created) navigate(`/${site.slug}/products/${created.id}`, { replace: true })
    },
    onError: (e) =>
      toast.error(e instanceof Error ? e.message : 'Failed to save product'),
  })

  const remove = useMutation({
    mutationFn: () => deleteProduct(site, product!.id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['products', site.slug] })
      toast.success('Product deleted')
      navigate(`/${site.slug}/products`, { replace: true })
    },
    onError: (e) =>
      toast.error(e instanceof Error ? e.message : 'Failed to delete product'),
  })

  return (
    <div className="flex flex-col gap-6 md:flex-row">
      {/* Превью картинки; путь пока правится текстом (пикер — следующая итерация) */}
      <div className="w-full shrink-0 md:w-64">
        <div className="flex aspect-square items-center justify-center overflow-hidden rounded-xl border bg-muted/30">
          {imagePath && !imageBroken ? (
            <img
              src={resolveImageUrl(site, imagePath)}
              alt={form.watch('title') || 'Product image'}
              className="size-full object-contain"
              onError={() => setImageBroken(true)}
            />
          ) : (
            <ImageOff className="size-8 text-muted-foreground" />
          )}
        </div>
        {product && (
          <p className="mt-2 truncate text-xs text-muted-foreground" title={product.slug}>
            Slug: {product.slug}
          </p>
        )}
      </div>

      <form
        onSubmit={form.handleSubmit((values) => save.mutate(values))}
        noValidate
        className="min-w-0 max-w-xl flex-1"
      >
        <FieldGroup>
          <Field data-invalid={!!errors.title}>
            <FieldLabel htmlFor="title">Title</FieldLabel>
            <Input id="title" aria-invalid={!!errors.title} {...form.register('title')} />
            <FieldError errors={[errors.title]} />
          </Field>

          <div className="grid grid-cols-2 gap-4">
            <Field data-invalid={!!errors.price}>
              <FieldLabel htmlFor="price">Price</FieldLabel>
              <Input
                id="price"
                type="number"
                step="0.01"
                min="0"
                aria-invalid={!!errors.price}
                {...form.register('price')}
              />
              <FieldError errors={[errors.price]} />
            </Field>
            <Field data-invalid={!!errors.brand}>
              <FieldLabel htmlFor="brand">Brand</FieldLabel>
              <Input id="brand" {...form.register('brand')} />
            </Field>
          </div>

          <Field data-invalid={!!errors.referral_url}>
            <FieldLabel htmlFor="referral_url">Referral URL</FieldLabel>
            <Input
              id="referral_url"
              type="url"
              placeholder="https://…"
              aria-invalid={!!errors.referral_url}
              {...form.register('referral_url')}
            />
            <FieldError errors={[errors.referral_url]} />
          </Field>

          <div className="grid grid-cols-2 gap-4">
            <Field>
              <FieldLabel htmlFor="category">Category</FieldLabel>
              <Controller
                control={form.control}
                name="category"
                render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger id="category" className="w-full">
                      <SelectValue />
                    </SelectTrigger>
                    <SelectContent>
                      <SelectItem value={NONE}>None</SelectItem>
                      {categories.map((name) => (
                        <SelectItem key={name} value={name}>
                          {name}
                        </SelectItem>
                      ))}
                    </SelectContent>
                  </Select>
                )}
              />
            </Field>
            <Field>
              <FieldLabel htmlFor="image_style">Image style</FieldLabel>
              <Controller
                control={form.control}
                name="image_style"
                render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger id="image_style" className="w-full">
                      <SelectValue />
                    </SelectTrigger>
                    <SelectContent>
                      <SelectItem value="photo">Photo</SelectItem>
                      <SelectItem value="cutout">Cutout</SelectItem>
                    </SelectContent>
                  </Select>
                )}
              />
            </Field>
          </div>

          <Field data-invalid={!!errors.image_path}>
            <FieldLabel htmlFor="image_path">Image path</FieldLabel>
            <Input
              id="image_path"
              placeholder="flat key in bucket or external URL"
              {...form.register('image_path', {
                onChange: () => setImageBroken(false),
              })}
            />
          </Field>

          <Field>
            <FieldLabel htmlFor="description">Description</FieldLabel>
            <Textarea id="description" rows={4} {...form.register('description')} />
          </Field>

          <Field>
            <FieldLabel htmlFor="seo_title">SEO title</FieldLabel>
            <Input id="seo_title" {...form.register('seo_title')} />
          </Field>
          <Field>
            <FieldLabel htmlFor="seo_description">SEO description</FieldLabel>
            <Textarea id="seo_description" rows={3} {...form.register('seo_description')} />
          </Field>

          <div className="flex items-center gap-3">
            <Button type="submit" disabled={isSubmitting || save.isPending}>
              {save.isPending ? 'Saving…' : 'Save'}
            </Button>
            {!isNew && (
              <AlertDialog>
                <AlertDialogTrigger asChild>
                  <Button type="button" variant="destructive" disabled={remove.isPending}>
                    <Trash2 />
                    {remove.isPending ? 'Deleting…' : 'Delete'}
                  </Button>
                </AlertDialogTrigger>
                <AlertDialogContent>
                  <AlertDialogHeader>
                    <AlertDialogTitle>Delete this product?</AlertDialogTitle>
                    <AlertDialogDescription>
                      “{product.title}” will be permanently deleted. If it is used in
                      blog post product sections, it will disappear from them.
                    </AlertDialogDescription>
                  </AlertDialogHeader>
                  <AlertDialogFooter>
                    <AlertDialogCancel>Cancel</AlertDialogCancel>
                    <AlertDialogAction
                      variant="destructive"
                      onClick={() => remove.mutate()}
                    >
                      Delete
                    </AlertDialogAction>
                  </AlertDialogFooter>
                </AlertDialogContent>
              </AlertDialog>
            )}
          </div>
        </FieldGroup>
      </form>
    </div>
  )
}
```

Примечание: если в установленном `alert-dialog.tsx` у `AlertDialogAction` нет пропа
`variant` (зависит от версии registry) — заменить на
`className={buttonVariants({ variant: 'destructive' })}` с импортом `buttonVariants`
из `@/components/ui/button` (посмотреть, как сделано в `MediaCard.tsx` /
`MediaDetailsDialog.tsx`, там уже есть destructive-подтверждение).

- [ ] **Step 2: Добавить маршруты в App.tsx**

Импорт:

```tsx
import { ProductEditPage } from '@/features/products/ProductEditPage'
```

После строки `<Route path="products" element={<ProductsPage />} />`:

```tsx
<Route path="products/new" element={<ProductEditPage />} />
<Route path="products/:productId" element={<ProductEditPage />} />
```

- [ ] **Step 3: Проверить сборку и линт**

Run: `npm run build && npm run lint`
Expected: успех.

- [ ] **Step 4: Commit (только с разрешения пользователя)**

```bash
git add src/features/products/ProductEditPage.tsx src/App.tsx
git commit -m "feat: product detail page with edit, create and delete"
```

---

### Task 5: Документация и финальная проверка

**Files:**
- Modify: `CLAUDE.md` (раздел «Статус» и «Следующие шаги»)
- Modify: `docs/ux-ui.md` (карта маршрутов + раздел Products вместо заглушки)

- [ ] **Step 1: Обновить CLAUDE.md**

В «Сделано» добавить пункт о фазе 3 (products v1: список одной колонкой, детальная
страница, create/update/delete, select категорий, данные — `lib/products.ts`; спека —
`docs/superpowers/specs/2026-07-10-products-crud-design.md`). В «Следующие шаги»
заменить пункт про фазу 3 на: пикер картинки из Media для товара → categories →
posts (+ секции) → pages/hero/footer.

- [ ] **Step 2: Обновить docs/ux-ui.md**

В карте маршрутов заменить `/:siteSlug/products — заглушка «Coming soon»` на три
маршрута (`products`, `products/new`, `products/:productId`) и кратко описать
раздел Products по образцу описания Media (список → детальная страница → форма).

- [ ] **Step 3: Финальная проверка**

Run: `npm run build && npm run lint`
Expected: успех.

Ручная e2e (пользователь или dev-сервер `npm run dev` под админом):
1. `/cozycorner/products` — список названий, счётчик, новые сверху.
2. Клик по товару — форма заполнена, превью картинки отображается.
3. Изменить title → Save → toast «Product saved», в списке имя обновилось.
4. New product: пустая форма, валидация (пустой title / кривой URL / отрицательная
   цена — ошибки), после Save — переход на страницу товара, slug сгенерирован.
5. Delete созданного товара: AlertDialog → подтверждение → возврат к списку, товара нет.
6. Кривой id в URL — «Product not found».

- [ ] **Step 4: Commit (только с разрешения пользователя)**

```bash
git add CLAUDE.md docs/ux-ui.md docs/superpowers/specs/2026-07-10-products-crud-design.md docs/superpowers/plans/2026-07-10-products-crud.md
git commit -m "docs: products CRUD v1 spec, plan and status update"
```
