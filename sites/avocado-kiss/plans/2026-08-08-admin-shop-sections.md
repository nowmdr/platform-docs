# Avocado Shop — разделы админки (Products / Categories / Brands) — план (План 3)

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans. Шаги — чекбоксы.
> Разрешение на коммит/пуш в сессии дано. Предпосылки: Планы 1–2 выполнены (M2M/brands/folders/триггеры в БД; сайт на M2M).

**Goal:** Включить для сайта avocado-kiss разделы админки Products / Categories (=shop_categories) / Brands, переиспользуя generic-код cozy, **не зарегрессив cozycorner**.

**Architecture:** Роуты в `App.tsx` уже работают для любого `:siteSlug` — включение = добавить секции в `sites.ts` allowlist + сделать site-aware только то, что отличается по схеме. Отличия: (1) таблица категорий `categories` (cozy) vs `shop_categories` (avocado) с разным набором колонок; (2) колонка `products.image_style` только у cozy. Приём: `categories.ts` алиасит картинку категории к каноничному `image_path` (`image_path:hero_image_path` для avocado) и добавляет `hero_eyebrow`; `product_categories`/`brands`/M2M-код уже портируемы. Ветка `cozycorner` во всех site-aware местах = текущее поведение (регресс-гард — существующие тесты web.admin + build/lint).

**Tech Stack:** Vite+React SPA, TanStack Query, supabase-js `.schema(site.schema)`, Vitest+RTL.

---

## Что НЕ трогаем (уже портируемо, подтверждено картой)
`src/lib/productCategories.ts`, `src/lib/brands.ts`, `src/features/taxonomy/{TaxonomyManager,TaxonomyCombobox,config}.tsx/ts`, `src/features/products/{CategoryChipsField,ImagePickerDialog,ProductListRow}.tsx`, `App.tsx` (роуты), `ProductsPage.tsx` (фильтры). Они используют только общие таблицы (`product_categories`, `brands`, `products.brand`) и `id/name`, либо чистая презентация.

---

## Task 1: Brands + Products в allowlist + Categories (пока не готова — добавим в Task 4)

**Files:** `src/config/sites.ts`

Промежуточно включаем только то, что заработает без дальнейших правок. Brands — сразу
(таблица идентична, seeded 20). Products и Categories добавим после Task 2–4, единым
финальным включением, чтобы не показать полурабочий раздел. → Этот Task делаем
**последним**; ставим сюда финальный allowlist `["media","products","categories","brands"]`.

_(Порядок исполнения: Task 2 → 3 → 4 → 1 → 5.)_

---

## Task 2: `products.image_style` — сделать cozycorner-only

**Files:** `src/lib/products.ts:15`, `src/features/products/ProductEditPage.tsx` (toInput)

- [ ] **Step 1:** В `src/lib/products.ts` строка 15 — сделать поле опциональным:
```ts
  image_style?: "photo" | "cutout"; // cozycorner-only колонка; у др. сайтов её нет
```
(Так `ProductInput = Omit<Product,"id"|"created_at"|"slug">` тоже делает `image_style` опциональным.)

- [ ] **Step 2:** В `ProductEditPage.tsx` `toInput` — включать `image_style` только для cozycorner. Заменить строку `image_style: values.image_style,` на условие: собрать базовый объект без image_style, добавить его только для `site.schema === "cozycorner"`. Точный приём — вернуть `{ ...base, ...(site.schema === "cozycorner" ? { image_style: values.image_style } : {}) }`. `toFormValues`/Zod-схему **не трогаем** (форма всегда несёт `image_style='cutout'`, но для не-cozy он не уходит в payload).

- [ ] **Step 3:** Прогон тестов раздела products (регресс cozy):
Run: `npm --prefix web.admin test -- products`  Expected: PASS (fixture cozycorner → image_style по-прежнему в payload).

- [ ] **Step 4: Commit** `web.admin` (см. общий коммит в Task 5, либо отдельный).

---

## Task 3: `categories.ts` — site-aware таблица/колонки (canonical image_path)

**Files:** `src/lib/categories.ts` (переписать)

- [ ] **Step 1:** Ввести модель по схеме и переписать функции. Ключевое: SELECT алиасит
картинку к `image_path` (`image_path:<real>`), WRITE мапит `image_path` → реальную колонку;
avocado добавляет `hero_eyebrow`. Полный новый файл:

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

// Модель таксономии категорий по схеме сайта. cozy: таблица categories, картинка
// в image_path. avocado: shop_categories, картинка в hero_image_path + hero_eyebrow.
type CategoryModel = { table: string; imageColumn: string; hasEyebrow: boolean };
const CATEGORY_MODELS: Record<string, CategoryModel> = {
  cozycorner: { table: "categories", imageColumn: "image_path", hasEyebrow: false },
  avocado_kiss: { table: "shop_categories", imageColumn: "hero_image_path", hasEyebrow: true },
};
function model(site: SiteConfig): CategoryModel {
  return CATEGORY_MODELS[site.schema] ?? CATEGORY_MODELS.cozycorner;
}

export type Category = {
  id: string;
  name: string;
  slug: string;
  image_path: string | null; // каноничное имя; для avocado = hero_image_path (alias)
  position: number;
};

export type CategoryFull = Category & {
  hero_eyebrow: string | null; // только avocado (cozy → null/не выводится)
  hero_title: string | null;
  hero_description: string | null;
  seo_title: string | null;
  seo_description: string | null;
};

export type CategoryInput = {
  name: string;
  image_path: string | null;
  hero_eyebrow?: string | null;
  hero_title: string | null;
  hero_description: string | null;
  seo_title: string | null;
  seo_description: string | null;
};

// Каноничный image_path → реальная колонка + hero_eyebrow только где есть.
function toDbPayload(m: CategoryModel, input: CategoryInput): Record<string, unknown> {
  const p: Record<string, unknown> = {
    name: input.name,
    [m.imageColumn]: input.image_path,
    hero_title: input.hero_title,
    hero_description: input.hero_description,
    seo_title: input.seo_title,
    seo_description: input.seo_description,
  };
  if (m.hasEyebrow) p.hero_eyebrow = input.hero_eyebrow ?? null;
  return p;
}

// Есть ли у категорий этого сайта поле hero_eyebrow (для редактора).
export function categoryHasEyebrow(site: SiteConfig): boolean {
  return model(site).hasEyebrow;
}

export async function listCategories(site: SiteConfig): Promise<Category[]> {
  const m = model(site);
  const { data, error } = await getDb(site)
    .from(m.table)
    .select(`id,name,slug,position,image_path:${m.imageColumn}`)
    .order("position", { ascending: true })
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Category[];
}

export async function getCategory(site: SiteConfig, id: string): Promise<CategoryFull | null> {
  const m = model(site);
  const cols = `id,name,slug,position,image_path:${m.imageColumn},hero_title,hero_description,seo_title,seo_description${m.hasEyebrow ? ",hero_eyebrow" : ""}`;
  const { data, error } = await getDb(site).from(m.table).select(cols).eq("id", id).maybeSingle();
  if (error) throw error;
  if (!data) return null;
  return { hero_eyebrow: null, ...(data as object) } as CategoryFull;
}

export async function createCategory(site: SiteConfig, name: string): Promise<Category> {
  const m = model(site);
  const { data, error } = await getDb(site)
    .from(m.table)
    .insert({ name })
    .select(`id,name,slug,position,image_path:${m.imageColumn}`)
    .single();
  if (error) throw error;
  return data as Category;
}

export async function createCategoryFull(site: SiteConfig, input: CategoryInput): Promise<{ id: string }> {
  const m = model(site);
  const { data, error } = await getDb(site)
    .from(m.table)
    .insert(toDbPayload(m, input))
    .select("id")
    .single();
  if (error) throw error;
  return data as { id: string };
}

export async function updateCategory(
  site: SiteConfig,
  id: string,
  _oldName: string,
  input: CategoryInput,
): Promise<void> {
  const m = model(site);
  const { error } = await getDb(site).from(m.table).update(toDbPayload(m, input)).eq("id", id);
  if (error) throw error;
}

export async function renameCategory(
  site: SiteConfig,
  id: string,
  _oldName: string,
  newName: string,
): Promise<void> {
  const m = model(site);
  const { error } = await getDb(site).from(m.table).update({ name: newName }).eq("id", id);
  if (error) throw error;
}

export async function deleteCategory(site: SiteConfig, id: string, _name: string): Promise<void> {
  const m = model(site);
  const { error } = await getDb(site).from(m.table).delete().eq("id", id);
  if (error) throw error;
}

export async function reorderCategories(site: SiteConfig, orderedIds: string[]): Promise<void> {
  const m = model(site);
  const db = getDb(site);
  const results = await Promise.all(
    orderedIds.map((id, i) => db.from(m.table).update({ position: i }).eq("id", id)),
  );
  const failed = results.find((r) => r.error);
  if (failed?.error) throw failed.error;
}
```

- [ ] **Step 2:** Typecheck: `npm --prefix web.admin run build` — TS ок.

---

## Task 4: `CategoryEditDialog` — поле hero_eyebrow (только где есть)

**Files:** `src/features/taxonomy/CategoryEditDialog.tsx`

- [ ] **Step 1:** Импортировать `categoryHasEyebrow` из `@/lib/categories`; в Zod-схему добавить `hero_eyebrow: z.string()`; в `toFormValues` — `hero_eyebrow: c?.hero_eyebrow ?? ''`; в `toInput` — `hero_eyebrow: orNull(values.hero_eyebrow)`.
- [ ] **Step 2:** В `CategoryForm` вычислить `const hasEyebrow = categoryHasEyebrow(site)` и отрендерить поле **над** Hero title только при `hasEyebrow`:
```tsx
{hasEyebrow && (
  <Field>
    <FieldLabel htmlFor="cat-hero-eyebrow">Hero eyebrow</FieldLabel>
    <Input id="cat-hero-eyebrow" {...form.register('hero_eyebrow')} />
  </Field>
)}
```
(Для cozy поле не рендерится; `toDbPayload` для cozy `hero_eyebrow` не отправляет — колонки нет.)
- [ ] **Step 3:** Прогон тестов taxonomy: `npm --prefix web.admin test -- taxonomy` — PASS (cozy: hasEyebrow=false, поведение прежнее).

---

## Task 1 (финал): включить секции avocado

**Files:** `src/config/sites.ts`
- [ ] Заменить `sections: ["media"]` → `sections: ["media", "products", "categories", "brands"]`.

---

## Task 5: Проверка + документация + коммит

- [ ] **Unit (регресс cozy + новое):** `npm --prefix web.admin test` — всё зелёное.
- [ ] **Build + lint:** `npm --prefix web.admin run build && npm --prefix web.admin run lint` — ок (2 известных shadcn-warning допустимы).
- [ ] **Живой смоук (MCP, как admin-запрос под avocado_kiss):** `select id,name,slug,position,hero_image_path,hero_eyebrow from avocado_kiss.shop_categories order by position;` — 6 строк; убедиться, что alias image_path соответствует hero_image_path.
- [ ] **Ручной чек (по возможности):** залогиниться в админку → Avocado Kiss → Products (список 48, фильтр Brand/Category), Categories (6, edit c hero_eyebrow), Brands (20). Если dev-сервер занят — отметить как ручной шаг для пользователя.
- [ ] **Доки:** `platform-docs/admin-panel/status.md` (+ строка о Shop-разделах avocado), при необходимости `products.md`/`media.md` не трогаем. Обновить `sites/avocado-kiss.md` §7 (админ-фаза B: Media + Shop готовы).
- [ ] **Commit + push** web.admin и platform-docs.

---

## Готово, когда
- Cozy-тесты web.admin зелёные (регресс-гард), build+lint ок.
- В админке у Avocado Kiss видны Products/Categories/Brands и работают против `avocado_kiss`.
- Cozy-разделы не изменились по поведению.

## Вне объёма
- Переименование nav-лейбла «Categories»→«Shop Categories» для avocado (когда добавим recipe-категории — тогда развести). Пока slug `categories` = shop-категории avocado.
- Recipes/Home/Pages/Footer разделы avocado — следующие фазы.
