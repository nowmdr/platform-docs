# Category & Brand Unification — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Привести Category и Brand к единой модели-справочнику: таблица `brands`, общий слой данных с каскадом в `products`, generic-UI (Manager + Combobox), интеграция в форму товара и разделы навигации.

**Architecture:** Одна конфигурация `TaxonomyConfig` (category/brand) описывает lib-функции, query-ключ, тексты и флаг `hasImage`. Её потребляют `TaxonomyManager` (полный CRUD-список, паттерн `FoldersPanel`) и `TaxonomyCombobox` (выбор/создание в форме). Rename/delete каскадят в `products` на уровне приложения (FK нет). Мутации — немедленные (как папки), не буферизуются формой товара.

**Tech Stack:** Vite + React 19 + TS, Tailwind v4, shadcn/ui (radix, + `command`/`popover`), TanStack Query, Supabase (проект `zwrkphynupdubevzwdzy`, схема `cozycorner`).

**Спека:** `../specs/2026-07-22-category-brand-unification-design.md`

**Верификация:** тест-раннера нет. Каждая задача — `npm run build` + `npm run lint` в
`web.admin/` + ручной прогон. БД-задачи — через Supabase MCP (`apply_migration`,
`execute_sql`, `list_tables`). **Коммиты — только с разрешения пользователя (AGENTS.md).**
**Порядок обязателен:** сначала БД (Task 1–2), затем слой данных (Task 3), примитивы
(Task 4), config (Task 5), UI (Task 6–7), интеграция (Task 8–10), доки (Task 11).

---

## Task 1: Миграция `brands` (Supabase MCP + локальный файл)

**Files:**
- Create: `cozycorner/supabase/migrations/0023_brands.sql`
- DB: проект `zwrkphynupdubevzwdzy`, схема `cozycorner` (через MCP `apply_migration`)

- [ ] **Step 1: Проверить, что brands ещё нет**

MCP: `mcp__supabase__list_tables` project_id=`zwrkphynupdubevzwdzy`, schemas=`["cozycorner"]`.
Expected: `brands` отсутствует; `products`, `categories` есть.

- [ ] **Step 2: Применить миграцию через MCP**

MCP: `mcp__supabase__apply_migration` project_id=`zwrkphynupdubevzwdzy`, name=`0023_brands`, query:

```sql
create table if not exists cozycorner.brands (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  name text not null unique,
  slug text not null,
  position integer not null default 0
);
create unique index if not exists brands_slug_key on cozycorner.brands (slug);

create or replace function public.brands_set_slug() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.name), ''), 'brand');
  end if;
  return new;
end; $$;
drop trigger if exists brands_set_slug on cozycorner.brands;
create trigger brands_set_slug before insert or update on cozycorner.brands
  for each row execute function public.brands_set_slug();

grant select on cozycorner.brands to anon;
grant all on cozycorner.brands to authenticated, service_role;

alter table cozycorner.brands enable row level security;
drop policy if exists "Public read brands" on cozycorner.brands;
create policy "Public read brands" on cozycorner.brands for select
  to anon, authenticated using (true);
drop policy if exists "Admin write brands" on cozycorner.brands;
create policy "Admin write brands" on cozycorner.brands for all
  to authenticated using (public.is_admin()) with check (public.is_admin());

insert into cozycorner.brands (name)
  select distinct brand from cozycorner.products
  where brand is not null and brand <> ''
on conflict (name) do nothing;
```

Expected: успех, без ошибок.

- [ ] **Step 3: Проверить результат**

MCP: `mcp__supabase__execute_sql` project_id=`zwrkphynupdubevzwdzy`, query:
`select name, slug, position from cozycorner.brands order by position, name;`
Expected: строки-бренды, засеянные из `products.brand` (slug сгенерирован триггером).
Также проверить политики: `select policyname, cmd from pg_policies where schemaname='cozycorner' and tablename='brands';`
Expected: `Public read brands` (SELECT) и `Admin write brands` (ALL).

- [ ] **Step 4: Продублировать миграцию файлом (source of truth)**

Создать `cozycorner/supabase/migrations/0023_brands.sql` с тем же SQL из Step 2, плюс
шапка-комментарий:

```sql
-- Бренды каталога: справочник-сущность (как categories, но без картинки). Товары
-- привязаны по имени (products.brand = brands.name), FK нет сознательно — как у категорий.
-- Применено через Supabase MCP; файл — source of truth (schema.md §7).
```

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd cozycorner && git add supabase/migrations/0023_brands.sql
git commit -m "feat(db): brands table (mirror categories, no image)"
```

---

## Task 2: Синхронизировать типы сайта + schema.md

**Files:**
- Modify: `cozycorner/lib/types.ts` (добавить тип Brand)
- Modify: `platform-docs/database/schema.md` (§5 таблица brands, §6 история миграций)

- [ ] **Step 1: Найти форму типов сайта**

Открыть `cozycorner/lib/types.ts`, найти существующий тип `Category` (образец формы).

- [ ] **Step 2: Добавить тип Brand**

Рядом с `Category` добавить (поля по факту таблицы; без image):

```ts
export type Brand = {
  id: string;
  created_at: string;
  name: string;
  slug: string;
  position: number;
};
```

- [ ] **Step 3: Обновить schema.md**

В `platform-docs/database/schema.md`:
- §5 (`cozycorner`): после «### categories — категории каталога» добавить блок
  «### brands — бренды каталога» с таблицей полей (`id, created_at, name unique,
  slug (автоген триггером `brands_set_slug`), position`), пометкой «FK нет,
  `products.brand = brands.name`».
- §6 (история миграций): дописать `0023 brands`.

- [ ] **Step 4: Верификация**

Run: `cd cozycorner && npm run build` (Next.js — типы должны собраться).
Expected: build без ошибок типов. (Сайт `brands` пока не читает — тип объявлен на будущее.)

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd cozycorner && git add lib/types.ts && git commit -m "types: add Brand"
cd ../platform-docs && git add database/schema.md && git commit -m "docs(db): brands table"
```

---

## Task 3: Слой данных web.admin — categories.ts, brands.ts

**Files:**
- Create: `web.admin/src/lib/categories.ts`
- Create: `web.admin/src/lib/brands.ts`
- Modify: `web.admin/src/lib/products.ts` (удалить `listCategoryNames` — заменяется `listCategories`)

- [ ] **Step 1: Создать lib/categories.ts**

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

export type Category = {
  id: string;
  name: string;
  slug: string;
  image_path: string | null;
  position: number;
};

export async function listCategories(site: SiteConfig): Promise<Category[]> {
  const { data, error } = await getDb(site)
    .from("categories")
    .select("id,name,slug,image_path,position")
    .order("position", { ascending: true })
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Category[];
}

export async function createCategory(site: SiteConfig, name: string): Promise<Category> {
  // slug/position — БД (триггер / default). name unique — конфликт всплывёт ошибкой.
  const { data, error } = await getDb(site)
    .from("categories")
    .insert({ name })
    .select("id,name,slug,image_path,position")
    .single();
  if (error) throw error;
  return data as Category;
}

// Rename каскадит в products (FK нет): сначала строку справочника, затем товары.
export async function renameCategory(
  site: SiteConfig,
  id: string,
  oldName: string,
  newName: string,
): Promise<void> {
  const db = getDb(site);
  const { error } = await db.from("categories").update({ name: newName }).eq("id", id);
  if (error) throw error;
  const { error: pErr } = await db
    .from("products")
    .update({ category: newName })
    .eq("category", oldName);
  if (pErr) throw pErr;
}

// Delete каскадит: строку справочника + обнуляет ссылку у затронутых товаров.
export async function deleteCategory(
  site: SiteConfig,
  id: string,
  name: string,
): Promise<void> {
  const db = getDb(site);
  const { error } = await db.from("categories").delete().eq("id", id);
  if (error) throw error;
  const { error: pErr } = await db
    .from("products")
    .update({ category: null })
    .eq("category", name);
  if (pErr) throw pErr;
}

export async function updateCategoryImage(
  site: SiteConfig,
  id: string,
  imagePath: string | null,
): Promise<void> {
  const { error } = await getDb(site)
    .from("categories")
    .update({ image_path: imagePath })
    .eq("id", id);
  if (error) throw error;
}

export async function reorderCategories(site: SiteConfig, orderedIds: string[]): Promise<void> {
  const db = getDb(site);
  for (let i = 0; i < orderedIds.length; i++) {
    const { error } = await db.from("categories").update({ position: i }).eq("id", orderedIds[i]);
    if (error) throw error;
  }
}
```

- [ ] **Step 2: Создать lib/brands.ts (без image)**

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

export type Brand = {
  id: string;
  name: string;
  slug: string;
  position: number;
};

export async function listBrands(site: SiteConfig): Promise<Brand[]> {
  const { data, error } = await getDb(site)
    .from("brands")
    .select("id,name,slug,position")
    .order("position", { ascending: true })
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Brand[];
}

export async function createBrand(site: SiteConfig, name: string): Promise<Brand> {
  const { data, error } = await getDb(site)
    .from("brands")
    .insert({ name })
    .select("id,name,slug,position")
    .single();
  if (error) throw error;
  return data as Brand;
}

export async function renameBrand(
  site: SiteConfig,
  id: string,
  oldName: string,
  newName: string,
): Promise<void> {
  const db = getDb(site);
  const { error } = await db.from("brands").update({ name: newName }).eq("id", id);
  if (error) throw error;
  const { error: pErr } = await db.from("products").update({ brand: newName }).eq("brand", oldName);
  if (pErr) throw pErr;
}

export async function deleteBrand(site: SiteConfig, id: string, name: string): Promise<void> {
  const db = getDb(site);
  const { error } = await db.from("brands").delete().eq("id", id);
  if (error) throw error;
  const { error: pErr } = await db.from("products").update({ brand: null }).eq("brand", name);
  if (pErr) throw pErr;
}

export async function reorderBrands(site: SiteConfig, orderedIds: string[]): Promise<void> {
  const db = getDb(site);
  for (let i = 0; i < orderedIds.length; i++) {
    const { error } = await db.from("brands").update({ position: i }).eq("id", orderedIds[i]);
    if (error) throw error;
  }
}
```

- [ ] **Step 3: Удалить listCategoryNames из products.ts**

В `src/lib/products.ts` удалить функцию `listCategoryNames` (строки 78–86). Её вызовы
заменит `listCategories` (Task 8, Task 10). Импорт в `ProductEditPage` тоже уберём (Task 8).

- [ ] **Step 4: Верификация**

Run: `cd web.admin && npm run build`
Expected: build упадёт на использовании `listCategoryNames` в `ProductEditPage.tsx` /
`ProductsPage.tsx` (ожидаемо — чиним в Task 8/10). Убедиться, что ОШИБКИ ТОЛЬКО про
`listCategoryNames`, других нет. (Если хочется зелёный build здесь — сделать Task 8/10
до финальной сборки; порядок задач это допускает.)

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd web.admin && git add src/lib/categories.ts src/lib/brands.ts src/lib/products.ts
git commit -m "feat(admin): categories/brands data layer with product cascade"
```

---

## Task 4: Добавить shadcn command + popover

**Files:**
- Create: `web.admin/src/components/ui/command.tsx`, `web.admin/src/components/ui/popover.tsx` (через CLI)

- [ ] **Step 1: Добавить компоненты через CLI**

Run: `cd web.admin && npx shadcn@latest add command popover`
Expected: созданы `src/components/ui/command.tsx` и `src/components/ui/popover.tsx`.
(Если CLI спросит подтверждение перезаписи — не перезаписывать существующие файлы.)

- [ ] **Step 2: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: build чисто; lint — новые ui-файлы могут дать те же известные shadcn-warning
(допустимо). Компоненты пока не используются.

- [ ] **Step 3: Commit (по разрешению)**

```bash
cd web.admin && git add src/components/ui/command.tsx src/components/ui/popover.tsx
git commit -m "chore(admin): add shadcn command + popover"
```

---

## Task 5: Конфигурация таксономии

**Files:**
- Create: `web.admin/src/features/taxonomy/config.ts`

- [ ] **Step 1: Создать config.ts**

```ts
import type { SiteConfig } from "@/config/sites";
import {
  createCategory,
  deleteCategory,
  listCategories,
  renameCategory,
  reorderCategories,
  updateCategoryImage,
  type Category,
} from "@/lib/categories";
import {
  createBrand,
  deleteBrand,
  listBrands,
  renameBrand,
  reorderBrands,
  type Brand,
} from "@/lib/brands";

// Единичный элемент справочника, как его видят Manager и Combobox.
export type TaxonomyItem = {
  id: string;
  name: string;
  position: number;
  image_path: string | null; // у brand всегда null
};

export type TaxonomyConfig = {
  kind: "category" | "brand";
  noun: string; // 'category' | 'brand' — тексты
  title: string; // 'Categories' | 'Brands'
  hasImage: boolean;
  productField: "category" | "brand"; // поле products для подсчёта использования
  queryKey: (site: SiteConfig) => readonly unknown[];
  list: (site: SiteConfig) => Promise<TaxonomyItem[]>;
  create: (site: SiteConfig, name: string) => Promise<TaxonomyItem>;
  rename: (site: SiteConfig, id: string, oldName: string, newName: string) => Promise<void>;
  remove: (site: SiteConfig, id: string, name: string) => Promise<void>;
  reorder: (site: SiteConfig, orderedIds: string[]) => Promise<void>;
  setImage?: (site: SiteConfig, id: string, imagePath: string | null) => Promise<void>;
};

const toItem = (c: Category | Brand): TaxonomyItem => ({
  id: c.id,
  name: c.name,
  position: c.position,
  image_path: "image_path" in c ? c.image_path : null,
});

export const categoryConfig: TaxonomyConfig = {
  kind: "category",
  noun: "category",
  title: "Categories",
  hasImage: true,
  productField: "category",
  queryKey: (site) => ["categories", site.slug],
  list: async (site) => (await listCategories(site)).map(toItem),
  create: async (site, name) => toItem(await createCategory(site, name)),
  rename: renameCategory,
  remove: deleteCategory,
  reorder: reorderCategories,
  setImage: updateCategoryImage,
};

export const brandConfig: TaxonomyConfig = {
  kind: "brand",
  noun: "brand",
  title: "Brands",
  hasImage: false,
  productField: "brand",
  queryKey: (site) => ["brands", site.slug],
  list: async (site) => (await listBrands(site)).map(toItem),
  create: async (site, name) => toItem(await createBrand(site, name)),
  rename: renameBrand,
  remove: deleteBrand,
  reorder: reorderBrands,
};
```

- [ ] **Step 2: Верификация**

Run: `cd web.admin && npm run build`
Expected: `config.ts` компилируется (ошибки про `listCategoryNames` из Task 3 ещё могут
быть, если Task 8/10 не сделаны — это ок).

- [ ] **Step 3: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/taxonomy/config.ts
git commit -m "feat(admin): taxonomy config (category/brand)"
```

---

## Task 6: TaxonomyManager (полный CRUD-список)

**Files:**
- Create: `web.admin/src/features/taxonomy/TaxonomyManager.tsx`

Паттерн — `src/features/folders/FoldersPanel.tsx` (inline create/rename + AlertDialog delete),
плюс reorder (↑/↓) и фото (для `hasImage`). Счётчик использования — из общего кэша
`['products', site.slug]`.

- [ ] **Step 1: Создать TaxonomyManager.tsx**

```tsx
import { useState } from 'react'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { ArrowDown, ArrowUp, Check, Ellipsis, Loader2, Plus, X } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { listProducts } from '@/lib/products'
import { resolveImageUrl, toStoragePath } from '@/lib/images'
import type { TaxonomyConfig, TaxonomyItem } from './config'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import { ImagePreviewPicker } from '@/components/ImagePreviewPicker'
import { ImagePickerDialog } from '@/features/products/ImagePickerDialog'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from '@/components/ui/alert-dialog'

export function TaxonomyManager({ site, config }: { site: SiteConfig; config: TaxonomyConfig }) {
  const queryClient = useQueryClient()
  const key = config.queryKey(site)

  const { data: items, isPending, error } = useQuery({
    queryKey: key,
    queryFn: () => config.list(site),
  })
  // Счётчик «сколько товаров используют» — из общего кэша products (не свой ключ).
  const { data: products } = useQuery({
    queryKey: ['products', site.slug],
    queryFn: () => listProducts(site),
  })
  function usageCount(name: string): number {
    return (products ?? []).filter((p) => p[config.productField] === name).length
  }

  const [creating, setCreating] = useState(false)
  const [draftName, setDraftName] = useState('')
  const [editingId, setEditingId] = useState<string | null>(null)
  const [editName, setEditName] = useState('')
  const [deleteTarget, setDeleteTarget] = useState<TaxonomyItem | null>(null)
  const [imageTarget, setImageTarget] = useState<TaxonomyItem | null>(null)

  function invalidateAll() {
    queryClient.invalidateQueries({ queryKey: key })
    queryClient.invalidateQueries({ queryKey: ['products', site.slug] })
  }

  const create = useMutation({
    mutationFn: (name: string) => config.create(site, name),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: key })
      setCreating(false)
      setDraftName('')
      toast.success(`${cap(config.noun)} created`)
    },
    onError: (e) => toast.error(msg(e, `Failed to create ${config.noun}`)),
  })

  const rename = useMutation({
    mutationFn: ({ item, name }: { item: TaxonomyItem; name: string }) =>
      config.rename(site, item.id, item.name, name),
    onSuccess: () => {
      invalidateAll()
      setEditingId(null)
      toast.success(`${cap(config.noun)} renamed`)
    },
    onError: (e) => toast.error(msg(e, `Failed to rename ${config.noun}`)),
  })

  const remove = useMutation({
    mutationFn: (item: TaxonomyItem) => config.remove(site, item.id, item.name),
    onSuccess: () => {
      invalidateAll()
      setDeleteTarget(null)
      toast.success(`${cap(config.noun)} deleted`)
    },
    onError: (e) => toast.error(msg(e, `Failed to delete ${config.noun}`)),
  })

  const reorder = useMutation({
    mutationFn: (orderedIds: string[]) => config.reorder(site, orderedIds),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: key }),
    onError: (e) => toast.error(msg(e, `Failed to reorder ${config.noun}s`)),
  })

  const setImage = useMutation({
    mutationFn: ({ id, path }: { id: string; path: string | null }) =>
      config.setImage!(site, id, path),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: key })
      toast.success('Image updated')
    },
    onError: (e) => toast.error(msg(e, 'Failed to update image')),
  })

  function move(index: number, dir: -1 | 1) {
    if (!items) return
    const next = [...items]
    const j = index + dir
    if (j < 0 || j >= next.length) return
    ;[next[index], next[j]] = [next[j], next[index]]
    reorder.mutate(next.map((i) => i.id))
  }

  function submitCreate() {
    const name = draftName.trim()
    if (!name) return setCreating(false), setDraftName('')
    create.mutate(name)
  }
  function submitRename(item: TaxonomyItem) {
    const name = editName.trim()
    if (!name || name === item.name) return setEditingId(null)
    rename.mutate({ item, name })
  }

  if (error) {
    return (
      <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
        Failed to load {config.noun}s: {error.message}
      </p>
    )
  }

  return (
    <div className="flex max-w-2xl flex-col gap-2">
      {isPending &&
        Array.from({ length: 6 }, (_, i) => <Skeleton key={i} className="h-12 w-full rounded-md" />)}

      {items && items.length === 0 && !creating && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          No {config.noun}s yet.
        </p>
      )}

      {items?.map((item, index) => (
        <div key={item.id} className="group flex items-center gap-2 rounded-md border p-2">
          {config.hasImage && (
            <button
              type="button"
              onClick={() => setImageTarget(item)}
              className="size-10 shrink-0 overflow-hidden rounded border bg-muted/30"
              aria-label={`Change image for ${item.name}`}
            >
              {item.image_path ? (
                <img
                  src={resolveImageUrl(site, item.image_path)}
                  alt=""
                  className="size-full object-contain"
                />
              ) : (
                <span className="flex size-full items-center justify-center text-xs text-muted-foreground">
                  +
                </span>
              )}
            </button>
          )}

          {editingId === item.id ? (
            <div className="flex min-w-0 flex-1 items-center gap-1">
              <Input
                autoFocus
                value={editName}
                onChange={(e) => setEditName(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === 'Enter') submitRename(item)
                  if (e.key === 'Escape') setEditingId(null)
                }}
                disabled={rename.isPending}
                className="h-8 text-sm"
              />
              <Button variant="ghost" size="icon-sm" aria-label="Save name"
                onClick={() => submitRename(item)} disabled={rename.isPending}>
                {rename.isPending ? <Loader2 className="animate-spin" /> : <Check />}
              </Button>
              <Button variant="ghost" size="icon-sm" aria-label="Cancel renaming"
                onClick={() => setEditingId(null)} disabled={rename.isPending}>
                <X />
              </Button>
            </div>
          ) : (
            <>
              <span className="min-w-0 flex-1 truncate text-sm font-medium" title={item.name}>
                {item.name}
              </span>
              <span className="shrink-0 text-xs text-muted-foreground">
                {usageCount(item.name)} product{usageCount(item.name) === 1 ? '' : 's'}
              </span>
              <div className="flex shrink-0 items-center">
                <Button variant="ghost" size="icon-sm" aria-label="Move up"
                  disabled={index === 0 || reorder.isPending} onClick={() => move(index, -1)}>
                  <ArrowUp className="size-3.5" />
                </Button>
                <Button variant="ghost" size="icon-sm" aria-label="Move down"
                  disabled={index === (items?.length ?? 0) - 1 || reorder.isPending}
                  onClick={() => move(index, 1)}>
                  <ArrowDown className="size-3.5" />
                </Button>
                <DropdownMenu>
                  <DropdownMenuTrigger asChild>
                    <Button variant="ghost" size="icon-sm" aria-label={`Actions for ${item.name}`}>
                      <Ellipsis className="size-3.5" />
                    </Button>
                  </DropdownMenuTrigger>
                  <DropdownMenuContent align="end">
                    <DropdownMenuItem onClick={() => {
                      setEditName(item.name)
                      setEditingId(item.id)
                    }}>
                      Rename
                    </DropdownMenuItem>
                    <DropdownMenuItem variant="destructive" onClick={() => setDeleteTarget(item)}>
                      Delete
                    </DropdownMenuItem>
                  </DropdownMenuContent>
                </DropdownMenu>
              </div>
            </>
          )}
        </div>
      ))}

      {creating ? (
        <div className="flex items-center gap-1">
          <Input
            autoFocus
            placeholder={`${cap(config.noun)} name`}
            value={draftName}
            onChange={(e) => setDraftName(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter') submitCreate()
              if (e.key === 'Escape') { setCreating(false); setDraftName('') }
            }}
            disabled={create.isPending}
            className="h-9 text-sm"
          />
          <Button variant="ghost" size="icon-sm" aria-label={`Create ${config.noun}`}
            onClick={submitCreate} disabled={create.isPending}>
            {create.isPending ? <Loader2 className="animate-spin" /> : <Check />}
          </Button>
          <Button variant="ghost" size="icon-sm" aria-label="Cancel"
            onClick={() => { setCreating(false); setDraftName('') }} disabled={create.isPending}>
            <X />
          </Button>
        </div>
      ) : (
        <Button variant="outline" size="sm" className="w-fit" onClick={() => setCreating(true)}>
          <Plus className="size-4" />
          New {config.noun}
        </Button>
      )}

      <AlertDialog open={deleteTarget !== null}
        onOpenChange={(open) => { if (!open) setDeleteTarget(null) }}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete {config.noun} "{deleteTarget?.name}"?</AlertDialogTitle>
            <AlertDialogDescription>
              {deleteTarget && usageCount(deleteTarget.name) > 0
                ? `${usageCount(deleteTarget.name)} product(s) use this ${config.noun} and will lose it. `
                : `No products use this ${config.noun}. `}
              This cannot be undone.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction variant="destructive"
              onClick={() => { if (deleteTarget) remove.mutate(deleteTarget) }}>
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>

      {config.hasImage && imageTarget && (
        <ImagePickerDialog
          site={site}
          open={imageTarget !== null}
          onOpenChange={(open) => { if (!open) setImageTarget(null) }}
          currentPath={imageTarget.image_path}
          onSelect={(path) => {
            setImage.mutate({ id: imageTarget.id, path: toStoragePath(site, path) })
            setImageTarget(null)
          }}
        />
      )}
    </div>
  )
}

const cap = (s: string) => s.charAt(0).toUpperCase() + s.slice(1)
const msg = (e: unknown, fallback: string) => (e instanceof Error ? e.message : fallback)
```

- [ ] **Step 2: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: компилируется (ошибки про `listCategoryNames` из Task 3 всё ещё возможны до
Task 8/10). `ImagePickerDialog.onSelect` отдаёт `path` (media.path); мы конвертируем через
`toStoragePath` — но `path` из пикера уже плоский ключ, `toStoragePath` внешние/плоские
пропускает как есть (свериться с `lib/images.ts` — если `path` уже ключ, `setImage`
можно звать прямо с `path`). Уточнить при реализации по факту сигнатуры.

- [ ] **Step 3: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/taxonomy/TaxonomyManager.tsx
git commit -m "feat(admin): TaxonomyManager (CRUD + reorder + image)"
```

---

## Task 7: TaxonomyCombobox (выбор/создание в форме)

**Files:**
- Create: `web.admin/src/features/taxonomy/TaxonomyCombobox.tsx`

Combobox на `Popover` + `Command`: выбор существующего, «Create "<query>"» при отсутствии
совпадения, пункт None, кнопка Manage… (открывает `TaxonomyManager` в `Dialog`).
Значение — строка-имя (или '' = None).

- [ ] **Step 1: Создать TaxonomyCombobox.tsx**

```tsx
import { useState } from 'react'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { Check, ChevronsUpDown, Plus, Settings2 } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { cn } from '@/lib/utils'
import type { TaxonomyConfig } from './config'
import { TaxonomyManager } from './TaxonomyManager'
import { Button } from '@/components/ui/button'
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from '@/components/ui/command'
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover'
import { Dialog, DialogBody, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'

const NONE_LABEL = 'None'

export function TaxonomyCombobox({
  site,
  config,
  value,
  onChange,
}: {
  site: SiteConfig
  config: TaxonomyConfig
  value: string // '' = None
  onChange: (name: string) => void
}) {
  const queryClient = useQueryClient()
  const key = config.queryKey(site)
  const [open, setOpen] = useState(false)
  const [query, setQuery] = useState('')
  const [manageOpen, setManageOpen] = useState(false)

  const { data: items } = useQuery({ queryKey: key, queryFn: () => config.list(site) })

  const create = useMutation({
    mutationFn: (name: string) => config.create(site, name),
    onSuccess: (item) => {
      queryClient.invalidateQueries({ queryKey: key })
      onChange(item.name)
      setOpen(false)
      setQuery('')
      toast.success(`${config.noun.charAt(0).toUpperCase() + config.noun.slice(1)} created`)
    },
    onError: (e) =>
      toast.error(e instanceof Error ? e.message : `Failed to create ${config.noun}`),
  })

  const trimmed = query.trim()
  const exists = (items ?? []).some((i) => i.name.toLowerCase() === trimmed.toLowerCase())

  return (
    <>
      <Popover open={open} onOpenChange={setOpen}>
        <PopoverTrigger asChild>
          <Button
            type="button"
            variant="outline"
            role="combobox"
            aria-expanded={open}
            className="w-full justify-between font-normal"
          >
            <span className={cn('truncate', !value && 'text-muted-foreground')}>
              {value || NONE_LABEL}
            </span>
            <ChevronsUpDown className="size-4 shrink-0 opacity-50" />
          </Button>
        </PopoverTrigger>
        <PopoverContent className="w-[var(--radix-popover-trigger-width)] p-0" align="start">
          <Command>
            <CommandInput
              placeholder={`Search ${config.noun}…`}
              value={query}
              onValueChange={setQuery}
            />
            <CommandList>
              <CommandItem
                value="__none__"
                onSelect={() => { onChange(''); setOpen(false) }}
              >
                <Check className={cn('size-4', value ? 'opacity-0' : 'opacity-100')} />
                {NONE_LABEL}
              </CommandItem>
              <CommandGroup>
                {(items ?? []).map((item) => (
                  <CommandItem
                    key={item.id}
                    value={item.name}
                    onSelect={() => { onChange(item.name); setOpen(false) }}
                  >
                    <Check
                      className={cn('size-4', value === item.name ? 'opacity-100' : 'opacity-0')}
                    />
                    <span className="truncate">{item.name}</span>
                  </CommandItem>
                ))}
              </CommandGroup>
              {trimmed && !exists && (
                <CommandItem
                  value={`__create__${trimmed}`}
                  onSelect={() => create.mutate(trimmed)}
                  disabled={create.isPending}
                >
                  <Plus className="size-4" />
                  Create "{trimmed}"
                </CommandItem>
              )}
              {!trimmed && (items?.length ?? 0) === 0 && (
                <CommandEmpty>No {config.noun}s yet. Type to create one.</CommandEmpty>
              )}
              <CommandItem
                value="__manage__"
                onSelect={() => { setOpen(false); setManageOpen(true) }}
                className="text-muted-foreground"
              >
                <Settings2 className="size-4" />
                Manage {config.noun}s…
              </CommandItem>
            </CommandList>
          </Command>
        </PopoverContent>
      </Popover>

      <Dialog open={manageOpen} onOpenChange={setManageOpen}>
        <DialogContent className="sm:max-w-2xl">
          <DialogHeader>
            <DialogTitle>Manage {config.title.toLowerCase()}</DialogTitle>
          </DialogHeader>
          <DialogBody>
            <TaxonomyManager site={site} config={config} />
          </DialogBody>
        </DialogContent>
      </Dialog>
    </>
  )
}
```

- [ ] **Step 2: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: компилируется. `--radix-popover-trigger-width` — стандартная radix-переменная
(проверить, что shadcn popover её выставляет; если имя иное — взять из сгенерированного
`popover.tsx`). Ручная проверка отложена до Task 8 (combobox в форме).

- [ ] **Step 3: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/taxonomy/TaxonomyCombobox.tsx
git commit -m "feat(admin): TaxonomyCombobox (select/create + manage)"
```

---

## Task 8: Интеграция combobox в форму товара

**Files:**
- Modify: `web.admin/src/features/products/ProductEditPage.tsx`

- [ ] **Step 1: Обновить импорты**

Убрать импорт `listCategoryNames` из `@/lib/products` (строка ~21). Добавить:

```tsx
import { TaxonomyCombobox } from '@/features/taxonomy/TaxonomyCombobox'
import { categoryConfig, brandConfig } from '@/features/taxonomy/config'
```

- [ ] **Step 2: Убрать NONE-сентинел и загрузку категорий**

- Удалить `const NONE = '__none__'` (строка 52).
- В `ProductEditPage` удалить `useQuery` категорий (строки ~129–132) и переменные
  `categoriesError`; `error`/`loading`/`showForm` пересчитать без категорий:

```tsx
  const error = productError
  const loading = !isNew && productPending
  const showForm = !error && !loading && (isNew || product)
```

- Убрать проп `categories` из `<ProductForm .../>` (строки ~164–170) и из сигнатуры
  `ProductForm` (строки ~176–184) — форма справочники больше не принимает.

- [ ] **Step 3: category '' = None (унифицировать с brand)**

- В `toFormValues` (строка ~108): `category: product?.category ?? ''` (было `?? NONE`).
- В `toInput` (строка ~85): `category: orNull(values.category)` (было `values.category === NONE ? null : values.category`).

- [ ] **Step 4: Заменить поле Brand на combobox**

Заменить блок `<Field data-invalid={!!errors.brand}>` с `<Input id="brand" …/>` (строки ~379–382):

```tsx
              <Field>
                <FieldLabel htmlFor="brand">Brand</FieldLabel>
                <Controller
                  control={form.control}
                  name="brand"
                  render={({ field }) => (
                    <TaxonomyCombobox
                      site={site}
                      config={brandConfig}
                      value={field.value}
                      onChange={(name) => field.onChange(name)}
                    />
                  )}
                />
              </Field>
```

- [ ] **Step 5: Заменить поле Category на combobox**

Заменить блок `<Field>` с `<Controller name="category" … Select …/>` (строки ~420–441):

```tsx
            <Field>
              <FieldLabel htmlFor="category">Category</FieldLabel>
              <Controller
                control={form.control}
                name="category"
                render={({ field }) => (
                  <TaxonomyCombobox
                    site={site}
                    config={categoryConfig}
                    value={field.value}
                    onChange={(name) => field.onChange(name)}
                  />
                )}
              />
            </Field>
```

- [ ] **Step 6: Убрать неиспользуемый импорт Select (если больше не нужен)**

Проверить, используется ли `Select*` в файле после замены (image_style ещё использует
Select — тогда импорт оставить). Убрать только реально неиспользуемые импорты, чтобы
lint не ругался.

- [ ] **Step 7: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто (ошибки про `listCategoryNames` в этом файле исчезают). Ручная проверка
(`npm run dev` → Products → New product): Brand и Category — combobox; выбор существующего
работает; ввод нового имени → «Create "X"» создаёт и подставляет; None очищает; Manage…
открывает менеджер в модалке; Save пишет имя (или null). Проверить, что созданный
combobox'ом бренд/категория сразу появляется в списке (инвалидация ключа).

- [ ] **Step 8: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/products/ProductEditPage.tsx
git commit -m "feat(admin): unified brand/category combobox in product form"
```

---

## Task 9: Разделы Categories и Brands в навигации

**Files:**
- Create: `web.admin/src/features/taxonomy/CategoriesPage.tsx`, `web.admin/src/features/taxonomy/BrandsPage.tsx`
- Modify: `web.admin/src/components/SiteLayout.tsx`, `web.admin/src/App.tsx`

- [ ] **Step 1: Создать страницы-обёртки**

`src/features/taxonomy/CategoriesPage.tsx`:

```tsx
import { useOutletContext } from 'react-router-dom'
import type { SiteConfig } from '@/config/sites'
import { categoryConfig } from './config'
import { TaxonomyManager } from './TaxonomyManager'

export function CategoriesPage() {
  const site = useOutletContext<SiteConfig>()
  return (
    <div className="flex flex-col gap-4">
      <h1 className="font-heading text-lg font-medium">Categories</h1>
      <TaxonomyManager site={site} config={categoryConfig} />
    </div>
  )
}
```

`src/features/taxonomy/BrandsPage.tsx` — то же с `brandConfig` и заголовком «Brands».

- [ ] **Step 2: Добавить пункты навигации**

В `src/components/SiteLayout.tsx`: импорт иконок (строка 2) дополнить `Tags, Bookmark`:

```tsx
import { Image, FileText, Package, Newspaper, Search, Tags, Bookmark } from 'lucide-react'
```

В `NAV_ITEMS` (строки 6–12) добавить после `products`:

```tsx
  { to: 'products', label: 'Products', icon: Package },
  { to: 'categories', label: 'Categories', icon: Tags },
  { to: 'brands', label: 'Brands', icon: Bookmark },
```

(Если `Tags`/`Bookmark` не экспортируются в lucide-react@1.x — взять существующие,
напр. `Tag` и `Award`; проверить импортом.)

- [ ] **Step 3: Добавить роуты**

В `src/App.tsx` импорт:

```tsx
import { CategoriesPage } from '@/features/taxonomy/CategoriesPage'
import { BrandsPage } from '@/features/taxonomy/BrandsPage'
```

В блок роутов сайта (после `products/:productId`, строка ~40) добавить:

```tsx
            <Route path="categories" element={<CategoriesPage />} />
            <Route path="brands" element={<BrandsPage />} />
```

- [ ] **Step 4: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто. Ручная проверка (`npm run dev`): в левом меню появились Categories и
Brands; переход открывает менеджер; create/rename/delete/reorder работают; у Categories
есть смена фото, у Brands — нет; после rename связанные товары в списке Products
показывают новое имя.

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/taxonomy/CategoriesPage.tsx src/features/taxonomy/BrandsPage.tsx src/components/SiteLayout.tsx src/App.tsx
git commit -m "feat(admin): Categories & Brands nav sections"
```

---

## Task 10: Фильтр Brand в списке товаров — из справочника

**Files:**
- Modify: `web.admin/src/features/products/ProductsPage.tsx`

- [ ] **Step 1: Прочитать текущие фильтры**

Открыть `ProductsPage.tsx`, найти, как строится список брендов для фильтра (сейчас —
уникальные из загруженных товаров) и категорий (сейчас `listCategoryNames`).

- [ ] **Step 2: Источник — справочники**

- Заменить загрузку категорий с `listCategoryNames` на `listCategories` (map `.name`),
  ключ `['categories', site.slug]` (тот же, что combobox — общий кэш).
- Бренды для фильтра — из `listBrands` (map `.name`), ключ `['brands', site.slug]`.

Конкретные правки зависят от текущего кода файла — сохранить существующую семантику
фильтров (AND, Clear, счётчик), поменять только ИСТОЧНИК списков на справочники.

- [ ] **Step 3: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто. Ручная проверка: фильтр Brand показывает ВСЕ бренды справочника (даже
не использованные ни одним товаром); Category — как раньше; фильтрация работает.

- [ ] **Step 4: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/products/ProductsPage.tsx
git commit -m "feat(admin): product list brand filter from brands table"
```

---

## Task 11: Доки + финальная верификация

**Files:**
- Modify: `platform-docs/admin-panel/products.md`, `platform-docs/admin-panel/components.md`, `platform-docs/admin-panel/status.md`

- [ ] **Step 1: Обновить products.md**

- §1: `category`/`brand` — оба привязаны к справочникам по имени; data-слой
  `lib/categories.ts` + `lib/brands.ts` (list/create/rename/delete/reorder, у категории
  updateImage), rename/delete каскадят в products (app-level, не атомарно — известное
  упрощение). Убрать упоминание `listCategoryNames`.
- §2: фильтр Brand — из справочника `brands` (полный список).
- §3: Brand и Category в форме — `TaxonomyCombobox` (выбор/создание + Manage).

- [ ] **Step 2: Обновить components.md**

- §2 (навигация): добавить Categories и Brands в карту меню.
- §3.7 (форма товара): Brand/Category — combobox вместо Input/Select.
- Добавить §3.11 «Categories / Brands» — generic `TaxonomyManager` (CRUD + reorder,
  фото у категорий), паттерн inline-edit как у папок.
- §6: Categories/Brands больше не «будущие» — реализованы.

- [ ] **Step 3: Обновить status.md**

Дописать: brands-таблица (миграция 0023), унификация category/brand, известные хвосты
(каскад не атомарный; hero/SEO категорий не в админке).

- [ ] **Step 4: Полная верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: build без ошибок; lint ≤ 2 известных shadcn-warning (могут добавиться warning
от новых ui-компонентов command/popover — допустимо, зафиксировать сколько).

- [ ] **Step 5: Сквозной ручной прогон**

`npm run dev`: (1) создать бренд и категорию из раздела; (2) из combobox в форме товара;
(3) переименовать — товары показывают новое имя; (4) удалить — товары теряют ссылку;
(5) сменить фото категории; (6) фильтры списка товаров видят полный справочник брендов.

- [ ] **Step 6: get_advisors (безопасность/производительность БД)**

MCP: `mcp__supabase__get_advisors` project_id=`zwrkphynupdubevzwdzy`, type=`security`, затем `performance`.
Expected: нет новых предупреждений про `brands` (RLS включён, политики есть). Если есть —
разобрать.

- [ ] **Step 7: Commit доков (по разрешению)**

```bash
cd platform-docs && git add admin-panel/products.md admin-panel/components.md admin-panel/status.md
git commit -m "docs(admin): category/brand unification"
```

---

## Self-Review (spec coverage)

- Спека §1 (миграция brands, RLS, схема, seed) → Task 1. ✓
- Спека §1 (типы сайта, schema.md) → Task 2. ✓
- Спека §2 (lib categories/brands, каскад, query-ключи) → Task 3. ✓
- Спека §3 (generic-абстракция) → Task 5 (config) + Task 6 (Manager) + Task 7 (Combobox). ✓
- Спека §3 (command/popover через CLI) → Task 4. ✓
- Спека §4 (интеграция в форму товара) → Task 8. ✓
- Спека §5 (разделы навигации) → Task 9. ✓
- Спека §6 (фильтр Brand из справочника) → Task 10. ✓
- Спека «затрагиваемые доки» → Task 2 + Task 11. ✓
- Дефолты: delete обнуляет ссылку у товаров (Task 3 remove) ✓; hasImage только category
  (config + Manager) ✓; hero/SEO вне scope (не запланировано — верно).

## Type consistency

- `TaxonomyItem { id, name, position, image_path }` — определён в Task 5, используется в
  Task 6/7 идентично.
- `TaxonomyConfig` методы (`list/create/rename/remove/reorder/setImage`) — сигнатуры
  совпадают между config.ts (Task 5) и вызовами в Manager (Task 6) / Combobox (Task 7).
- Значение поля формы — строка-имя, '' = None; согласовано между Task 3 (`orNull`),
  Task 7 (combobox value) и Task 8 (`toInput`/`toFormValues`).

## Открытые уточнения при реализации (не блокеры)

- Точная сигнатура `ImagePickerDialog.onSelect` vs `updateCategoryImage` (плоский ключ /
  публичная ссылка) — свериться с `lib/images.ts` в Task 6.
- Имена lucide-иконок в v1.x (`Tags`/`Bookmark`) — проверить экспорт в Task 9.
- `--radix-popover-trigger-width` — взять фактическое имя из сгенерированного popover.tsx (Task 7).
- Точный код фильтров `ProductsPage` (Task 10) — по факту текущего файла.
