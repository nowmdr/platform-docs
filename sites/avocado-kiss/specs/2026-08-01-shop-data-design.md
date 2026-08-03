# Avocado Kiss — Curated Shop, спека B: слой данных

> Дата: 2026-08-01 | Статус: одобрено пользователем | Проект: `avocado.kiss`
> Парная спека UI: `2026-08-01-shop-ui-design.md`

Слой данных раздела **Curated Shop**: схема `avocado_kiss` (товары, категории
магазина, курация), реальные лоадеры `lib/shop.ts` и посев тестового контента.
Реализует **тот же контракт**, что спека A строит на фикстурах, — после подмены
внутренностей `lib/shop.ts` фича собирается в одно целое без правок UI.

## Принятые решения

| Вопрос | Решение |
|---|---|
| Категории магазина | Отдельная таблица `shop_categories` (не рецептные `categories` — другая таксономия) |
| Связь товар→категория | **Текстовая** `products.category` = `shop_categories.slug`, без FK — паттерн `recipes.category` этого репо (не M2M cozycorner) |
| Бренд | Текстовая колонка `products.brand` (без справочника — фильтры пока не функциональны) |
| Курация | **Три таблицы**, по образцу `home_slots`: `shop_editors_picks`, `product_pairings`, `product_reading` |
| Related reading | Курируется на **реальные** `recipes` (`product_reading.recipe_id` → `recipes.id`) |
| Картинки | Тестовый контент `image_path: null`; `resolveProductImage` — бакет `avocado-kiss-photos` |
| Посев | Через скилл `avocado-content-ops` из общего списка §5 спеки A (или seed-миграцией) |

## 1. Миграция `0007_shop.sql` (схема `avocado_kiss`)

Следовать конвенциям 0001: `enable row level security` + публичные select-политики
+ admin-write через `is_admin()`; `grant select … to anon`, `grant all … to
authenticated, service_role`. `overflow`/индексы — как у существующих таблиц.

### Таблицы

```sql
-- Категории магазина (таксономия товаров; независима от рецептных categories)
shop_categories (
  id uuid pk default gen_random_uuid(),
  created_at timestamptz not null default now(),
  slug text not null unique,            -- /shop/[slug]
  name text not null,                   -- «Ceramics & Table»
  position int not null default 0,      -- порядок сетки хаба
  hero_eyebrow text, hero_title text, hero_description text, hero_image_path text,
  seo_title text, seo_description text
)

-- Товары
products (
  id uuid pk default gen_random_uuid(),
  created_at timestamptz not null default now(),
  slug text not null unique,            -- /product/[slug]
  title text not null,
  brand text,                           -- эйброу карточки; текст, без справочника
  price numeric(10,2) not null,
  description text,
  image_path text,                      -- плоский ключ avocado-kiss-photos или URL; тест → null
  referral_url text not null,           -- «Buy from …»
  category text                         -- = shop_categories.slug (ТЕКСТ, без FK — паттерн recipes.category)
)
-- индекс: (category), (created_at desc, id desc) для детерминированной пагинации

-- Курация: 4 товара в «Editors' picks this month» на /shop
shop_editors_picks (
  id uuid pk default gen_random_uuid(),
  product_id uuid not null references avocado_kiss.products(id) on delete cascade,
  position int not null default 0,
  unique (product_id)
)

-- Курация: «Pairs well with» на странице товара (товар → товар)
product_pairings (
  id uuid pk default gen_random_uuid(),
  product_id uuid not null references avocado_kiss.products(id) on delete cascade,
  paired_product_id uuid not null references avocado_kiss.products(id) on delete cascade,
  position int not null default 0,
  unique (product_id, paired_product_id),
  check (product_id <> paired_product_id)
)

-- Курация: «Related reading» на странице товара (товар → РЕАЛЬНЫЙ рецепт)
product_reading (
  id uuid pk default gen_random_uuid(),
  product_id uuid not null references avocado_kiss.products(id) on delete cascade,
  recipe_id uuid not null references avocado_kiss.recipes(id) on delete cascade,
  position int not null default 0,
  unique (product_id, recipe_id)
)
```

### RLS / гранты (как 0001)

- `enable row level security` на всех 5 таблицах.
- Публичное чтение: `shop_categories`, `products`, `shop_editors_picks`,
  `product_pairings` — `for select using (true)`. `product_reading` — `using (true)`,
  но лоадер embed'ит `recipes!inner` с `is_published = true`, чтобы неопубликованные
  рецепты не протекали в блок (паттерн `fetchHomeSlots`).
- Admin-write через `is_admin()` (цикл/шаблон из 0001).
- `grant select on <таблицы> to anon`; `grant all to authenticated, service_role`.

### Страница SEO хаба

- Вставить строку в `pages`: `slug='shop'` (+ `seo_title`, `seo_description`,
  `og_image_path`) — для `generateMetadata` хаба через существующий `fetchPageSeo`.

## 2. Типы (`lib/types.ts`) — синхронизировать с БД

Добавить `Product`, `ShopCategory`, `ReadingItem`, `SHOP_PAGE_SIZE = 9` — **дословно
как в §2 спеки A** (контракт единый). `category_name` в `Product` — вычисляемое поле
(join к `shop_categories.name` по slug), не колонка.

## 3. Лоадеры (`lib/shop.ts`) — реализация на Supabase

Реализовать сигнатуры контракта (§2 спеки A) поверх Supabase. Ключевые моменты:

- `fetchShopCategories` — `order('position').order('name')`.
- `fetchShopCategoryBySlug` — `eq('slug').maybeSingle()`.
- `fetchProductsPage({categorySlug, page, pageSize=SHOP_PAGE_SIZE})` —
  `eq('category', categorySlug)` при наличии; `.order('created_at',{desc}).order('id',
  {desc})` (детерминизм — как cozycorner) `.range(from,to)`. Возвращать с
  `category_name` (доп. lookup или embed к `shop_categories`).
- `fetchEditorsPicks` — `shop_editors_picks` embed `product:products(*)`,
  `order('position')`, limit 4.
- `fetchProductBySlug` — `eq('slug').maybeSingle()`, дополнить `category_name`.
- `fetchProductPairings(product)` — `product_pairings` где `product_id = product.id`,
  embed `paired:products(*)`, `order('position')`, вернуть массив товаров (3).
- `fetchRelatedReading(product)` — `product_reading` где `product_id = product.id`,
  embed `recipe:recipes!inner(slug,title,category,hero_image_path,is_published)`
  с `eq('recipe.is_published', true)`, `order('position')`; спроецировать в
  `ReadingItem` (`href=/recipes/${slug}`, `eyebrow=category`, `imagePath=hero_image_path`).
- `fetchShopCategorySlugs` / `fetchProductSlugs` — для `generateStaticParams`.
- `resolveProductImage(path)` — копия контракта cozycorner, но бакет
  **`avocado-kiss-photos`** и `NEXT_PUBLIC_SUPABASE_URL`; относительный путь →
  public URL (наш, оптимизируем), внешний → `unoptimized`.
- `formatPrice` — `Intl.NumberFormat('en-US',{style:'currency',currency:'USD'})`.
- `productPath` — `/product/${slug}`.
- Файл серверный (`import "server-only"`) для RSC-лоадеров, но
  `fetchProductsPage`/`resolveProductImage`/`formatPrice`/`productPath` должны быть
  вызываемы из браузерного `ProductGrid` (Load more) — вынести клиентобезопасные
  части так, как это сделано в cozycorner (`lib/products.ts` без `server-only`).
  **Согласовать с UI-спекой**: `ProductGrid` импортирует `fetchProductsPage` +
  браузерный `supabase`-клиент.

## 4. Обновление документации

- `platform-docs/database/schema.md` — добавить 5 таблиц shop + строку `pages(shop)`.
- `platform-docs/sites/avocado-kiss.md` — маршруты `/shop`, `/shop/[category]`,
  `/product/[slug]`, курация shop-слотов, бакет картинок товаров.
- Реестр проектов не трогаем (проект уже есть).

## 5. Посев тестового контента (`avocado-content-ops`)

Из единого списка §5 спеки A (источник истины для контента):

- 6 `shop_categories` (Cookware, Ceramics & Table, Bakeware, Knives & Boards, Pantry,
  Linens) с `item_count`-совместимым числом товаров, `position`, hero-текстами.
- ~30 `products` (включая дословные из макета), `image_path=null`,
  правдоподобные `referral_url`. Распределить так, чтобы ≥1 категория имела >9
  товаров (проверка Load more).
- `item_count` категорий = фактическое число товаров категории (держать
  согласованным при посеве).
- 4 строки `shop_editors_picks`.
- По 3 строки `product_pairings` на товар (без самоссылок).
- По 3 строки `product_reading` на товар → реальные **опубликованные** рецепты
  (взять существующие из `recipes`; если рецептов мало — переиспользовать).
- Строка `pages(shop)` с SEO.

## 6. Границы спеки B (не делаем)

- UI/компоненты/CSS — это спека A.
- Функциональная фильтрация/сортировка (нет колонок/индексов под неё сверх нужного;
  панель фильтров статическая в A). Справочник брендов — не заводим.
- Корзина/оплата/инвентарь — нет.
- Не коммитим/пушим и не применяем миграцию на прод без разрешения пользователя;
  применение — MCP/дашборд по стандартному воркфлоу.

## 7. Проверка готовности B

- Миграция применяется чисто; `list_tables`/advisors без ошибок RLS.
- `lib/types.ts` синхронизирован; `lib/shop.ts` реализует весь контракт.
- Точечный смоук: анон-`select` видит published-контент, `product_reading` не отдаёт
  неопубликованные рецепты.
- После подмены `lib/shop.ts` на Supabase (замена фикстур из A): `npm run build` +
  `npm run lint` зелёные, три маршрута рендерятся на реальных данных.
- Интеграция A+B: `lib/shop-fixtures.ts` удаляется (или остаётся seed-источником),
  страницы работают без изменений (сигнатуры/типы неизменны).
