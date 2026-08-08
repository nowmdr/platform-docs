# Avocado Shop — переключение сайта на M2M-категории — план (План 2)

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development или superpowers:executing-plans. Шаги — чекбоксы (`- [ ]`).
>
> ⚠️ Не коммитить/пушить без разрешения (в текущей сессии разрешение дано).
> Предпосылка: **План 1 применён** (`product_categories`, `brands`, триггеры существуют в base-one).

**Goal:** Перевести чтение связи «товар↔категория» на сайте avocado.kiss с текстового `products.category` на M2M `product_categories`, сохранив весь UI и контракт типов.

**Architecture:** Меняется только `avocado.kiss/lib/shop.ts` (+ комментарии в `lib/types.ts`). `products.category`/`category_name` в типе `Product` **сохраняются**, но теперь = **главная категория** товара (наименьший `shop_categories.position`). Фильтрация каталога по категории и бренды-в-категории считаются через `product_categories` (резолв slug→id→product_ids, затем `.in("id", …)` — без embed-фильтров, максимально надёжно). `ProductDetail`/`ProductCard`/`ShopFilters` не трогаются. `products.category` из БД **не удаляется** в этом плане (drop — отдельным шагом после обновления коннектор-скилла, чтобы не сломать content-ops).

**Tech Stack:** Next.js 16 RSC, Supabase PostgREST (supabase-js), Vitest (unit), Playwright :3100 (e2e, live Supabase).

**Совместимость (сверено):** `searchProducts` (`lib/search.ts`) категорию НЕ читает и бренд ищет по `products.brand` (текст) — **не трогаем**. Бренд остаётся текстом на `products` — фильтр/поиск по бренду продолжают работать.

---

## Структура файлов

- Modify: `avocado.kiss/lib/shop.ts` — `attachCategoryNames` (→ M2M), `fetchProductsPage` (фильтр категории), `fetchShopFilterOptions` (бренды в категории), новый экспорт `primaryCategory`, тип `ProductRow`.
- Modify: `avocado.kiss/lib/types.ts` — комментарии к `Product.category`/`category_name` (теперь «главная категория»).
- Create: `avocado.kiss/lib/shop.test.ts` — unit-тесты (Vitest).
- Modify (если есть shop-e2e) / Create: `avocado.kiss/e2e/shop.spec.ts` — e2e фильтра категории.
- Modify: `platform-docs/sites/avocado-kiss.md` §8 + `platform-docs/database/schema.md` (пометка, что сайт читает M2M).

Порядок сортировки для «главной» категории — `position asc`, tiebreaker `slug asc` (детерминизм).

---

## Task 1: `primaryCategory` + перевод `attachCategoryNames` на M2M

**Files:**
- Modify: `avocado.kiss/lib/shop.ts:17-46`
- Create: `avocado.kiss/lib/shop.test.ts`

- [ ] **Step 1: Написать падающий тест на `primaryCategory` и M2M-`attachCategoryNames`**

Создать `avocado.kiss/lib/shop.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";
import { primaryCategory, fetchProductBySlug } from "./shop";
import type { DbClient } from "./supabase/client";

// Мок клиента с результатом per-таблица (from(table) → свой билдер/данные).
function makeShopClient(byTable: Record<string, { data: unknown; error: unknown }>) {
  const calls: Record<string, Record<string, ReturnType<typeof vi.fn>>> = {};
  const client = {
    from: vi.fn((table: string) => {
      const result = byTable[table] ?? { data: [], error: null };
      const builder: Record<string, ReturnType<typeof vi.fn>> & { then?: unknown } = {};
      for (const m of ["select","eq","in","gte","lt","order","limit","range","maybeSingle"]) {
        builder[m] = vi.fn(() => builder);
      }
      builder.then = (onF: (v: typeof result) => unknown, onR?: (e: unknown) => unknown) =>
        Promise.resolve(result).then(onF, onR);
      calls[table] = builder;
      return builder;
    }),
  } as unknown as DbClient;
  return { client, calls };
}

describe("primaryCategory", () => {
  it("выбирает категорию с наименьшим position (tiebreaker slug)", () => {
    expect(
      primaryCategory([
        { slug: "pantry", name: "Pantry", position: 5 },
        { slug: "cookware", name: "Cookware", position: 1 },
      ]),
    ).toEqual({ slug: "cookware", name: "Cookware" });
  });
  it("пустой список → null", () => {
    expect(primaryCategory([])).toBeNull();
  });
});

describe("fetchProductBySlug (M2M category)", () => {
  it("достраивает главную категорию из product_categories", async () => {
    const { client } = makeShopClient({
      products: { data: { id: "p1", slug: "pan", title: "Pan", brand: "Field Co.", price: 40, description: null, image_path: null, referral_url: "x" }, error: null },
      product_categories: {
        data: [
          { product_id: "p1", shop_categories: { slug: "pantry", name: "Pantry", position: 5 } },
          { product_id: "p1", shop_categories: { slug: "cookware", name: "Cookware", position: 1 } },
        ],
        error: null,
      },
    });
    const product = await fetchProductBySlug(client, "pan");
    expect(product?.category).toBe("cookware");
    expect(product?.category_name).toBe("Cookware");
  });
});
```

- [ ] **Step 2: Запустить — тест падает (нет экспорта `primaryCategory`, старая логика)**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: FAIL (`primaryCategory` не экспортирован / category приходит из старого поля).

- [ ] **Step 3: Заменить блок `ProductRow` + `attachCategoryNames` в `lib/shop.ts`**

Заменить строки 17-46 на:

```ts
// products(*) в embed'ах приходит без вычисляемых полей категории — они из M2M.
type ProductRow = Omit<Product, "category" | "category_name">;

// Категория товара из join product_categories → shop_categories.
type CategoryRef = { slug: string; name: string; position: number };

/**
 * Главная категория товара = с наименьшим shop_categories.position (tiebreaker slug).
 * Используется для эйброу и бэклинка страницы товара. Пустой список → null.
 */
export function primaryCategory(cats: CategoryRef[]): { slug: string; name: string } | null {
  if (cats.length === 0) return null;
  const [top] = [...cats].sort((a, b) => a.position - b.position || a.slug.localeCompare(b.slug));
  return { slug: top.slug, name: top.name };
}

/**
 * Достраивает `category`/`category_name` (ГЛАВНУЮ категорию) списку товаров одним
 * запросом к product_categories (M2M). Товар без категорий → оба поля null.
 */
async function attachCategoryNames(
  client: DbClient,
  rows: ProductRow[],
): Promise<Product[]> {
  if (rows.length === 0) return [];
  const ids = rows.map((r) => r.id);
  const { data, error } = await client
    .from("product_categories")
    .select("product_id, shop_categories!inner(slug, name, position)")
    .in("product_id", ids);
  if (error) throw error;
  const catsByProduct = new Map<string, CategoryRef[]>();
  for (const row of (data ?? []) as unknown as {
    product_id: string;
    shop_categories: CategoryRef | null;
  }[]) {
    if (!row.shop_categories) continue;
    const list = catsByProduct.get(row.product_id) ?? [];
    list.push(row.shop_categories);
    catsByProduct.set(row.product_id, list);
  }
  return rows.map((r) => {
    const primary = primaryCategory(catsByProduct.get(r.id) ?? []);
    return { ...r, category: primary?.slug ?? null, category_name: primary?.name ?? null };
  });
}
```

- [ ] **Step 4: Запустить — тест зелёный**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd avocado.kiss && git add lib/shop.ts lib/shop.test.ts \
  && git commit -m "refactor(shop): resolve product category via product_categories M2M (primary category)"
```

---

## Task 2: `fetchProductsPage` — фильтр категории через M2M

**Files:**
- Modify: `avocado.kiss/lib/shop.ts:108-144`
- Modify: `avocado.kiss/lib/shop.test.ts`

- [ ] **Step 1: Дописать падающий тест фильтра категории**

Добавить в `shop.test.ts`:

```ts
import { fetchProductsPage } from "./shop";

describe("fetchProductsPage (M2M category filter)", () => {
  it("резолвит slug→id→product_ids и фильтрует products по .in(id)", async () => {
    const { client, calls } = makeShopClient({
      shop_categories: { data: { id: "c1" }, error: null },
      product_categories: { data: [{ product_id: "p1" }, { product_id: "p2" }], error: null },
      products: { data: [], error: null },
    });
    await fetchProductsPage(client, { categorySlug: "cookware", page: 0 });
    expect(client.from).toHaveBeenCalledWith("shop_categories");
    expect(client.from).toHaveBeenCalledWith("product_categories");
    expect(calls.product_categories.eq).toHaveBeenCalledWith("category_id", "c1");
    expect(calls.products.in).toHaveBeenCalledWith("id", ["p1", "p2"]);
  });

  it("категория без товаров → пустой массив без запроса products", async () => {
    const { client } = makeShopClient({
      shop_categories: { data: { id: "c1" }, error: null },
      product_categories: { data: [], error: null },
    });
    const res = await fetchProductsPage(client, { categorySlug: "empty", page: 0 });
    expect(res).toEqual([]);
    expect(client.from).not.toHaveBeenCalledWith("products");
  });
});
```

- [ ] **Step 2: Запустить — падает**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: FAIL (сейчас `fetchProductsPage` фильтрует `.eq("category", …)`).

- [ ] **Step 3: Переписать тело `fetchProductsPage` (строки 108-144)**

Заменить внутренность (от `const from = …` до `return attachCategoryNames(...)`) на:

```ts
  const from = page * pageSize;
  const to = from + pageSize - 1;

  // Категория теперь M2M: slug → id → множество product_id, затем .in("id", …).
  let productIds: string[] | null = null;
  if (categorySlug) {
    const { data: cat, error: catErr } = await client
      .from("shop_categories")
      .select("id")
      .eq("slug", categorySlug)
      .maybeSingle();
    if (catErr) throw catErr;
    if (!cat) return [];
    const { data: memberships, error: memErr } = await client
      .from("product_categories")
      .select("product_id")
      .eq("category_id", (cat as { id: string }).id);
    if (memErr) throw memErr;
    productIds = ((memberships ?? []) as { product_id: string }[]).map((m) => m.product_id);
    if (productIds.length === 0) return [];
  }

  let query = client.from("products").select("*");
  if (productIds) query = query.in("id", productIds);
  if (brand) query = query.eq("brand", brand);
  if (priceMin != null) query = query.gte("price", priceMin);
  if (priceMax != null) query = query.lt("price", priceMax);

  const ordered =
    sort === "price-asc"
      ? query.order("price", { ascending: true }).order("id", { ascending: false })
      : sort === "price-desc"
        ? query.order("price", { ascending: false }).order("id", { ascending: false })
        : query
            .order("created_at", { ascending: false })
            .order("id", { ascending: false });

  const { data, error } = await ordered.range(from, to);
  if (error) throw error;

  return attachCategoryNames(client, (data ?? []) as ProductRow[]);
```

Примечание: пагинация `.range()` применяется к products после `.in("id", …)` — порядок и постраничность сохраняются (категории у avocado — десятки товаров, `.in` со списком id безопасен).

- [ ] **Step 4: Запустить — зелёный**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd avocado.kiss && git add lib/shop.ts lib/shop.test.ts \
  && git commit -m "refactor(shop): filter product listing by category via product_categories"
```

---

## Task 3: `fetchShopFilterOptions` — бренды в категории через M2M

**Files:**
- Modify: `avocado.kiss/lib/shop.ts:152-166`
- Modify: `avocado.kiss/lib/shop.test.ts`

- [ ] **Step 1: Дописать падающий тест**

```ts
import { fetchShopFilterOptions } from "./shop";

describe("fetchShopFilterOptions (brands within category)", () => {
  it("бренды только товаров категории (через product_categories), distinct+sorted", async () => {
    const { client, calls } = makeShopClient({
      shop_categories: { data: { id: "c1" }, error: null },
      product_categories: { data: [{ product_id: "p1" }, { product_id: "p2" }], error: null },
      products: { data: [{ brand: "Zed" }, { brand: "Acme" }, { brand: "Zed" }, { brand: null }], error: null },
    });
    const res = await fetchShopFilterOptions(client, "cookware");
    expect(calls.products.in).toHaveBeenCalledWith("id", ["p1", "p2"]);
    expect(res.brands).toEqual(["Acme", "Zed"]);
  });

  it("без категории → бренды всех товаров", async () => {
    const { client, calls } = makeShopClient({
      products: { data: [{ brand: "Acme" }], error: null },
    });
    const res = await fetchShopFilterOptions(client);
    expect(res.brands).toEqual(["Acme"]);
    expect(calls.products.in).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Запустить — падает**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: FAIL.

- [ ] **Step 3: Переписать тело `fetchShopFilterOptions` (строки 152-166)**

```ts
export async function fetchShopFilterOptions(
  client: DbClient,
  categorySlug?: string,
): Promise<{ brands: string[] }> {
  let query = client.from("products").select("brand");
  if (categorySlug) {
    const { data: cat, error: catErr } = await client
      .from("shop_categories")
      .select("id")
      .eq("slug", categorySlug)
      .maybeSingle();
    if (catErr) throw catErr;
    if (!cat) return { brands: [] };
    const { data: memberships, error: memErr } = await client
      .from("product_categories")
      .select("product_id")
      .eq("category_id", (cat as { id: string }).id);
    if (memErr) throw memErr;
    const ids = ((memberships ?? []) as { product_id: string }[]).map((m) => m.product_id);
    if (ids.length === 0) return { brands: [] };
    query = query.in("id", ids);
  }
  const { data, error } = await query;
  if (error) throw error;

  const rows = (data ?? []) as { brand: string | null }[];
  const brands = [
    ...new Set(rows.map((r) => r.brand).filter((v): v is string => Boolean(v))),
  ].sort((a, b) => a.localeCompare(b));
  return { brands };
}
```

- [ ] **Step 4: Запустить — зелёный**

Run: `npm --prefix avocado.kiss test -- shop.test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd avocado.kiss && git add lib/shop.ts lib/shop.test.ts \
  && git commit -m "refactor(shop): scope brand filter options to category via product_categories"
```

---

## Task 4: Обновить комментарии типов

**Files:**
- Modify: `avocado.kiss/lib/types.ts` (строки с `category`/`category_name` в `Product`)

- [ ] **Step 1: Обновить комментарии**

Заменить две строки в типе `Product`:

```ts
  category: string | null; // ГЛАВНАЯ категория (slug) — из M2M product_categories (наименьший position)
  category_name: string | null; // имя главной категории (эйброу/бэклинк страницы товара)
```

- [ ] **Step 2: Проверить typecheck**

Run: `npm --prefix avocado.kiss run build`
Expected: TS компилируется без ошибок (prerender тянет данные из base-one — M2M уже есть).

- [ ] **Step 3: Commit**

```bash
cd avocado.kiss && git add lib/types.ts \
  && git commit -m "docs(types): Product.category is now the primary M2M category"
```

---

## Task 5: Полная проверка (build + lint + unit)

**Files:** —

- [ ] **Step 1: Unit-тесты**

Run: `npm --prefix avocado.kiss test`
Expected: все тесты зелёные (в т.ч. `shop.test.ts`; `search.test.ts` не затронут).

- [ ] **Step 2: Build + lint**

Run: `npm --prefix avocado.kiss run build && npm --prefix avocado.kiss run lint`
Expected: build ок (TS + prerender `/shop`, `/shop/[category]`, `/product/[slug]`); lint без новых ошибок.

- [ ] **Step 3: Sitemap-инвариант**

Проверить, что список публичных роутов не изменился (категории/товары те же). `app/sitemap.ts` изменений не требует (загрузчики slug'ов не трогались) — только подтвердить.
Run: `grep -n "shop\|product\|category" avocado.kiss/app/sitemap.ts`
Expected: те же записи, ничего дописывать не нужно.

---

## Task 6: e2e фильтра категории (live Supabase, :3100)

**Files:**
- Create/Modify: `avocado.kiss/e2e/shop.spec.ts`

- [ ] **Step 1: Написать e2e (Playwright)**

Категория Pantry имеет 14 товаров (сверено). Тест: открыть `/shop/pantry`, увидеть сетку товаров; применить фильтр бренда — сетка сужается.

```ts
import { test, expect } from "@playwright/test";

test("category page shows products and brand filter narrows them", async ({ page }) => {
  await page.goto("/shop/pantry");
  const cards = page.locator("a[href^='/product/']");
  await expect(cards.first()).toBeVisible();
  const initial = await cards.count();
  expect(initial).toBeGreaterThan(0);

  const brandSelect = page.getByLabel("Filter by brand");
  const optionCount = await brandSelect.locator("option").count();
  if (optionCount > 1) {
    await brandSelect.selectOption({ index: 1 });
    await expect(cards.first()).toBeVisible();
    expect(await cards.count()).toBeLessThanOrEqual(initial);
  }
});
```

- [ ] **Step 2: Запустить e2e**

Run: `npm --prefix avocado.kiss run test:e2e -- shop`
Expected: PASS (dev-сервер :3100 поднимается конфигом Playwright; читает live base-one).

- [ ] **Step 3: Commit**

```bash
cd avocado.kiss && git add e2e/shop.spec.ts \
  && git commit -m "test(shop): e2e category listing + brand filter (M2M)"
```

---

## Task 7: Документация + пуш

**Files:**
- Modify: `platform-docs/sites/avocado-kiss.md` §8, `platform-docs/database/schema.md`

- [ ] **Step 1: Обновить §8 avocado-kiss.md**

Отразить: связь товар↔категория теперь M2M (`product_categories`); `Product.category` = главная категория (наименьший position); фильтр каталога и бренды-в-категории идут через M2M; `products.category` в БД пока сохранён (drop позже).

- [ ] **Step 2: Пометить в schema.md**

В описании `products.category` (уже помечено «устаревает») — добавить, что **сайт больше не читает** колонку (читает M2M) начиная с этого изменения.

- [ ] **Step 3: Commit + push всех репо (с разрешения)**

```bash
cd avocado.kiss && git push
cd ../platform-docs && git add sites/avocado-kiss.md database/schema.md \
  && git commit -m "docs(shop): site reads product categories via M2M" && git push
```

---

## Готово, когда

- `npm --prefix avocado.kiss test` и `run build`/`lint` зелёные; e2e фильтра проходит.
- Сайт нигде не читает `products.category` (grep по `lib/` пуст, кроме, возможно, комментариев).
- `products.category` в БД сохранён (удаляется после обновления коннектор-скилла — План 4/финальная очистка), чтобы не сломать content-ops.

## НЕ в этом плане (следующие)

- **Drop `products.category`** — отдельным шагом после обновления `avocado-content-ops`, чтобы новые товары из коннектора не терялись (иначе запись в `products.category` без M2M-строки → товар не виден на сайте).
- Разделы админки (План 3), обновление коннектор-скилла (План 4).
