# Раздел Media — правила и концепции (+ папки всех разделов)

> Last updated: 2026-07-17 | Source project: web.admin (docs/media-manager.md) — пути `src/…` относятся к репозиторию `web.admin/`

Документ фиксирует, как устроен раздел Media админки: контракт с данными, поведение
операций и решения по UI. Базовый контракт — [api.md](api.md) §4; здесь — его
актуальная реализация со всеми уточнениями, принятыми в ходе разработки
(2026-07-08/09, папки — 2026-07-16). История решений —
`../archive/web.admin/superpowers/specs/2026-07-08-media-manager-design.md` и
`…/2026-07-16-media-folders-design.md`. Читать перед правками раздела.

## 1. Главные концепции

- **Источник списка — таблица `<schema>.media`**, не Storage. Каждая строка описывает
  один загруженный файл: `id, created_at, original_name, path, mime, size, folder_id`.
  Таблица и бакет держатся в синхроне кодом админки (см. «Загрузка» и «Удаление»).
  `folder_id` — см. §3 «Папки».
- **Один бакет на сайт** (для CozyCorner — `cozycorner-photos`), пути **плоские**:
  ключ объекта = имя файла, без папок. Старые бакеты проекта (`photos`,
  `product-images`) — наследие прежней встроенной админки, менеджером не используются.
- **В БД хранится относительный путь** (`product-1.png`), никогда — абсолютный URL
  нашего Storage. Публичный URL строится на лету:
  `${projectUrl}/storage/v1/object/public/${bucket}/${path}`
  (`src/lib/images.ts :: resolveImageUrl`). Внешние URL (`http(s)://…`) валидны и
  возвращаются как есть. Поэтому переезд на другой Supabase-проект = смена `projectUrl`
  в реестре, ссылки в данных не бьются.
- **Медиа-менеджер показывает только загруженные файлы.** Картинки сайта, заданные
  внешними ссылками (Unsplash и т.п. — сейчас это большинство полей `*image_path*`),
  в гриде не появляются: они не лежат в нашем Storage.
- **Вся запись — только под сессией админа** (RLS + `public.is_admin()`). У `media`
  нет и публичного чтения: анониму список пуст. Service-role ключа в админке нет.

## 2. Операции (`src/lib/media.ts`)

### Загрузка `uploadImage(site, file, folderId = null)`
1. Валидация: mime ∈ {jpeg, png, webp, gif}, размер ≤ 6 МБ. Невалидный файл
   отклоняется до обращения к сети.
2. Ключ: `sanitize(file.name)` — нижний регистр, `[^a-z0-9._-] → -`, схлопывание и
   обрезка дефисов. Коллизии решаются суффиксом `-2`, `-3`, … (занятость проверяется
   по `media.path`).
3. `storage.upload(key, file, { upsert: false })` → insert в `media`
   (`original_name` — исходное имя файла, `path` — ключ, `folder_id` — куда класть
   файл, `null` = Unsorted).
4. **Откат:** если insert упал — загруженный объект удаляется из Storage,
   «сироты» не остаются.
5. Мульти-выбор грузится **последовательно**; ошибка одного файла не прерывает
   остальные (в конце — тост об успехах и отдельные тосты об ошибках).

### Список `listImages(site)`
- `media` сортируется `created_at desc` — свежие первыми.
- Для каждого файла собирается `usedBy` — где путь встречается в данных сайта.
  Источники (прямое равенство `path`): `products.image_path`,
  `categories.image_path|hero_image_path`, `posts.cover_image_path`,
  `hero_sections.bg_image_path`, `pages.og_image_path`.
  При добавлении новых полей картинок в схему сайта — дополнить `USAGE_SOURCES`.

### Переименование `renameImage(site, id, newName)`
- Меняется **только** отображаемое имя `media.original_name`. Ключ в Storage, `path`
  и публичный URL не трогаются — ссылки на сайте не ломаются. Пустое имя запрещено.

### Удаление `deleteImage(site, item)`
- Порядок: объект из Storage → строка из `media`.
- Если картинка используется — confirm-диалог **предупреждает, но не блокирует**
  (осознанный выбор; на сайте картинка станет битой).

### Перемещение `moveToFolder(site, 'media', ids, folderId)` (`src/lib/folders.ts`)
- Generic-функция для всех секций (бывшая `moveImages` удалена). Меняет **только**
  `folder_id` у переданных id (`in`-update, маппинг секция→таблица). Для media это
  значит: `path`, ключ в Storage и публичный URL не трогаются — ссылки на сайте не
  бьются никогда. `folderId = null` — переместить в Unsorted.

## 3. Папки (`src/lib/folders.ts`) — канонические правила для ВСЕХ разделов

Спеки: `../archive/web.admin/superpowers/specs/2026-07-16-{media,content}-folders-design.md`.

- **Папки — организационная сущность самой админки**, не сайта: сайт таблицу
  `admin_folders` не читает и о папках не знает. Файл, лежащий в папке, отдаётся сайту
  тем же `path`/URL, что и без папки.
- **Папки работают в разделах** media / products / posts / seo_posts — секции в
  `admin_folders`, у каждой контентной таблицы своя колонка `folder_id` (миграции
  `0021_admin_folders.sql` для media и `0022_content_folders.sql` для products/posts
  в `cozycorner/supabase/migrations/`; применены 2026-07-16 через MCP под именами
  `add_admin_folders`/`add_content_folders`, см. [schema.md](../database/schema.md)
  §6). У products/posts есть публичное чтение
  (сайт) — `folder_id` виден anon-выборкам; это просто uuid без чувствительных
  данных, принято осознанно.
- **Хранение**: одна generic-таблица `<schema>.admin_folders` (`id, created_at, section,
  name`, `unique(section, name)`, RLS admin-only — как у `media`) на все разделы админки;
  раздел различает колонка `section`. В контентной таблице — nullable
  `folder_id uuid references admin_folders(id) on delete set null`
  + индекс: удаление папки не удаляет элементы, они становятся Unsorted через FK,
  без кода в приложении.
- **Операции** (`src/lib/folders.ts`):
  - `foldersKey(site, section)` — общий query-key `['folders', site.slug, section]`;
    использовать и на странице раздела, и в `FoldersPanel`, чтобы инвалидация/рефетч
    совпадали.
  - `listFolders(site, section)` — сортировка по `name` (алфавит).
  - `createFolder`/`renameFolder` — тримят имя, пустое имя — ошибка до запроса;
    уникальность `(section, name)` ловится по коду `23505` и превращается в «Folder
    with this name already exists».
  - `deleteFolder` — удаляет только строку папки; элементы отвязываются каскадом FK
    (`on delete set null`), их данные не трогаются.
  - `moveToFolder(site, section, ids, folderId)` — generic-перемещение (см. §2).
- **UI** (`src/features/folders/` — общий код разделов):
  - `FoldersPanel` — generic левая панель (пропсы `section`/`itemNoun`/`contentKey`):
    пункты All / Unsorted / папки (алфавитный порядок), счётчики считаются на клиенте
    из уже загруженного списка раздела (доп. запросов нет). Инлайн-создание
    («New folder») и переименование (Enter/Esc, Check/X — как rename картинки в
    `MediaDetailsDialog`); по ховеру на папке — меню «…» (Rename/Delete). Delete —
    `AlertDialog`: «N {itemNoun}s will be moved to Unsorted» (или «The folder is
    empty»), сами элементы не удаляются; после delete инвалидируется `contentKey`
    раздела. Мутации CRUD папок живут в `FoldersPanel`; move — нет (см. ниже).
  - `MoveToFolderMenu` — общее dropdown-меню «Move to…», переиспользуется на карточке,
    в модалке деталей, в строках списков products/posts и в bulk-барах. Текущая папка
    элемента исключается из целей; `currentFolderId` не передаётся из bulk-бара
    (выборка может быть из разных папок — показываются все цели, включая Unsorted).
  - **Хук `useFolders`** (`useFolders.ts`) — вся обвязка страницы раздела: query папок
    (+`foldersError`), фильтр `folder` с fallback на All (effectiveFolder), счётчики,
    `matchesFolder`, выборка (`selectedIds`/`toggleSelect`/`clearSelection`/
    `removeFromSelection`), move-мутация с тостами и точечной чисткой выборки,
    `activeFolderId` для upload-в-папку. Страница сама считает `visibleItems`
    (семантика поиска/фильтров у разделов своя) и пересекает выборку хелпером
    `intersectSelected`.
  - **Multi-select**: чекбокс всегда виден (на превью карточки в media / в строке
    списка в products и blog; сосед интерактивного элемента, не вложен в него —
    вложенные интерактивные элементы невалидны). Bulk-бар оперирует только
    `visibleSelectedIds` — пересечением выборки с видимыми (после фильтра/поиска)
    элементами, чтобы скрытые чекбоксы не участвовали в перемещении незаметно.
    Одиночный move убирает из выборки только перемещённые id; смена активной папки
    и удаление элемента сбрасывают/чистят выборку.
  - **Upload** (media) идёт в открытую сейчас папку (`All`/`Unsorted` →
    `folder_id = null`); новые товары/посты всегда попадают в Unsorted (выбора папки
    в редакторах нет — YAGNI).
  - **Поиск** — сквозь все папки: фильтр по активной папке применяется только когда
    строка поиска пустая. Селекты products/blog (Brand/Category/Status) — AND с
    активной папкой; Clear чистит поиск/селекты, папку не трогает.
  - **effectiveFolder** — если активная папка была удалена в другой сессии (второй
    админ), страница после рефетча ведёт себя как `All`, не оставляя список на
    мёртвом id.
  - `ImagePickerDialog` (пикер картинок для товаров/постов) **папок не знает**:
    загрузка из пикера всегда идёт в Unsorted (`uploadImage(site, file)` без третьего
    аргумента), никакого folder-UI там нет — сознательное решение, не баг.

## 4. UI (`src/features/media/`)

Полная UX/UI-структура раздела — [components.md](components.md) §3.6. Ключевое:
грид 2/3/4 колонки, карточки одной высоты; маркеры на превью («не используется» /
«>1 МБ»); модалка деталей с rename/габаритами/Used by; deferred-поиск; bulk-бар;
query-ключи `['media', site.slug]` + `foldersKey(site, 'media')` (delete папки
инвалидирует оба); курсор pointer для кнопок задан глобально в `src/index.css`.

## 5. Правила для будущих фаз (CRUD контента)

- При выборе картинки для товара/категории/поста в поле БД писать **относительный
  `path`** из `media`, не публичный URL. Пикер картинок должен опираться на
  `listImages`.
- Загрузку переиспользовать из `uploadImage` — та же валидация и контракт путей.
- Новые поля `*image_path*` в схеме сайта → добавить в `USAGE_SOURCES`
  (`src/lib/media.ts`), чтобы «used by» и предупреждения при удалении оставались
  точными.
- «Path in bucket» сознательно не показывается в UI (публичный URL его содержит);
  если для CRUD понадобится копировать относительный путь — добавлять пикером,
  а не текстом в модалке.
- Папки для новых разделов — переиспользовать `admin_folders`/`src/lib/folders.ts`/
  `useFolders` (новая `section`, своя колонка `folder_id` отдельной миграцией; §3),
  а не заводить отдельную таблицу.
