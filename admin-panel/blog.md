# Раздел Blog / SEO Posts — правила и концепции

> Last updated: 2026-07-17 | Source project: web.admin (docs/blog.md) — пути `src/…` относятся к репозиторию `web.admin/`

Документ фиксирует, как устроен раздел Blog админки: контракты с данными,
поведение операций и решения по UI, принятые в ходе разработки (2026-07-10;
SEO Posts — 2026-07-16). Модель данных — [../database/schema.md](../database/schema.md) §5;
дизайн — спеки `../archive/web.admin/superpowers/specs/2026-07-10-blog-design.md` и
`…/2026-07-16-seo-posts-design.md`; планы — `../archive/web.admin/superpowers/plans/`.
Читать этот файл ПЕРЕД любыми правками раздела.

## 1. Контракты с данными

- **Слой данных — `src/lib/posts.ts`**: `listPosts` (лёгкий select
  `id,title,published_at,is_published,folder_id` для списка; сортировка `published_at
  desc` + tiebreaker `title asc` — паттерн products, у постов с совпадающими
  датами иначе нестабильный порядок), `getPost` (пост + обе таблицы секций
  параллельно, слияние в единый `PostModule[]` по `position`),
  `createPost`/`updatePost`/`deletePost`. Всё через `getDb(site)`.
- **Модуль ↔ две таблицы**: `PostModule` — discriminated union
  `kind: "text" | "products"`, ↔ `post_text_sections` / `post_product_sections`.
  **Позиции сквозные между таблицами**: `position` = индекс модуля в едином
  массиве формы; сайт сливает обе таблицы и сортирует по `position`.
- **Diff-сохранение** (`updatePost` → внутренний `syncModules`/`syncTable`):
  для каждой из двух таблиц секций — insert модулей без `id` → update всех
  существующих (контент И `position` пишутся всегда, т.к. оба могли
  измениться) → delete пропавших строго в конце (после insert/update).
  Несколько запросов без транзакции — осознанно (один админ-пользователь);
  при падении — тост, форма остаётся dirty. Повторный Save безопасен для
  update/delete (идемпотентны), но НЕ для insert: если падение случилось после
  части insert'ов новых секций, у них в форме ещё нет `sectionId`, и retry
  вставит их повторно — дубли придётся удалить руками (ещё один аргумент за
  RPC-атомарность из §5). Diff считается против снапшота загрузки (`loadedModulesRef`), НЕ
  против текущего состояния БД — конкурентная админ-сессия может создать
  секции, которые diff не увидит (принято как риск; RPC-атомарность —
  возможный hardening на потом).
- **Пересинхронизация после Save — КРИТИЧНО**: `updatePost` возвращает
  свежие пост+модули (`getPost` после записи, `PostWithModules`); `onSuccess`
  в форме обновляет `loadedModulesRef.current = fresh.modules` и делает
  `form.reset(toFormValues(site, fresh))` — иначе у только что вставленных
  модулей нет `id` в form state, и повторный Save продублирует их как новые
  секции. Инвалидация списка — `queryClient.invalidateQueries({ queryKey:
  ['posts', site.slug], exact: true })`, чтобы не сбросить только что
  положенный `setQueryData(['posts', site.slug, id], fresh)` детальный ключ.
- `createPost`: insert поста (slug не отправляется — генерирует БД-триггер
  `posts_set_slug`) → insert секций с `position` = индекс; при падении секций
  пост откатывается `delete` (паттерн `uploadImage`/products — не оставлять
  пост-огрызок без содержимого).
- `deletePost`: удаляется только строка поста, секции убирает FK
  `ON DELETE CASCADE` на обеих таблицах.
- **Легаси `posts.content` НЕ читаем и НЕ пишем**: `POST_COLUMNS` — явный
  список колонок (не `*`), чтобы случайно не потащить легаси-поле. На потом:
  удалить колонку миграцией в репозитории сайта cozycorner.
- Пустые optional-поля → `null` (`orNull`, как products): `excerpt`,
  `cover_image_path`, module `heading`. `updated_at` на всех трёх таблицах —
  БД-триггерами, вручную не задавать.

## 1a. Типы постов — Blog / SEO Posts (2026-07-16)

- Колонка `posts.post_type` (`'blog'` | `'seo'`, default `'blog'`; миграция
  `add_seo_post_type` — в репо сайта продублирована как `0020_posts_post_type.sql`).
  SEO-посты — programmatic SEO: не выводятся в ленте блога на сайте, но доступны
  по прямому URL и в sitemap. **Никакого cloaking'а**: редиректы/User-Agent-логика/
  noindex для них запрещены.
- Разделы Blog и SEO Posts рендерят **одни и те же** `PostsPage`/`PostEditPage`;
  различия — только конфиг-объект `PostsSectionConfig`
  (`src/features/posts/sections.ts`: `postType`, `basePath` blog/seo-posts,
  `folderSection` posts/seo_posts, лейблы). Маршруты SEO Posts —
  `/:siteSlug/seo-posts{,/new,/:postId}`.
- `listPosts(site, postType)` фильтрует по типу; query-ключи включают тип:
  список `['posts', site.slug, postType]`, пост `['posts', site.slug,
  postType, id]`. `toInput` проставляет `post_type` из конфига; форма тип
  никогда не меняет — **перенос поста между Blog и SEO Posts не
  поддерживается**.
- Папки SEO Posts — отдельная секция `'seo_posts'` в `admin_folders`
  (в `SECTION_TABLES` она указывает на таблицу `posts` — имя секции ≠ таблице).
- SEO-посты обычно создаёт агент claude.ai через Supabase-коннектор —
  инструкция для него: `seo-posts-skill.md` (контракт insert'а +
  SEO-чек-лист; загрузить в проект claude.ai — см. [status.md](status.md)).

## 2. Список (`src/features/posts/PostsPage.tsx`)

Одна колонка строк-ссылок — title слева, справа приглушённо Draft-бейдж (если
`is_published=false`) + `formatDate(published_at)`. Поиск по title
(`useDeferredValue`) + Select-фильтр статуса (All statuses/Published/Draft,
сентинел `__all__`, как `__none__` в products) — условия по AND; Clear
сбрасывает и поиск, и фильтр; счётчик `видимые / все` при активном
поиске/фильтре (паттерн [products.md](products.md) §2). Кнопка **New post** →
`/:siteSlug/blog/new`. Маршруты: `/:siteSlug/blog`, `/blog/new`, `/blog/:postId`.

### Папки

- Слева от списка — панель папок (`FoldersPanel`), в строках — чекбоксы
  multi-select и иконка Move (`MoveToFolderMenu`), над списком — bulk-бар
  «N selected · Move to… · Delete · Clear» (delete — `BulkDeleteButton` +
  `useBulkDelete`, confirm-диалог). Канонические правила и контракты папок —
  [media.md](media.md) §3 (общий код — `src/features/folders/`, хук
  `useFolders`, секция `'posts'` в `admin_folders`).
- `folder_id` есть **ТОЛЬКО** в списочном типе `PostListItem` — в
  `Post`/`PostInput` его нет: редактор и payload'ы create/update поле не
  трогают, новые посты попадают в Unsorted.
- Семантика фильтров: поиск — сквозь все папки; селект Status — AND с активной
  папкой; Clear чистит поиск/статус, папку не сбрасывает. Пустая папка без
  прочих фильтров — «This folder is empty.»

## 3. Форма (`src/features/posts/PostEditPage.tsx`)

- Один компонент `PostForm` на create (`blog/new`) и edit
  (`blog/:postId`); монтируется только после загрузки данных (`getPost`) —
  defaultValues задаются один раз, «мигания» формы нет. RHF + Zod v4 + Field,
  Select/Switch — через Controller.
- **Slug** — read-only при edit, при create не показывается (генерируется
  триггером, как у products).
- **Cover image** — контракт как у products: в БД плоский ключ бакета или
  внешний URL, в UI-инпуте всегда публичная ссылка (`resolveImageUrl` при
  чтении, `toStoragePath(site, value)` при сохранении); пикер — общий
  `ImagePickerDialog` из `features/products/` (переиспользован как есть, не
  задублирован).
- **Published** (`Switch` на `is_published`) + **Published at**
  (`type="datetime-local"`, `published_at`) — сортировка ленты на сайте. При
  каждом Save секунды обрезаются до минут: `toDatetimeLocalValue` в
  `lib/format.ts` форматирует под `datetime-local`, при сабмите
  `new Date(value).toISOString()` уже не восстанавливает секунды исходного
  значения из БД.
- **SEO** — тот же живой фолбэк, что в products: `seo_title` → `title`,
  `seo_description` → `excerpt`, пустое или совпадающее с фолбэком значение
  пишется как `null`; секция свёрнута при пустых полях (Customize /
  Use defaults — идентичная механика, `shouldDirty` только на Use defaults).
- Невалидный Save не немой: `form.handleSubmit(onValid, onInvalid)` с
  `onInvalid` → `toast.error('Fix validation errors before saving')`.
- **Раскладка v2** (спека/план `../archive/web.admin/superpowers/{specs,plans}/2026-07-10-blog-v2-layout*`):
  sticky-панель действий сверху (`sticky top-0 z-10 bg-background` — работает,
  т.к. шапка `AppShell` не sticky) — слева Back, справа Switch
  Published/Draft, Save, Delete. Save вынесен из `<form>`: сабмит через
  `id="post-form"` на форме + атрибут `form="post-form"` на кнопке. Грид-шапка
  `md:grid-cols-[16rem_1fr]` — превью cover + slug слева, Title/Excerpt/Cover
  image справа; остальные поля (Published at, модули, SEO) ниже на всю
  ширину `max-w-3xl`. Поле Status в теле формы удалено — тоггл только в
  sticky-панели.
- Удаление — `AlertDialog` с предупреждением, что секции удалятся вместе с
  постом (в БД это гарантирует каскад).
- Гард несохранённых изменений — паттерн [products.md](products.md) §4 без
  изменений (`useBlocker` + `beforeunload`, условие `isDirty && !save.isPending &&
  !remove.isPending`, модалка Stay/Discard/Save).

## 4. Конструктор модулей (`PostModulesEditor.tsx` + `ProductPickerDialog.tsx`)

- `useFieldArray({ name: 'modules' })`; ключ рендера карточки — `field.id`
  (стабильный ключ useFieldArray), поэтому контент `RichTextEditor` (TipTap,
  внутреннее состояние редактора) корректно переезжает вместе с карточкой
  при `move`, а не пересоздаётся.
- **Поле id секции в форме называется `sectionId`**, не `id`: useFieldArray
  резервирует ключ `id` под собственный рендер-идентификатор и перезаписал
  бы значение из БД.
- Контролы карточки: глазик — `is_published` (`Eye`/`EyeOff`, скрытый модуль
  визуально приглушён — `border-dashed opacity-60`, но **валидацию проходит
  как обычно**: скрыть пустой text/products-модуль без контента/товаров
  нельзя, схема не делает исключения для `isPublished=false`); ↑/↓ (`move`,
  `disabled` на краях массива); корзина (`remove`).
- **Text-модуль**: `heading` (опциональный инпут) + `RichTextEditor`
  (markdown, тот же generic-компонент, что в products description).
- **Products-модуль**: `heading`, переключатель колонок 2/3
  (`aria-pressed`, дефолт при добавлении — 3, переключается на 2 в самой
  карточке — в БД это один тип секции с полем `columns`), строки товаров —
  порядок строк = порядок `productIds` = порядок карточек на сайте.
  Каждая строка — превью (`resolveImageUrl` по `image_path` из
  `listProducts`, который расширен этим полем специально для блога) +
  название + ↑/↓/✕. Удалённый (из products) товар рендерится как строка
  **«Unknown product (deleted?)»** — сам `id` при этом сохраняется в
  `productIds` до явного удаления пользователем из списка (не автоочищается
  молча).
- Пикер — **один общий `ProductPickerDialog`** на все products-модули
  формы: состояние `pickerFor: number | null` — индекс модуля, для которого
  открыт диалог; radix `Dialog` размонтирует контент при закрытии, поэтому
  смена `pickerFor` между открытиями не даёт stale-индекса. Диалог: поиск +
  мультивыбор (черновой `selected` сбрасывается при каждом открытии), уже
  добавленные в модуль товары скрыты (`excludeIds`), кнопка Add отдаёт
  выбранные id пачкой через `onAdd`.
- Дедупликация id товаров — **на Add** (`onAdd` в `PostModulesEditor`
  добавляет к текущему списку через простой конкат, но `excludeIds` уже
  исключает добавленные — дубликат через пикер невозможен) **и при загрузке
  из БД** (`toModuleFormValue`: `[...new Set(m.productIds)]` — на случай,
  если дубли всё же оказались в БД не через админку).
- Валидация Zod: text-модуль — `body` непустой (в БД `NOT NULL`);
  products-модуль — `productIds.min(1)`.

## 5. Известные хвосты / на потом

- Удалить легаси-колонку `posts.content` миграцией в репозитории сайта.
- RPC-атомарное сохранение поста (пост + все секции одной транзакцией) —
  возможный hardening взамен текущего diff без транзакции.
- Нет scroll-to-error при невалидном Save — только тост + инлайн-подсветка
  полей (как в products).
- Ручная e2e-проверка раздела под админом ещё не выполнена — чек-лист
  Task 6.1 плана `../archive/web.admin/superpowers/plans/2026-07-10-blog.md`
  (см. [status.md](status.md)).
