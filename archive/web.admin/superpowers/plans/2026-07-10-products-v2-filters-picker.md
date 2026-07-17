# Products v2 (list filters + image picker) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Панель поиска + фильтры Category/Brand на списке товаров; диалог-пикер картинок из медиа-библиотеки (с Upload) на детальной странице. Спека: `docs/superpowers/specs/2026-07-10-products-v2-filters-picker-design.md`.

**Architecture:** Фильтрация списка — клиентская поверх расширенного `listProducts` (+brand, +category). Пикер — новый управляемый `ImagePickerDialog` поверх существующих `listImages`/`uploadImage` (`lib/media`), общий query-кэш `['media', site.slug]` с разделом Media; выбор пишет `image_path` в форму через `setValue`, сохранение — кнопкой Save.

**Tech Stack:** как в products v1 (Vite + React 19 + TS, shadcn/ui, TanStack Query, RHF).

**Отступления (как в плане v1):** тест-раннера в репо нет — проверка `npm run build` + `npm run lint` + ручная e2e. Шаги Commit — только после явного разрешения пользователя.

---

### Task 1: Расширить `listProducts` (brand, category)

**Files:**
- Modify: `src/lib/products.ts` (тип `ProductListItem` и функция `listProducts`)

- [ ] **Step 1: Обновить тип и select**

Заменить объявление типа:

```ts
// Строка списка: название + поля для фильтров Category/Brand.
export type ProductListItem = Pick<
  Product,
  "id" | "title" | "created_at" | "brand" | "category"
>;
```

В `listProducts` заменить select:

```ts
    .select("id,title,created_at,brand,category")
```

- [ ] **Step 2: Проверить сборку**

Run: `npm run build`
Expected: успех.

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/lib/products.ts
git commit -m "feat: include brand and category in products list query"
```

---

### Task 2: Панель поиска и фильтров в `ProductsPage`

**Files:**
- Modify: `src/features/products/ProductsPage.tsx` (полная замена содержимого)

- [ ] **Step 1: Переписать ProductsPage**

```tsx
import { useDeferredValue, useMemo, useState } from 'react'
import { Link, useOutletContext } from 'react-router-dom'
import { useQuery } from '@tanstack/react-query'
import { Loader2, Plus, Search } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listCategoryNames, listProducts } from '@/lib/products'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'

const ALL = '__all__' // radix SelectItem не принимает пустое value

// Список товаров: одна колонка строк, поиск по названию + фильтры Category/Brand.
export function ProductsPage() {
  const site = useOutletContext<SiteConfig>()

  const [search, setSearch] = useState('')
  const [category, setCategory] = useState(ALL)
  const [brand, setBrand] = useState(ALL)
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const { data: products, isPending, error } = useQuery({
    queryKey: ['products', site.slug],
    queryFn: () => listProducts(site),
  })

  // Категории — полный справочник сайта (как в форме товара, общий кэш).
  const { data: categories } = useQuery({
    queryKey: ['categories', site.slug],
    queryFn: () => listCategoryNames(site),
  })

  // Бренды — только реально встречающиеся в товарах.
  const brands = useMemo(() => {
    const unique = new Set<string>()
    for (const p of products ?? []) if (p.brand) unique.add(p.brand)
    return [...unique].sort((a, b) => a.localeCompare(b))
  }, [products])

  const query = deferredSearch.trim().toLowerCase()
  const hasFilter = Boolean(query) || category !== ALL || brand !== ALL
  const visible = (products ?? []).filter(
    (p) =>
      (!query || p.title.toLowerCase().includes(query)) &&
      (category === ALL || p.category === category) &&
      (brand === ALL || p.brand === brand),
  )

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">Products</h1>
        {products && (
          <span className="text-sm text-muted-foreground">
            {hasFilter ? `${visible.length} / ${products.length}` : products.length}
          </span>
        )}
        <Button asChild className="ml-auto">
          <Link to="new">
            <Plus />
            New product
          </Link>
        </Button>
      </div>

      <div className="flex flex-wrap items-center gap-2">
        <div className="relative w-full max-w-56">
          {isFiltering ? (
            <Loader2 className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 animate-spin text-muted-foreground" />
          ) : (
            <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          )}
          <Input
            type="search"
            placeholder="Search by title…"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="h-9 pl-8"
          />
        </div>
        <Select value={category} onValueChange={setCategory}>
          <SelectTrigger className="h-9 w-44" aria-label="Filter by category">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL}>All categories</SelectItem>
            {(categories ?? []).map((name) => (
              <SelectItem key={name} value={name}>
                {name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
        <Select value={brand} onValueChange={setBrand}>
          <SelectTrigger className="h-9 w-44" aria-label="Filter by brand">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL}>All brands</SelectItem>
            {brands.map((name) => (
              <SelectItem key={name} value={name}>
                {name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
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

      {products && products.length === 0 && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          No products yet.
        </p>
      )}

      {products && products.length > 0 && visible.length === 0 && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          No products match your filters.
        </p>
      )}

      {visible.length > 0 && (
        <ul
          className={
            isFiltering
              ? 'flex flex-col opacity-60 transition-opacity'
              : 'flex flex-col transition-opacity'
          }
        >
          {visible.map((product) => (
            <li key={product.id}>
              <Link
                to={product.id}
                className="flex items-baseline justify-between gap-3 rounded-md px-3 py-2 transition-colors hover:bg-accent/50"
              >
                <span className="truncate text-sm font-medium">{product.title}</span>
                {(product.brand || product.category) && (
                  <span className="shrink-0 text-xs text-muted-foreground">
                    {[product.brand, product.category].filter(Boolean).join(' · ')}
                  </span>
                )}
              </Link>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Проверить сборку и линт**

Run: `npm run build && npm run lint`
Expected: успех (2 известных warning).

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/features/products/ProductsPage.tsx
git commit -m "feat: products list search and category/brand filters"
```

---

### Task 3: Компонент `ImagePickerDialog`

**Files:**
- Create: `src/features/products/ImagePickerDialog.tsx`

- [ ] **Step 1: Написать компонент**

```tsx
import { useDeferredValue, useRef, useState } from 'react'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { Loader2, Search, Upload } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { listImages, uploadImage } from '@/lib/media'
import { resolveImageUrl } from '@/lib/images'
import { cn } from '@/lib/utils'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'

const ACCEPT = 'image/jpeg,image/png,image/webp,image/gif'

// Пикер картинки из медиа-библиотеки сайта: выбор из загруженных или Upload.
// Управляемый диалог; выбор отдаёт плоский путь (media.path) через onSelect.
export function ImagePickerDialog({
  site,
  open,
  onOpenChange,
  currentPath,
  onSelect,
}: {
  site: SiteConfig
  open: boolean
  onOpenChange: (open: boolean) => void
  currentPath: string | null
  onSelect: (path: string) => void
}) {
  const queryClient = useQueryClient()
  const fileInputRef = useRef<HTMLInputElement>(null)
  const [search, setSearch] = useState('')
  const deferredSearch = useDeferredValue(search)

  // Тот же ключ, что в разделе Media, — общий кэш; грузим только при открытии.
  const { data: items, isPending, error } = useQuery({
    queryKey: ['media', site.slug],
    queryFn: () => listImages(site),
    enabled: open,
  })

  const query = deferredSearch.trim().toLowerCase()
  const visible = (items ?? []).filter(
    (item) =>
      !query ||
      item.original_name.toLowerCase().includes(query) ||
      item.path.toLowerCase().includes(query),
  )

  const upload = useMutation({
    mutationFn: (file: File) => uploadImage(site, file),
    onSuccess: (row) => {
      queryClient.invalidateQueries({ queryKey: ['media', site.slug] })
      toast.success('Image uploaded')
      onSelect(row.path)
      onOpenChange(false)
    },
    onError: (e) =>
      toast.error(e instanceof Error ? e.message : 'Failed to upload image'),
  })

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-3xl">
        <DialogHeader>
          <DialogTitle>Choose image</DialogTitle>
        </DialogHeader>

        <div className="flex items-center gap-2">
          <div className="relative w-full max-w-56">
            <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
            <Input
              type="search"
              placeholder="Search by name…"
              value={search}
              onChange={(e) => setSearch(e.target.value)}
              className="h-9 pl-8"
            />
          </div>
          <input
            ref={fileInputRef}
            type="file"
            accept={ACCEPT}
            className="hidden"
            onChange={(e) => {
              const file = e.target.files?.[0]
              if (file) upload.mutate(file)
              e.target.value = ''
            }}
          />
          <Button
            type="button"
            variant="outline"
            className="ml-auto"
            onClick={() => fileInputRef.current?.click()}
            disabled={upload.isPending}
          >
            {upload.isPending ? <Loader2 className="animate-spin" /> : <Upload />}
            {upload.isPending ? 'Uploading…' : 'Upload'}
          </Button>
        </div>

        {error && (
          <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
            Failed to load media: {error.message}
          </p>
        )}

        <div className="max-h-[55svh] overflow-y-auto">
          {isPending && !error && (
            <div className="grid grid-cols-3 gap-3 sm:grid-cols-4 lg:grid-cols-5">
              {Array.from({ length: 10 }, (_, i) => (
                <Skeleton key={i} className="aspect-square rounded-lg" />
              ))}
            </div>
          )}

          {items && visible.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              {items.length === 0 ? 'No images yet.' : 'No images match your search.'}
            </p>
          )}

          {visible.length > 0 && (
            <div className="grid grid-cols-3 gap-3 sm:grid-cols-4 lg:grid-cols-5">
              {visible.map((item) => (
                <button
                  key={item.id}
                  type="button"
                  onClick={() => {
                    onSelect(item.path)
                    onOpenChange(false)
                  }}
                  className={cn(
                    'flex flex-col gap-1 rounded-lg p-1 text-left transition-colors hover:bg-accent/50',
                    item.path === currentPath && 'ring-2 ring-ring',
                  )}
                >
                  <img
                    src={resolveImageUrl(site, item.path)}
                    alt={item.original_name}
                    loading="lazy"
                    className="aspect-square w-full rounded-md border object-cover"
                  />
                  <span className="truncate text-xs text-muted-foreground" title={item.original_name}>
                    {item.original_name}
                  </span>
                </button>
              ))}
            </div>
          )}
        </div>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 2: Проверить сборку**

Run: `npm run build`
Expected: успех (компонент ещё не подключён — просто компилируется).

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/features/products/ImagePickerDialog.tsx
git commit -m "feat: image picker dialog over media library"
```

---

### Task 4: Кнопка Gallery в форме товара

**Files:**
- Modify: `src/features/products/ProductEditPage.tsx` (поле Image path + состояние диалога)

- [ ] **Step 1: Подключить пикер**

Добавить импорты:

```tsx
import { ArrowLeft, ImageOff, Images, Trash2 } from 'lucide-react'
import { ImagePickerDialog } from './ImagePickerDialog'
```

(строка с `ArrowLeft, ImageOff, Trash2` уже есть — добавить в неё `Images`.)

В компоненте `ProductForm` рядом с `imageBroken` добавить состояние:

```tsx
  const [pickerOpen, setPickerOpen] = useState(false)
```

Заменить поле Image path (блок `<Field data-invalid={!!errors.image_path}>…</Field>`) на:

```tsx
          <Field data-invalid={!!errors.image_path}>
            <FieldLabel htmlFor="image_path">Image path</FieldLabel>
            <div className="flex gap-2">
              <Input
                id="image_path"
                placeholder="flat key in bucket or external URL"
                {...form.register('image_path', {
                  onChange: () => setImageBroken(false),
                })}
              />
              <Button
                type="button"
                variant="outline"
                onClick={() => setPickerOpen(true)}
              >
                <Images />
                Gallery
              </Button>
            </div>
          </Field>
```

После закрывающего `</form>` (внутри корневого `<div className="flex flex-col gap-6 md:flex-row">`) добавить диалог:

```tsx
      <ImagePickerDialog
        site={site}
        open={pickerOpen}
        onOpenChange={setPickerOpen}
        currentPath={imagePath || null}
        onSelect={(path) => {
          form.setValue('image_path', path, { shouldDirty: true })
          setImageBroken(false)
        }}
      />
```

- [ ] **Step 2: Проверить сборку и линт**

Run: `npm run build && npm run lint`
Expected: успех.

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
git add src/features/products/ProductEditPage.tsx
git commit -m "feat: pick product image from media gallery"
```

---

### Task 5: Документация и финальная проверка

**Files:**
- Modify: `docs/ux-ui.md` (§3.7 — панель фильтров, строка списка, пикер)
- Modify: `CLAUDE.md` (статус)

- [ ] **Step 1: Обновить docs/ux-ui.md §3.7**

В блоке «Список»: добавить описание панели (поиск по title с deferred-паттерном,
Select Category — полный справочник categories + All, Select Brand — уникальные
бренды из товаров + All, условия по AND), счётчик `видимые / все` при активных
фильтрах, empty state «No products match your filters.», строка списка — название
слева + приглушённое `brand · category` справа.

В блоке «Детальная страница»: Image path — инпут + кнопка Gallery (диалог-пикер:
поиск, Upload одного файла через `uploadImage`, грид миниатюр из `listImages`,
общий кэш `['media', site.slug]`, подсветка текущей картинки; выбор пишет путь в
форму, сохранение — Save).

- [ ] **Step 2: Обновить CLAUDE.md**

В «Сделано» к пункту products v1 добавить v2: поиск + фильтры Category/Brand на
списке (клиентские), `brand · category` в строке, `ImagePickerDialog` (выбор из
галереи + Upload) вместо ручного ввода пути. В «Следующие шаги» убрать пункт про
пикер/поиск (выполнено), оставить categories → posts → pages/hero/footer.

- [ ] **Step 3: Финальная проверка**

Run: `npm run build && npm run lint`
Expected: успех.

Ручная e2e:
1. Список: поиск сужает по названию; фильтры Category/Brand работают и
   комбинируются; счётчик `X / Y`; «No products match your filters.»; сброс на All.
2. Строки показывают `brand · category` справа.
3. Детальная: Gallery → диалог; поиск в диалоге; клик по картинке подставляет путь,
   превью обновляется; Save сохраняет.
4. Upload из диалога: файл появляется в Media и сразу подставлен в товар.

- [ ] **Step 4: Commit (только с разрешения пользователя)**

```bash
git add CLAUDE.md docs/ux-ui.md docs/superpowers/specs/2026-07-10-products-v2-filters-picker-design.md docs/superpowers/plans/2026-07-10-products-v2-filters-picker.md
git commit -m "docs: products v2 spec, plan and status update"
```
