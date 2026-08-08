# Avocado Kiss — Curated Shop: M2M-категории, справочник брендов, папки + интеграция в админку

> Date: 2026-08-08 | Scope: avocado_kiss (БД) + avocado.kiss (сайт) + web.admin (админка) + skill avocado-content-ops
> Related: [2026-08-01-shop-data-design.md](2026-08-01-shop-data-design.md) (исходная модель Shop),
> [../../../admin-panel/products.md](../../../admin-panel/products.md) (раздел Products cozy — образец),
> [../../../database/schema.md](../../../database/schema.md) §9 (схема avocado_kiss)

## 1. Цель и контекст

Привести Curated Shop у Avocado Kiss к той же модели, что у CozyCorner, чтобы
управлять товарами/категориями/брендами магазина из общей админки `web.admin`
теми же generic-компонентами (Products / Categories / Brands / папки), что и у
cozy. Пользователь осознанно выбрал **паритет с cozy** (M2M-категории + справочник
брендов + папки), а не адаптацию админки под текущую упрощённую модель avocado.

Текущая модель avocado (миграция `0007_shop.sql`) отличается от cozy:
- товар↔категория — **одна** категория текстом (`products.category` = `shop_categories.slug`, без FK);
- **нет** таблицы `brands` — `products.brand` свободный текст;
- у товаров **нет** `folder_id`; `admin_folders.section` CHECK = `('media','recipes')`;
- `shop_categories.item_count` — **вручную** поддерживаемый счётчик (см. skill avocado-content-ops);
- у товаров avocado **нет** `image_style` / `seo_title` / `seo_description`.

Это затрагивает 4 поверхности: БД `avocado_kiss`, сайт `avocado.kiss`, админку
`web.admin`, коннектор-скилл `avocado-content-ops`.

### Ключевое требование пользователя по совместимости
На странице категории есть **фильтр по брендам**, и на сайте есть **поиск по бренду**.
Оба должны продолжать работать, данные — оставаться корректными.

## 2. Целевая модель данных (схема `avocado_kiss`)

| Изменение | Детали |
|---|---|
| `product_categories` (новая, M2M) | `product_id uuid → products(id) on delete cascade`, `category_id uuid → shop_categories(id) on delete cascade`, PK `(product_id, category_id)`, индекс по `category_id`. Товар ↔ много **shop_categories** (не `categories` — те про рецепты). |
| `brands` (новый справочник) | `id, created_at, name (unique), slug (unique), position` — точная копия `cozycorner.brands`. Публичное чтение + запись `is_admin()`. |
| `products.folder_id` | nullable `uuid → admin_folders(id) on delete set null` + индекс. |
| `admin_folders.section` CHECK | расширить `('media','recipes')` → `('media','recipes','products')`. |
| `products.seo_title` / `seo_description` | добавить (nullable text). Живой SEO-фолбэк как у cozy: пусто → сайт использует `title`/`description`. |
| `item_count` — триггер | триггер на `product_categories` (AFTER INSERT/DELETE) пересчитывает `shop_categories.item_count = count(*) product_categories where category_id = …`. Заменяет ручной счётчик. |
| `products.category` (текст) | **сохраняется на время перехода**, удаляется только в фазе очистки (после переключения сайта). |
| slug-триггеры | BEFORE INSERT/UPDATE на `products` и `shop_categories` — генерируют slug из `title`/`name`, если пуст (по образцу cozy `products_set_slug`). Нужны, чтобы админ-код совпадал с cozy. |

**НЕ добавляем `image_style`**: у avocado товары всегда «фото», выбора стиля картинки
нет ни на сайте, ни в админке (решение пользователя). Форма товара в админке делается
site-aware — для avocado поля `image_style` нет.

`products.brand` **остаётся text** и остаётся источником истины для сайта. `brands` —
это только словарь для пикера в админке (как у cozy: `products.brand = brands.name`,
не FK).

## 3. Миграция данных и совместимость

Все посевы — в той же миграции, что создаёт структуры (фаза 1), до того как что-либо
начнёт читать новые таблицы:

1. **`product_categories`** засевается из текущего `products.category`:
   `insert ... select p.id, c.id from products p join shop_categories c on c.slug = p.category where p.category is not null`.
   Лослесс: у каждого товара сейчас одна категория → одна строка membership.
2. **`brands`** засевается из различных непустых `products.brand`
   (`insert ... select distinct brand ...` + сгенерировать slug, `position` по алфавиту).
   Так combobox бренда в админке сразу совпадает с реальными данными.
3. **`item_count`** после посева `product_categories` пересчитывается триггером/явным
   апдейтом — устраняет возможный дрейф ручного счётчика.

**Совместимость (проверено по коду сайта, см. §5):**
- Фильтр брендов на странице категории (`fetchShopFilterOptions`) и поиск по бренду
  (`searchProducts` ilike) читают `products.brand` (text) → **работают без изменений**.
- Переименование бренда в админке каскадит в `products.brand` (app-level, как у cozy) →
  пикер, фильтр и поиск остаются согласованы.
- `product_categories` засеян до переключения сайта → переход на M2M лослесс.

## 4. Безопасный поэтапный выкат (сайт не ломается ни на одном шаге)

1. **Фаза 1 — аддитивная миграция БД** (`avocado.kiss/supabase/migrations/`): brands +
   product_categories + `products.folder_id` + продуктовый SEO + CHECK `products` +
   триггер `item_count` + посев. `products.category` **остаётся**. Сайт работает как есть.
2. **Фаза 2 — сайт**: `avocado.kiss` читает категории через M2M-join; деплой.
3. **Фаза 3 — очистка БД**: удалить `products.category` + индекс, когда сайт больше не читает.
4. **Фаза 4 — админка**: разделы Products / Shop Categories / Brands (site-aware),
   добавить в `sections` avocado. Зависит только от фазы 1 → может идти параллельно 2–3.
5. **Фаза 5 — коннектор-скилл**: обновить `avocado-content-ops` SKILL.md под новую модель.

Все миграции avocado живут в репозитории `avocado.kiss/supabase/migrations/`
(изменения схемы НЕ делаются из web.admin — правило `web.admin/AGENTS.md`). Применение
через Supabase MCP дублируется файлом миграции.

## 5. Изменения сайта `avocado.kiss` (фаза 2)

Инвентарь мест, читающих текущую связь (по обследованию кода):

- **`lib/shop.ts`**:
  - `attachCategoryNames` (читает `r.category`, lookup по `shop_categories.slug`) →
    переписать на M2M: подтягивать категории товара через `product_categories`
    (embed или отдельный запрос), выставлять **главную категорию** (наименьший
    `shop_categories.position`) в `category`/`category_name` + полный список
    `categories: {slug,name}[]`.
  - `fetchProductsPage` — фильтр `.eq("category", categorySlug)` (line ~125) →
    фильтр по членству через join `product_categories` (например
    `.select("*, product_categories!inner(category_id)")` + `.eq` по id категории,
    либо предвыборка product_id по категории). Фильтр `.eq("brand", brand)` — **без изменений**.
  - `fetchShopFilterOptions` — сейчас `select("brand")` scoped `.eq("category", slug)`
    (line ~156) → бренды товаров **в категории** через тот же join; значения по-прежнему
    из `products.brand`.
- **`lib/types.ts`**: `Product.category` (single slug) → **главная категория** (для
  эйброу/бэклинка) + новое поле `categories: {slug,name}[]`.
- **`components/ProductDetail.tsx`**: эйброу (`category_name`) и **бэклинк**
  (`/shop/${product.category}`, line ~14) — использовать главную категорию (наименьший
  `position`). Для товаров с одной категорией поведение не меняется. Новых кнопок не добавляем.
- **`lib/search.ts`**: `searchProducts` категорию не читает — **без изменений**; бренд
  ищется по `products.brand` (остаётся text) — **без изменений**.
- **`shop_categories.item_count`**: сайт читает как есть (`fetchShopCategories`,
  `ShopCategoryCard`) — теперь значение корректно за счёт триггера, код сайта не трогаем.
- **`app/sitemap.ts`**: набор роутов не меняется (категории и товары перечисляются
  теми же загрузчиками) — сверить по правилу «новый/изменённый роут → sitemap».

Загрузчики курации (`shop_editors_picks`/`product_pairings`/`product_reading`)
ссылаются на `products(id)` по FK и **не затрагиваются**.

## 6. Изменения админки `web.admin` (фаза 4)

Переиспользуем generic-код cozy максимально; site-aware только там, где модель
отличается.

- **Brands** — таблица называется `brands`, поля совпадают → существующий
  `brandConfig` + `TaxonomyManager` + `TaxonomyCombobox` работают как есть (через
  `getDb(site)`). Каскад rename/delete в `products.brand` — тот же код `lib/brands.ts`.
- **Shop Categories** — новый site-aware taxonomy-конфиг, указывающий на
  `shop_categories` с её набором полей (`name`, `hero_eyebrow`, `hero_title`,
  `hero_description`, `hero_image_path`, `seo_title`, `seo_description`, `position`).
  Переиспользуем `TaxonomyManager` + отдельный edit-dialog под поля shop-категории
  (у cozy `categories` поля другие: `image_path`, `hero_badge` — их не мешать).
  Страница категории — список товаров-членов (M2M) + **Add product / Remove from
  category**, как у cozy (`CategoryProductsPage`), через `product_categories`.
- **Products** — переиспользуем `ProductsPage` / `ProductEditPage` / `CategoryChipsField`
  (M2M-чипы), папки, `ImagePickerDialog`. Форма товара site-aware по дельте колонок:
  **нет `image_style`** у avocado; slug — read-only/автоген через slug-триггер (см. ниже).
  `folder_id` — как у cozy (в `ProductListItem`, редактор не трогает).
- **Реестр/навигация**: добавить в `SITES[avocado-kiss].sections`
  `["media","products","shop-categories","brands"]` (точные `to`/лейблы уточнить при
  реализации; «Shop Categories» отдельно от будущих recipe-«Categories»). Нужен
  site-aware маппинг «какая таблица категорий у сайта»: cozy → `categories`,
  avocado → `shop_categories`; и «какие поля у edit-dialog».
- **`USAGE_SOURCES`** (media, уже site-aware) — при добавлении полей картинок ничего
  нового: `products.image_path`, `shop_categories.hero_image_path` уже учтены.

### Slug-триггеры (решено)
У avocado `products`/`shop_categories` сейчас **нет** slug-триггера (skill
avocado-content-ops отмечает «no slug trigger»). Админка cozy полагается на
`products_set_slug`. Решение: **добавить аналогичные BEFORE INSERT/UPDATE slug-триггеры**
для `avocado_kiss.products` и `avocado_kiss.shop_categories` (генерация slug из
`title`/`name`, если slug пуст) — по образцу cozy, чтобы админ-код был идентичен.
Триггеры идут в аддитивной миграции фазы 1. Существующие строки уже имеют slug —
бэкофилл не нужен; триггер срабатывает только при пустом slug.

## 7. Обновление коннектор-скилла `avocado-content-ops` (фаза 5)

SKILL.md сейчас кодирует старую модель: `products.category` = один slug; `item_count`
поддерживается вручную; бренд — свободный текст без словаря. После миграции обновить:
- товар↔категория — через `product_categories` (много категорий), а не `products.category`;
- `item_count` больше **не** трогать вручную — его держит триггер;
- бренды — писать `products.brand` (text) как раньше, но новые бренды заодно заводить
  строкой в `brands` (или оставить это админке — уточнить в плане);
- убрать инструкции про `products.category` и про ручной пересчёт `item_count`.

## 8. Тестирование

- **avocado.kiss**: unit на изменённые загрузчики `lib/shop.ts` (M2M-фильтр,
  filter-options по join, главная категория); e2e магазина (:3100, live Supabase) —
  страница категории с фильтром бренда, бэклинк товара. Порядок и порты — методология.
- **web.admin**: компонентные тесты новых site-aware конфигов (shop-categories),
  прогон существующих тестов Products/Brands на схеме avocado; `npm run build` +
  `npm run lint` (oxlint, 2 известных warning ок).
- **Данные**: после посева — сверка `product_categories` (кол-во membership = кол-во
  товаров с непустой категорией), `item_count` = реальному членству, `brands` = distinct
  `products.brand`.

## 9. Вне объёма / риски

- Не трогаем `recipes.category`, `categories` (рецептные) и `fetchRelatedReading`
  (там `recipes.category`, не про магазин).
- Риск рассинхрона `products.brand` (text) и `brands` (словарь) при правках мимо
  админки — снимается тем, что источник истины для сайта остаётся `products.brand`.
- Фаза 3 (drop `products.category`) — только после деплоя фазы 2, иначе прод-сайт
  сломается. До деплоя фазы 2 колонка обязана существовать.
- Многопользовательский дрейф `item_count` при ручных вставках в обход триггера —
  исключён, т.к. триггер на самой `product_categories`.

## 10. Решения (зафиксировано с пользователем)

- (a) **`image_style` не добавляем** — avocado всегда «фото», выбора стиля нет.
- (b) **Бэклинк/эйброу на странице товара** ведут на **главную категорию** (наименьший
  `position`); новую кнопку не добавляем — меняем цель существующей. Это правка **сайта**,
  не админки.
- Админка Shop Categories — **полный вариант как cozy** (страница категории с товарами +
  Add/Remove product).
- Модель категорий — **M2M как cozy**; бренды — **справочник `brands` как cozy**; папки
  товаров — **добавляем как cozy**.
- **Slug-триггеры как в cozy** — добавляем на `products` и `shop_categories` (фаза 1),
  чтобы админ-код был идентичен cozy.
