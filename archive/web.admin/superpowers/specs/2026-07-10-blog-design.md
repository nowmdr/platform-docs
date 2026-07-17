# Блог (posts) — дизайн раздела админки

Дата: 2026-07-10. Фаза 3, шаг 2 (после products). Утверждён пользователем.
Паттерны переиспользуются из раздела Products — читать `docs/products.md` перед
реализацией. Модель данных — `docs/admin-app-spec.md` §8.

## 1. Цель

Раздел **Blog** в левом меню сайта (`SiteLayout`, вместо заглушки):
- список постов с поиском по заголовку и кнопкой создания;
- редактор поста: все редактируемые поля БД + интерактивный конструктор тела
  поста из модулей (текст / сетка предложенных товаров 2 или 3 колонки) —
  добавить, передвинуть, отредактировать, скрыть, удалить.

## 2. Контракт с данными (схема сайта, менять схему НЕ нужно)

Проверено по живой БД cozycorner (2026-07-10):

- **posts** (12): `id`, `created_at`, `updated_at`, `published_at` (default now),
  `is_published` (default true), `slug` (автоген триггером `posts_set_slug` из
  title, если пуст — как у products: при create не отправлять, при edit
  read-only), `title`, `excerpt?`, `content?` (ЛЕГАСИ — см. ниже),
  `cover_image_path?`, `seo_title?`, `seo_description?`.
- **post_text_sections** (24): `id`, timestamps, `post_id`→posts, `position`,
  `is_published`, `heading?`, `body` (NOT NULL).
- **post_product_sections** (18): `id`, timestamps, `post_id`→posts, `position`,
  `is_published`, `heading?`, `columns` (CHECK 2|3, default 3),
  `product_ids uuid[]` (порядок массива = порядок карточек на сайте).
- **Позиции сквозные между двумя таблицами**: сайт сливает обе таблицы одного
  поста и сортирует по `position` (в данных: текст 0,1 → продукты 2). Значит
  единый список модулей в админке ↔ `position` = индекс модуля в списке.
- FK секций — **ON DELETE CASCADE**: удаление поста чистит секции само.
- `updated_at` — БД-триггерами на всех трёх таблицах, вручную не задавать.
- RLS: чтение постов аноном — только `is_published=true`; запись — `is_admin()`.

**Легаси-поле `posts.content`** (заполнено у 6 старых постов — тестовый текст
первых постов без секций): в админке НЕ показываем и НЕ трогаем. На потом:
удалить колонку миграцией в репозитории сайта cozycorner.

## 3. Слой данных — `src/lib/posts.ts`

Всё через `getDb(site)`. Пустые optional-поля → `null` (`orNull`, как products).

- `PostListItem`: `id, title, published_at, is_published`.
  `listPosts(site)` — лёгкий select, сортировка `published_at desc, title asc`
  (tiebreaker обязателен — паттерн products).
- Модуль — discriminated union (порядок в массиве = position):
  ```ts
  type PostModule =
    | { kind: "text"; id?: string; heading: string | null; body: string;
        isPublished: boolean }
    | { kind: "products"; id?: string; heading: string | null; columns: 2 | 3;
        productIds: string[]; isPublished: boolean };
  ```
  `id` отсутствует у новых (ещё не сохранённых) модулей.
- `getPost(site, id)` → `{ post, modules }`: пост + обе таблицы секций
  (параллельно), слить и отсортировать по `position`.
- `createPost(site, input, modules)`: insert поста (slug не отправлять) →
  insert секций с `position` = индекс.
- `updatePost(site, id, input, modules, loadedModules)` — **diff-сохранение**:
  1) update поста; 2) по каждому модулю формы: без `id` → insert; с `id` →
  update (включая новый `position` = индекс); 3) модули из `loadedModules`,
  отсутствующие в форме, → delete (**удаления в конце**, после
  insert/update). Несколько запросов без транзакции — осознанно (один админ);
  при ошибке — тост, форма остаётся dirty, инвалидация query, пересохранение
  повторит diff корректно.
- `deletePost(site, id)` — только строка поста (каскад уберёт секции).
- Пикер товаров: **расширить `listProducts` в `src/lib/products.ts` полем
  `image_path`** и переиспользовать общий кэш `['products', site.slug]`
  (свой query-ключ не заводить — правило как с `['media', site.slug]`).

Query-ключи: список `['posts', site.slug]` (инвалидация после всех мутаций),
пост `['posts', site.slug, id]`.

## 4. Список — `src/features/posts/PostsPage.tsx`

Маршрут `/:siteSlug/blog`; в `SiteLayout` пункт **Blog** ведёт сюда (убрать
«Coming soon»). Копия паттерна ProductsPage без фильтров:
- одна колонка строк-ссылок: title слева; справа приглушённо дата
  `published_at` + бейдж **Draft**, если `is_published=false`;
- поиск по title (`useDeferredValue`), счётчик `видимые / все` при активном
  поиске, ghost-кнопка Clear;
- кнопка **New post** → `/:siteSlug/blog/new`.

## 5. Редактор — `src/features/posts/PostEditPage.tsx`

Один компонент на create (`blog/new`) и edit (`blog/:postId`). Паттерны
products целиком: форма монтируется после загрузки; RHF + Zod + Field;
гард несохранённых изменений (`useBlocker` + `beforeunload`, условие
`isDirty && !save.isPending && !remove.isPending`, модалка Stay/Discard/Save);
после Save — `form.reset(values)`; после Delete — `form.reset()` и возврат к
списку; при create — redirect на созданный пост (`leavingRef`).

**Поля поста:** Title* · Slug (read-only, только edit) · Excerpt (textarea) ·
Cover image (инпут публичной ссылки + кнопка выбора из Media через существующий
`ImagePickerDialog`; контракт `resolveImageUrl`/`toStoragePath` — в БД плоский
ключ или внешний URL) · **Published** (switch `is_published`) + **Published at**
(редактируемая дата/время `published_at` — сортировка ленты на сайте) · секция
**SEO** с живым фолбэком как у товаров: `seo_title`→title,
`seo_description`→excerpt (пустое/равное фолбэку → `null`; Customize /
Use defaults — та же механика, что в products).

**Конструктор модулей (Content)** — `useFieldArray`, вертикальный стек карточек:
- шапка карточки: тип («Text» / «Products»), контролы справа: видимость
  (глазик ↔ `is_published`; скрытый модуль визуально приглушён), ↑ / ↓
  (disabled на краях), удалить (корзина);
- **Text**: Heading (опциональный инпут) + `RichTextEditor` (существующий,
  markdown как в описании товара);
- **Products**: Heading, переключатель колонок **2 / 3**, выбранные товары
  компактными строками (превью через `resolveImageUrl` + название) со
  стрелками/крестиком — порядок строк = порядок в `product_ids`; кнопка
  **Add products** → новый `ProductPickerDialog` (`src/features/posts/`):
  поиск + мультивыбор из существующих товаров, уже выбранные помечены;
- внизу кнопки **+ Text** и **+ Products** (новый products-модуль — 3 колонки
  по умолчанию, внутри переключается на 2; в БД это один тип секции).

**Валидация (Zod):** title обязателен; text-модуль — `body` непустой (в БД NOT
NULL); products-модуль — минимум 1 товар; `published_at` — валидная дата.
Heading'и опциональны (пустое → `null`).

**Удаление поста** — AlertDialog с предупреждением, что секции удалятся вместе
с постом.

Компоненты редактора разнести по файлам (`TextModuleCard`, `ProductsModuleCard`,
`ProductPickerDialog`) — не повторять монолит на 500+ строк.

## 6. Утверждённые решения (Q&A 2026-07-10)

- `posts.content` — игнорировать; колонку потом удалить миграцией в репо сайта.
- Видимость модуля (`is_published` секции) — да, тоггл на каждой карточке.
- Публикация поста — тоггл `is_published` **и** редактируемая `published_at`.
- Перестановка модулей — стрелки ↑/↓ (без drag-and-drop, без новых зависимостей).
- Сохранение — подход A: одна форма, единый список модулей, diff по Save
  (вместо мгновенных per-module сохранений или атомарного RPC; RPC — возможный
  hardening потом).

## 7. Вне зоны админки / на потом

- Сайт cozycorner: рендер markdown в `post_text_sections.body` — дополнить
  `docs/site-markdown-prompt.md` (старые тексты «абзацы пустой строкой» —
  валидный markdown, фолбэк не нужен).
- Миграция в репо сайта: удалить колонку `posts.content`.
- Правила раздела после реализации зафиксировать в `docs/blog.md`
  (аналог `docs/products.md`).

## 8. Тестирование

Автотестов в репо нет — ручной e2e-чек-лист в плане реализации: создание поста
с модулями всех типов, перестановка, скрытие, удаление модуля и поста, гард
несохранённых изменений, SEO-фолбэк, поиск в списке, Draft-бейдж.
