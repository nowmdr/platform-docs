# Avocado Kiss — Curated Shop, спека A: фронтенд/UI

> Дата: 2026-08-01 | Статус: одобрено пользователем | Проект: `avocado.kiss`
> Парная спека данных: `2026-08-01-shop-data-design.md`
> Источник дизайна: `avocado.kiss/mockups/version-2/shop/avocado-shop{,-category,-product}.html`

Раздел **Curated Shop** — витрина товаров кулинарного журнала (посуда, керамика,
пантри), которыми редакция реально пользуется, со ссылками «где купить». Три
страницы: хаб магазина, страница категории, страница товара.

Работа разбита на **две параллельные спеки**, собираемые в одно целое через общий
контракт (`lib/types.ts` + сигнатуры `lib/shop.ts`):

- **Спека A (этот файл)** — компоненты, маршруты, CSS Modules. Строится против
  контракта и работает **standalone уже сейчас** на типизированном фикстур-адаптере
  (`lib/shop-fixtures.ts`), без БД. Пригодна к ручному тестированию сразу.
- **Спека B** — миграция `avocado_kiss`, реальные лоадеры `lib/shop.ts`, посев
  контента через скилл `avocado-content-ops`.
- **Сборка**: `page.tsx` импортируют лоадеры из `lib/shop.ts`. Спека A отдаёт их
  указывающими на фикстуры; спека B заменяет **внутренности** на Supabase. Сигнатуры
  и типы неизменны → UI не трогается, фича становится целой одним переключением.

## Принятые решения

| Вопрос | Решение |
|---|---|
| Вёрстка | Как весь avocado: дизайн-токены `globals.css` + CSS Modules, без Tailwind (макет на Tailwind переписываем 1:1 по значениям) |
| Маршрут товара | Верхнеуровневый `/product/[slug]` (как cozycorner) — не вложен в `/shop`, коллизии категорий/товаров нет |
| Load more | **Кнопка** (не бесконечный скролл) — как в макете категории |
| Фильтры | Панель фильтров **только презентационная** (3 селекта, без логики) в этой итерации |
| Картинки | Все тестовые товары `image_path: null` → существующий `MediaPlaceholder` |
| Данные в спеке A | Фикстуры `lib/shop-fixtures.ts`, тот же контракт, что у реальных лоадеров |
| Курация | Editors' picks / Pairs well with / Related reading приходят готовыми списками из лоадеров — UI не знает, курировано это или авто |

## 1. Маршруты

| Маршрут | Содержимое | Лоадеры (из `lib/shop.ts`) |
|---|---|---|
| `/shop` | Hero + «Shop by category» + «Editors' picks this month» (4 товара) | `fetchShopCategories`, `fetchEditorsPicks` |
| `/shop/[category]` | Hero категории + панель фильтров (статич.) + сетка 3×3 + Load more | `fetchShopCategoryBySlug`, `fetchProductsPage` |
| `/product/[slug]` | Детали товара + «Pairs well with» (3) + «Related reading» (3 рецепта) | `fetchProductBySlug`, `fetchProductPairings`, `fetchRelatedReading` |

- Все страницы — SSG + ISR `revalidate = 60` (стандарт репо), без `force-cache` в
  Supabase-fetch. `generateStaticParams` — из `fetchShopCategorySlugs` /
  `fetchProductSlugs` (см. контракт).
- Пункт «Curated Shop» в навигации шапки уже существует (в макете home ведёт на
  `avocado-shop.html`) — привязать `href="/shop"`.
- Ненайденные slug'и категории/товара → `notFound()` (стандартная 404 репо).
- SEO: `generateMetadata` через `fetchPageSeo('shop')` для хаба; для категории/товара
  — из полей `seo_*` соответствующей записи с фолбэком на `name`/`title` (детали
  формул — в спеке B; UI лишь дергает готовые значения).

## 2. Общий контракт (seam интеграции A↔B)

**Идентичен в обеих спеках.** Спека A реализует это на фикстурах, спека B — на
Supabase. Меняются только внутренности функций.

### Типы (`lib/types.ts`)

```ts
export type Product = {
  id: string;
  created_at: string;
  slug: string;               // /product/[slug]
  title: string;
  brand: string | null;       // «Field Co.» — эйброу карточки
  price: number;              // число; форматируется formatPrice()
  description: string | null; // тело страницы товара
  image_path: string | null;  // плоский ключ бакета avocado-kiss-photos или URL; тест → null
  referral_url: string;       // «Buy from …» / «Shop»
  category: string | null;    // = shop_categories.slug (текст, без FK — паттерн recipes.category)
  category_name: string | null; // отображаемое имя категории (для эйброу/бэклинка страницы товара)
};

export type ShopCategory = {
  id: string;
  slug: string;               // /shop/[slug]
  name: string;               // «Ceramics & Table»
  item_count: number;         // «31 items» на карточке хаба
  position: number;           // порядок в сетке хаба
  hero_eyebrow: string | null;// эйброу hero (фолбэк «Category»)
  hero_title: string | null;  // фолбэк name
  hero_description: string | null;
  hero_image_path: string | null;
  seo_title: string | null;
  seo_description: string | null;
};

// Разрешённый элемент «Related reading»: рецепт, приведённый к карточке-ссылке
// (UI не знает про источник). eyebrow — категория/тег рецепта.
export type ReadingItem = {
  id: string;
  href: string;               // /recipes/[slug]
  eyebrow: string | null;
  title: string;
  imagePath: string | null;
};

export const SHOP_PAGE_SIZE = 9; // сетка 3×3; страница Load more
```

### Лоадеры (`lib/shop.ts`) — сигнатуры

```ts
fetchShopCategories(client): Promise<ShopCategory[]>                 // порядок position
fetchShopCategoryBySlug(client, slug): Promise<ShopCategory | null>
fetchShopCategorySlugs(client): Promise<string[]>                    // generateStaticParams
fetchProductsPage(client, opts: { categorySlug?: string; page: number;
    pageSize?: number }): Promise<Product[]>                         // пагинация, детерминизм
fetchEditorsPicks(client): Promise<Product[]>                        // курировано, 4
fetchProductBySlug(client, slug): Promise<Product | null>
fetchProductSlugs(client): Promise<string[]>                         // generateStaticParams
fetchProductPairings(client, product: Product): Promise<Product[]>   // курировано, 3
fetchRelatedReading(client, product: Product): Promise<ReadingItem[]>// курировано, 3

resolveProductImage(path: string | null): { url; unoptimized } | null // бакет avocado-kiss-photos
formatPrice(price: number): string                                   // Intl en-US USD
productPath(p: Pick<Product,'slug'>): string                         // `/product/${slug}`
```

`client: DbClient` — сервер (RSC) или браузер (Load more), как во всём репо.

### Фикстур-адаптер (`lib/shop-fixtures.ts`, только спека A)

- Экспортирует массивы `PRODUCTS`, `SHOP_CATEGORIES` + курированные связи
  (`EDITORS_PICKS`, `PAIRINGS`, `RELATED_READING`) — ~30 товаров, 6 категорий
  (см. §5). Реальные рецепты для Related reading в фикстурах — заглушечные
  `ReadingItem` (спека B перевяжет на настоящие рецепты).
- `lib/shop.ts` в спеке A реализует контракт **поверх** этих массивов (игнорирует
  `client`, отдаёт срезы/фильтры синхронно, обёрнутые в Promise). `fetchProductsPage`
  режет `SHOP_PAGE_SIZE`-страницами с детерминированным порядком.
- Спека B перепишет тело `lib/shop.ts` на Supabase; `lib/shop-fixtures.ts` тогда
  удаляется (или остаётся источником для DB-seed — решает спека B).

## 3. Компоненты (создать один раз, переиспользовать)

Один компонент = один файл + свой `.module.css`. Токены-first, значения из макета
через `var(--token)`; новое значение → новый токен в `globals.css`.

| Компонент | Где | Разметка/особенности |
|---|---|---|
| `ProductCard` | хаб (picks), сетка категории, «Pairs well with» | **Одна карточка, идентична везде.** `<Link href={productPath}>`: `imageWrap` (aspect как в макете) → `Image`\|`MediaPlaceholder`; тело: `brand` (eyebrow), `title`, ряд `price` + «Shop». `image_path: null` → `MediaPlaceholder`. |
| `ShopCategoryCard` | хаб, «Shop by category» | Отдельный тип: `imageWrap` → картинка/плейсхолдер; `name` (b) + «{item_count} items» (span). `<Link href="/shop/{slug}">`. |
| `ShopHero` | `/shop` и `/shop/[category]` | Общая форма: фон-картинка + `eyebrow` + `h1` + `p`. Props: `eyebrow`, `title`, `description`, `imagePath`. Хаб: контент-константа (§4); категория: из `ShopCategory.hero_*` с фолбэками. |
| `ShopCategoryGrid` | хаб | Обёртка-сетка + заголовок «Shop by category»; маппит `ShopCategoryCard`. |
| `EditorsPicksProducts` | хаб | Заголовок «Editors' picks this month» + ряд из 4 `ProductCard`. (Отдельно от рецептного `EditorsPicks` — не путать.) |
| `ProductGrid` | `/shop/[category]` | **`'use client'`**. Props: `initialProducts`, `categorySlug`. Держит список + `page`; **кнопка Load more** дозагружает через `fetchProductsPage(browserClient, {categorySlug, page})`; дедуп по `id`; прячет кнопку когда пришло `< SHOP_PAGE_SIZE`. Сетка 3 колонки. Опционально `Reveal` для стаггера (как в cozycorner). |
| `ShopFilters` | `/shop/[category]` | **Статическая** панель: «Filter by» + 3 `<select>` (brands / price / sort) с опциями из макета. Без обработчиков/фильтрации в этой итерации. Презентационный компонент. |
| `ProductDetail` | `/product/[slug]` | Двухколоночный блок: слева `imageWrap` (картинка/плейсхолдер), справа: `category_name` (eyebrow) + `h1 title` + `p description` + `price` + кнопка-ссылка «Buy from {brand}» (`referral_url`, target `_blank` rel `noopener`). Бэклинк «← Back to {category_name}» над блоком (`/shop/{category}`). |
| `RelatedProducts` | `/product/[slug]` | «Pairs well with»: заголовок + 3 `ProductCard`. Тонкая обёртка. |
| `RelatedReading` | `/product/[slug]` | «Related reading»: заголовок + 3 карточки-ссылки на рецепты (`imageWrap` + `eyebrow` + `h3`, **без цены**). Рендерит `ReadingItem[]`. |

**Разницы карточек (по требованию «обращай внимание на все разницы»):**
- *ProductCard* — brand / name / price / «Shop». Идентичен на всех трёх страницах.
- *ShopCategoryCard* — name + «N items», **без бренда/цены**.
- *RelatedReading card* — eyebrow + title, **без цены/бренда/«Shop»**; ведёт на
  `/recipes/*`, не на товар.

Иконки (стрелка бэклинка, chevron селекта при необходимости) — компонентами в
`components/icons/` (паттерн репо).

## 4. Контент хаба (константа)

Hero `/shop` редко меняется — держим editorial-константой в коде страницы (или через
`fetchPageSeo`, но текст hero — константа):
`eyebrow: «Curated Shop»`, `title: «Everything we cook with»`,
`description: «Pans, ceramics and pantry staples our editors actually use in the test
kitchen — with links to where you can buy them.»` (точь-в-точь макет).

## 5. Тестовый контент (комфортно для ручного теста)

Единый список — источник и для фикстур (A), и для DB-seed (B) в спеке B.

- **6 категорий** (как в макете): Cookware, Ceramics & Table, Bakeware,
  Knives & Boards, Pantry, Linens — с `item_count`, `position`, hero-текстами.
- **~30 товаров**, распределены по категориям так, чтобы у нескольких категорий было
  **> 9** товаров (реальная работа сетки 3×3 + Load more хотя бы на одной странице).
  Реалистичные бренды (расширяем макет: Field Co., Clay & Co., De Buyer, Berard,
  Sur Oliva, Maison Lin, Wild Loaf + ещё несколько), цены, описания. Товары из макета
  включить дословно.
- **4** товара — курированные Editors' picks (`shop_editors_picks`).
- **Pairs well with**: у каждого товара 3 курированные пары.
- **Related reading**: у каждого товара 3 связи с рецептами (в фикстурах — заглушки
  `ReadingItem`; спека B привязывает к реальным опубликованным рецептам).
- Все `image_path: null` → `MediaPlaceholder`; все `referral_url` — правдоподобные
  внешние ссылки-заглушки (`https://example.com/...`).

## 6. Стиль / токены

- Переносим значения макета 1:1 в `var(--token)`. Палитра v1 уже в `:root`
  (`--background:#fcfaf6`, `--foreground:#0b1c2c`, `--paper`, `--muted`,
  `--muted-foreground:#576574`, `--accent:#467748`, `--rule`, `--border`,
  `--radius:.25rem`, `--tracking-eyebrow:.22em`, шкала типографики). Новых
  hardcode-значений нет; новое → новый токен. Утилита `.eyebrow` переиспользуется.
- Сетки/aspect-ratio карточек — из макета (сверить пропорции imageWrap для product
  vs category vs reading). `overflow-x` html/body остаётся `clip`.
- Адаптив как в макете (моб. 1 колонка → десктоп 3). Ховеры → accent.

## 7. Границы спеки A (не делаем)

- Никакой БД/миграций/`avocado-content-ops` — это спека B.
- Никакой рабочей фильтрации/сортировки (панель статическая).
- Корзины/оплаты нет — только реферальные ссылки наружу.
- Поиск товаров — вне объёма.
- Не коммитим/пушим без разрешения пользователя.

## 8. Проверка готовности A

- `npm run build` (TS + prerender всех трёх маршрутов на фикстурах) и `npm run lint`
  зелёные.
- Ручной проход: `/shop` (сетка категорий + 4 picks), `/shop/[category]` (3×3 +
  Load more добирает следующую страницу), `/product/[slug]` (детали + pairs + reading
  + бэклинк). Плейсхолдеры не ломают сетку.
- Тесты (Vitest+RTL) на `ProductCard`, `ProductGrid` (Load more/дедуп), `ShopHero`
  фолбэки — по стратегии `platform-docs/methodology/testing.md`. E2E — по желанию/
  отдельной просьбе.
