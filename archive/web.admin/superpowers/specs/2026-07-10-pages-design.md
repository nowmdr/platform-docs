# Раздел Pages — дизайн (утверждено 2026-07-10)

Админ-раздел для редактирования статических страниц сайта: SEO-поля таблицы
`pages` + контент существующих hero-блоков (`hero_sections`). Паттерны —
калька с products/blog (`docs/products.md`, `docs/blog.md`); модель данных —
`docs/admin-app-spec.md` §8.

## Решения по объёму (зафиксированы с пользователем)

- Редактируем **SEO-поля страницы + hero-блоки**.
- Hero-блоки — **только существующие**: правка полей и `is_published`;
  добавления, удаления и перестановки секций НЕТ (у страниц сейчас ≤1 hero;
  набор секций управляется миграциями сайта). Следствие: сохранение героев —
  только `update` существующих строк, diff insert/delete из blog не нужен,
  `position` не трогаем.
- Создания/удаления страниц НЕТ — набор из 7 страниц (`home`, `shop`, `blog`,
  `terms`, `privacy`, `about`, `featured`) задан кодом сайта. Кнопки New и
  Delete отсутствуют.
- `pages.meta` (jsonb) не читаем и не пишем: везде `{}`, контракта с сайтом
  нет. `slug` — read-only (на нём маршруты сайта).

## 1. Слой данных — `src/lib/pages.ts` (паттерн `lib/posts.ts`)

- `listPages(site)` — лёгкий select `id, slug, seo_title` + количество hero
  на страницу (отдельным запросом `hero_sections.select('page_id')` и
  группировкой на клиенте — счётчик для бейджа списка). Сортировка `slug asc`
  (набор фиксированный, дат публикации нет).
- `getPage(site, id)` — страница (явный список колонок, без `meta`) + её
  `hero_sections` (сорт `position asc`), тип `PageWithHeroes`.
- `updatePage(site, id, input)` — update строки `pages`, затем update каждой
  hero-секции по `id` (контентные поля + `is_published`; `position` и
  `page_id` не пишутся), затем свежий `getPage` → возврат для
  пересинхронизации формы (критичный паттерн blog §1: `onSuccess` делает
  `form.reset(toFormValues(fresh))`).
- `updated_at` на обеих таблицах — БД-триггерами, вручную не задавать.
- Пустые optional-поля → `null` (`orNull`, как products): `seo_title`,
  `seo_description`, `og_image_path`, hero `badge`/`description`/
  `bg_image_path`.

## 2. Список — `src/features/pages/PagesPage.tsx`

- Одна колонка строк-ссылок: слева `slug` (имя страницы), справа приглушённо
  `seo_title` и бейдж «N hero» при `heroCount > 0`.
- Без поиска, фильтров и кнопки New — 7 строк.
- Маршруты: `/:siteSlug/pages` (список), `/:siteSlug/pages/:pageId`
  (редактор). Пункт **Pages** в `SiteLayout` ведёт на список (заглушка
  «Coming soon» убирается).
- Query-ключи: список `['pages', site.slug]`, страница
  `['pages', site.slug, id]`; инвалидация списка после Save —
  `exact: true` (паттерн blog, чтобы не сбросить детальный ключ).

## 3. Редактор — `src/features/pages/PageEditPage.tsx` (раскладка v2)

- Только edit (create нет). `PageForm` монтируется после загрузки `getPage`
  — defaultValues один раз, без мигания.
- Sticky-панель действий (`sticky top-0 z-10 bg-background`): слева Back,
  справа Save (`form="page-form"`). Без Delete и тоггла Published.
- Грид-шапка `md:grid-cols-[16rem_1fr]`: слева превью OG image + slug
  (read-only), справа SEO title, SEO description, OG image path + кнопка
  Gallery. Ниже на всю ширину `max-w-3xl` — hero-секции.
- SEO-поля — обычные опциональные инпуты **без** механики Customize/Use
  defaults (у страницы нет собственных title/description для фолбэка).
- OG image — контракт картинок products §1: в БД плоский ключ или внешний
  URL, в инпуте публичная ссылка (`resolveImageUrl` при чтении,
  `toStoragePath(site, value)` при сохранении); пикер — общий
  `ImagePickerDialog` из `features/products/`.
- Невалидный Save — тост `'Fix validation errors before saving'`
  (`handleSubmit(onValid, onInvalid)`).
- Гард несохранённых изменений — паттерн products §4 целиком (`useBlocker` +
  `beforeunload`, условие `isDirty && !save.isPending`, модалка
  Stay/Discard/Save).

## 4. Hero-секции — `src/features/pages/HeroSectionsEditor.tsx`

- Упрощённый PostModulesEditor: `useFieldArray({ name: 'heroes' })`, ключ
  рендера — `field.id`; **id секции в форме — `sectionId`** (useFieldArray
  резервирует `id`).
- Карточка: заголовок «Hero N», глазик `is_published` (`Eye`/`EyeOff`,
  скрытая секция `border-dashed opacity-60`, валидация не ослабляется).
  Кнопок Add/удалить/↑↓ НЕТ.
- Поля карточки: badge (опц. инпут), title (обязательный инпут — `NOT NULL`
  в БД), description (опц. `Textarea` — сайт не рендерит markdown в hero),
  bg image (инпут с публичной ссылкой + Gallery, тот же контракт URL),
  align — переключатель left/right (`aria-pressed`, как columns 2/3 в блоге).
- Если у страницы нет hero-секций — блок с текстом «This page has no hero
  sections» (секция не рендерится пустой формой).
- Валидация Zod: `title` непустой; остальные поля опциональны.

## 5. Ошибки и край-кейсы

- Падение update героев после успешного update страницы — тост об ошибке,
  форма остаётся dirty, повторный Save безопасен (update идемпотентен).
  Транзакции нет — осознанно (один админ), как в blog.
- `bg_image_path`/`og_image_path` с несуществующим ключом бакета — превью
  просто битое, спец-обработки нет (как в products).

## 6. Вне объёма (на потом)

- Add/remove/reorder hero-секций из админки.
- Редактор `pages.meta` (jsonb).
- `footer_settings` — отдельная итерация (фаза «pages/hero/footer»).

## Дополнение (утверждено 2026-07-10, после реализации v1)

**`pages.body` — Markdown-тело страницы, только для `terms` и `privacy`**:
- Колонка `body text` (nullable) добавлена миграцией `add_pages_body`
  (применена через MCP; файл миграции продублировать в репозитории сайта —
  SQL в `docs/site-markdown-prompt.md`).
- Админка: поле Body (`RichTextEditor`, то же markdown-подмножество, что
  products.description) в форме страницы между SEO-шапкой и hero-секциями —
  ТОЛЬКО у страниц из белого списка `BODY_PAGE_SLUGS = ['terms','privacy']`
  (`lib/pages.ts`); у остальных поле скрыто и `body` не входит в payload
  сохранения. Пустое → `null`.
- Контракт с сайтом: `NULL` → страница показывает свой захардкоженный текст;
  заполнено → рендерить markdown из БД. Сайт ещё не обновлён — промпт для его
  агента дописан в `docs/site-markdown-prompt.md`.
