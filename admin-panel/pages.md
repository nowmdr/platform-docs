# Раздел Pages — правила и концепции

> Last updated: 2026-07-23 | Source project: web.admin (docs/pages.md) — пути `src/…` относятся к репозиторию `web.admin/`

Документ фиксирует, как устроен раздел Pages админки: контракты с данными,
поведение операций и решения по UI (2026-07-10). Модель данных —
[../database/schema.md](../database/schema.md) §5; дизайн — спека
`../archive/web.admin/superpowers/specs/2026-07-10-pages-design.md`; план —
`../archive/web.admin/superpowers/plans/2026-07-10-pages.md`. Читать этот файл ПЕРЕД
любыми правками раздела.

## 1. Контракты с данными

- **Слой данных — `src/lib/pages.ts`**: `listPages` (лёгкий select
  `id,slug,seo_title` + счётчик hero отдельным запросом `hero_sections
  .select('page_id')` с группировкой на клиенте; сортировка `slug asc` —
  набор фиксированный), `getPage` (страница + hero-секции параллельно, hero
  сорт `position asc`), `updatePage`. Всё через `getDb(site)`.
- **Create/delete страниц НЕТ** — набор из 7 страниц (`home`,`shop`,`blog`,
  `terms`,`privacy`,`about`,`featured`) задан кодом сайта. `slug` read-only
  (на нём маршруты сайта).
- **`pages.meta` (jsonb) НЕ читаем и НЕ пишем** — везде `{}`, контракта с
  сайтом нет; `PAGE_COLUMNS` — явный список колонок (не `*`), чтобы meta не
  утёк в форму.
- **`pages.body` — Markdown-тело страницы, ТОЛЬКО для `terms` и `privacy`**
  (белый список `BODY_PAGE_SLUGS` в `lib/pages.ts`; колонка добавлена
  2026-07-10 миграцией `add_pages_body`, в репо сайта продублирована как
  `0019_pages_body.sql`). У остальных страниц поле Body не
  показывается и `body` НЕ попадает в payload сохранения (`PageInput.body`
  optional) — не затирать данные. То же подмножество markdown, что
  `products.description`; редактор — общий `RichTextEditor`. Пустое → `null`
  → сайт показывает свой встроенный текст (безопасный фолбэк). Сайт рендерит
  `body` через `LegalArticle` + `MarkdownText` (выполнено — см.
  [../sites/cozycorner.md](../sites/cozycorner.md)). Расширение на другие
  страницы — добавить slug в `BODY_PAGE_SLUGS` (и научить сайт рендерить body
  на этой странице).
- **Hero-секции: только update существующих** (решение спеки): add/remove/
  reorder из админки нет — набор секций управляется миграциями сайта.
  Следствия: `HeroSection.id` обязателен, `position` и `page_id` не читаем
  для записи и не пишем; diff insert/delete из blog не нужен. `updatePage`
  делает update `pages` → update каждого hero по `id` → свежий `getPage` для
  пересинхронизации формы (`onSuccess` → `form.reset(toFormValues(fresh))`).
  Транзакции нет — осознанно; update идемпотентен, повторный Save после
  падения безопасен (в отличие от insert в blog).
- Пустые optional-поля → `null` (`orNull`): `seo_title`, `seo_description`,
  `og_image_path`, hero `badge`/`description`/`bg_image_path`. `updated_at`
  обеих таблиц — БД-триггерами.
- Query-ключи: список `['pages', site.slug]`, страница
  `['pages', site.slug, id]`; после Save детальный ключ кладётся
  `setQueryData`, список инвалидируется с `exact: true`.

## 2. Список (`src/features/pages/PagesPage.tsx`)

Одна колонка строк-ссылок: `slug` слева, справа приглушённо бейдж «N hero»
(если есть) + `seo_title`. Без поиска, фильтров и кнопки New — 7 строк.
Маршруты: `/:siteSlug/pages`, `/pages/:pageId`; пункт **Pages** в `SiteLayout`.

Сверху списка закреплена отдельная строка **«Header»** (бейдж `site`) — это НЕ
строка таблицы `pages`, а сайт-глобальные настройки шапки в своей singleton-таблице
`header_settings` (см. §6). Рендерится всегда, не зависит от загрузки списка страниц.

## 3. Форма (`src/features/pages/PageEditPage.tsx`)

- Только edit — `PageForm` монтируется после загрузки `getPage`.
- Раскладка v2 ([blog.md](blog.md) §3) минус лишнее: sticky-панель — Back + Save
  (`form="page-form"`); **без Delete и тоггла Published** (у страницы их
  нет), без `leavingRef` (после Save нет редиректа).
- Грид-шапка `md:grid-cols-[16rem_1fr]`: слева превью OG image + slug,
  справа SEO title / SEO description / OG image + Gallery. Ниже — Body
  (`RichTextEditor`, markdown; только у страниц из `BODY_PAGE_SLUGS` —
  terms/privacy; подсказка «Leave empty to keep the text built into the
  site»), затем hero-секции.
- **SEO-поля без механики Customize/Use defaults** — у страницы нет
  собственных title/description для фолбэка; пустое поле = `null` = сайт сам
  решает, что показывать.
- OG image — контракт картинок [products.md](products.md) §1: в БД плоский ключ
  или внешний URL, в инпуте публичная ссылка (`resolveImageUrl`/`toStoragePath`);
  пикер — общий `ImagePickerDialog` из `features/products/`.
- Невалидный Save — тост `'Fix validation errors before saving'`.
- Гард несохранённых изменений — паттерн [products.md](products.md) §4
  (`useBlocker` + `beforeunload`, условие `isDirty && !save.isPending`).

## 4. Hero-секции (`src/features/pages/HeroSectionsEditor.tsx`)

- Упрощённый `PostModulesEditor`: `useFieldArray({ name: 'heroes' })` только
  ради стабильных ключей рендера (`fields` без append/remove/move); **id
  секции в форме — `sectionId`** (useFieldArray резервирует `id`).
- Карточка «Hero N»: глазик `is_published` (скрытая — `border-dashed
  opacity-60` + «(hidden)», валидация не ослабляется); поля badge (опц.),
  title (обязателен — `NOT NULL` в БД), description (опц. Textarea — сайт
  не рендерит markdown в hero), bg image (публичная ссылка + Gallery),
  align — тоггл left/right (`aria-pressed`, как columns 2/3 в blog).
- Пикер картинок — **один общий на все hero-карточки**: `pickerFor: number |
  null` — индекс секции (паттерн `ProductPickerDialog` из [blog.md](blog.md) §4).
- Если у страницы нет hero — дашед-блок «This page has no hero sections».

## 5. Вне объёма / на потом

- Add/remove/reorder hero-секций из админки.
- Редактор `pages.meta` (jsonb).
- `footer_settings` — отдельная итерация (фаза «pages/hero/footer»).
- Ручная e2e-проверка раздела — чек-лист Task 5.3 плана
  `../archive/web.admin/superpowers/plans/2026-07-10-pages.md` (см. [status.md](status.md)).

## 6. Header — сайт-глобальные настройки шапки

Отдельный экран, открывается из списка Pages (строка «Header» сверху), но живёт в
своей таблице `header_settings` ([schema.md](../database/schema.md) §5, миграция 0024
репо cozycorner) — не в `pages`. Пока одно поле — **Instagram URL** (`instagram_url`);
таблица заведена с прицелом на рост (новые параметры шапки = новые колонки + строки формы).

- **Слой данных — `src/lib/header.ts`**: `getHeaderSettings` (первая singleton-строка,
  явные колонки `id,instagram_url`), `updateHeaderSettings` (update по `id` → свежая
  строка для пересинхронизации формы). Query-ключ `['header', site.slug]`.
- **Форма — `src/features/pages/HeaderEditPage.tsx`**: sticky Back + Save (`form=
  "header-form"`), без Delete/Published. Поле Instagram — optional: пусто → `null` →
  на сайте иконка скрыта; непустое валидируется как `http(s)://…` (zod refine).
  Гард несохранённых изменений — общий паттерн ([products.md](products.md) §4).
- **Маршрут** — `/:siteSlug/pages/header` (статический сегмент важнее динамического
  `pages/:pageId`, коллизии нет). `HeaderEditPage` в `App.tsx` перед `pages/:pageId`.
- Сайт читает `header_settings` через `fetchHeaderSettings` (`lib/content.ts`) и
  прокидывает `instagram_url` в `<Header>` из `app/layout.tsx`.
