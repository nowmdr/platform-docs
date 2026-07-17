# Products CRUD (v1) — дизайн

Дата: 2026-07-10. Статус: утверждён пользователем (чат).

## Цель

Раздел Products в админке (спека §8/§10 фаза 3, первый шаг CRUD контента): список
товаров активного сайта одной колонкой (только названия), детальная страница с
картинкой и всеми полями, редактирование, создание и удаление. Пикер картинок из
Media — НЕ в этой итерации (путь картинки пока правится текстом, превью живое).

## UX

**Список `/:siteSlug/products`** (заменяет заглушку Coming soon):
- Шапка: `Products` + счётчик товаров, справа кнопка `New product`.
- Одна колонка строк-ссылок: только `title`, сортировка `created_at desc`
  (новые сверху). Строка — ссылка на `/:siteSlug/products/:id`, hover-подсветка.
- Состояния: skeleton-строки при загрузке; empty state «No products yet.»;
  ошибка — красная плашка (паттерн MediaPage).

**Детальная `/:siteSlug/products/:id`** и **создание `/:siteSlug/products/new`**:
- Сверху ссылка `← Back to products`.
- Слева превью картинки (`resolveImageUrl`, aspect-square, object-contain,
  плейсхолдер при пустом/битом пути), справа форма.
- Внизу: `Save` (disabled пока isSubmitting) и, только в режиме edit, `Delete`
  (destructive) с AlertDialog: предупреждение, что товар может использоваться в
  секциях постов блога; после удаления — возврат к списку + toast.
- `Product not found` (+ Back) — если id не существует.
- Создание: insert без slug (БД автогенерирует из title) → toast → navigate на
  страницу созданного товара.

## Форма (RHF + Zod + Field, паттерн LoginPage)

| Поле | Контрол | Валидация |
|---|---|---|
| title | Input | required |
| price | Input (числовой) | required, number ≥ 0 |
| referral_url | Input | required, валидный http(s) URL |
| brand | Input | optional |
| category | Select из `categories.name` + пункт None | optional |
| description | Textarea | optional |
| image_path | Input + живое превью | optional; плоский ключ или внешний URL |
| image_style | Select: photo / cutout | default photo |
| seo_title | Input | optional |
| seo_description | Textarea | optional |

- `slug`: не редактируется; в edit показывается read-only (текст под title).
- Пустые optional-поля пишем в БД как `null` (не пустые строки).
- Числовое price: input `type="number" step="0.01"`, в Zod — coerce к number.

## Архитектура

- **`src/lib/products.ts`**: тип `Product` (по спеке §8), функции
  `listProducts(site)` (только `id,title,created_at` — колонки для списка),
  `getProduct(site, id)`, `createProduct(site, values)` (insert → select →
  single, вернуть созданную строку), `updateProduct(site, id, values)`,
  `deleteProduct(site, id)`, `listCategoryNames(site)` (`categories.name`,
  order `position`). Всё через `getDb(site)`.
- **`src/features/products/ProductsPage.tsx`** — список (useQuery
  `['products', site.slug]`).
- **`src/features/products/ProductEditPage.tsx`** — одна страница на оба режима
  (edit: useQuery `['products', site.slug, id]` + `['categories', site.slug]`;
  new: без загрузки товара). Мутации create/update/delete → инвалидация
  `['products', site.slug]` + toast (успех/ошибка).
- **Маршруты в `App.tsx`**: `products` → ProductsPage, `products/new` и
  `products/:productId` → ProductEditPage. Пункт меню в SiteLayout уже есть.
- **shadcn**: добавить компоненты `select`, `textarea` (registry v3, radix).

## Ошибки

- Загрузка списка/товара/категорий — красная плашка с текстом ошибки.
- Мутации — `toast.error(e.message)`; RLS-отказ покажется как ошибка Supabase.
- Битая картинка — onError у img → плейсхолдер.

## Проверка

`npm run build`, `npm run lint`; затем ручная e2e под админом:
список → открыть товар → изменить поле → Save → создать товар → удалить его.

## Вне скоупа (следующие итерации)

Пикер картинки из медиа-библиотеки, поиск/фильтр по списку, редактирование slug,
bulk-операции, «used in posts» подсказка при удалении (точная проверка
`post_product_sections.product_ids`).
