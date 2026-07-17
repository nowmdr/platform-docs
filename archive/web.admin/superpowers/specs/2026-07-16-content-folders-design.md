# Папки в Products и Blog — дизайн (v1)

Дата: 2026-07-16. Статус: утверждено пользователем (вариант A — калька с Media;
поиск сквозной, селекты AND; рефакторинг generic-частей в `src/features/folders/`).
База: папки в Media (спека `docs/superpowers/specs/2026-07-16-media-folders-design.md`,
правила `docs/media-manager.md` §3) — реализованы и проверены пользователем.

## Цель и рамки

Дать админу те же папки в разделах Products и Blog, что уже работают в Media, с
максимальным переиспользованием кода: одна таблица `admin_folders` (поле `section`),
общие компоненты и общая обвязка страницы. Разделы — списки строк, не грид, поэтому
UI адаптируется: чекбокс и кнопка Move встраиваются в строку списка.

Принципы не меняются: папки — организационная сущность админки, сайт о них не знает;
перемещение меняет только `folder_id`; элемент ровно в одной папке; структура плоская.

## 1. БД — миграция `add_content_folders`

```sql
-- Папки для products/posts: та же admin_folders (section 'products' | 'posts'),
-- в контентные таблицы добавляется только folder_id. Сайт колонку не читает.
alter table cozycorner.products
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;
create index products_folder_id_idx on cozycorner.products (folder_id);

alter table cozycorner.posts
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;
create index posts_folder_id_idx on cozycorner.posts (folder_id);
```

- `admin_folders` не меняется (RLS/гранты уже есть).
- FK не запрещает строке товара ссылаться на папку чужой секции — целостность секций
  обеспечивает UI (меню перемещения предлагает только папки своей секции). Осознанное
  упрощение, как и отсутствие enum на `section`.
- Миграция применяется через MCP к base-one; файл продублировать в репозитории сайта
  cozycorner (вместе с `add_admin_folders`, если ещё не сделано).

## 2. Слой данных

- `src/lib/folders.ts`:
  - `FolderSection` → `"media" | "products" | "posts"`;
  - новая `moveToFolder(site, section, ids, folderId)` — generic-перемещение:
    маппинг секция→таблица (`media`/`products`/`posts`, имена совпадают), один
    `update … set folder_id where id in (…)`, no-op на пустом `ids`.
- `src/lib/media.ts`: `moveImages` удаляется (единственный вызов в MediaPage переходит
  на `moveToFolder(site, 'media', …)`; функция наружу не публиковалась).
- `src/lib/products.ts`: `ProductListItem` + `folder_id: string | null`; `listProducts`
  добавляет колонку в select.
- `src/lib/posts.ts`: аналогично для `PostListItem`/`listPosts`.
- `ProductPickerDialog` (блог) и `ImagePickerDialog` не трогаем — поле аддитивное.

## 3. Рефакторинг → `src/features/folders/`

Цель — три страницы без дублей обвязки. Media после рефакторинга ведёт себя
идентично (это регресс-критерий).

- **Переезжают** из `src/features/media/`: `FoldersPanel.tsx`, `MoveToFolderMenu.tsx`
  (импорты в media-файлах обновляются).
- **`FoldersPanel` обобщается** пропсами:
  - `section: FolderSection` — какую секцию CRUD-ить (query-ключ через `foldersKey`);
  - `itemNoun: string` (`'image' | 'product' | 'post'`) — текст delete-диалога
    «N {noun}s will be moved to Unsorted» (единственное/множественное как сейчас);
  - `contentKey` — query-ключ контента раздела, инвалидируется при удалении папки
    (файлы/строки уходят в Unsorted по FK — грид/список должен обновиться).
  - aria-label панели — из секции.
- **Новый хук `useFolders`** (файл `src/features/folders/useFolders.ts`) — вся
  обвязка, сейчас живущая в MediaPage:
  - query папок (`foldersKey`), `foldersError`;
  - состояние фильтра `folder` + производный `effectiveFolder` (fallback на `all`,
    если активная папка удалена в другой сессии);
  - счётчики `{all, unsorted, byFolder}` из переданных `items` (клиент, без запросов);
  - выборка: `selectedIds`, `toggleSelect`, `clearSelection`, сброс при смене папки,
    точечная чистка при удалении элемента;
  - move-мутация: `moveToFolder` + инвалидация `contentKey`, тосты
    «{Noun} moved» / «N {noun}s moved», удаление перемещённых id из выборки;
  - helper пересечения выборки с видимыми элементами (`visibleSelectedIds` —
    bulk-бар оперирует только видимыми выбранными).
  Точная сигнатура — в плане реализации; принцип: страница отдаёт `items` и получает
  всё перечисленное, сама считает только `visibleItems` (семантика поиска/фильтров
  у разделов своя).
- **`MediaPage` переводится на хук** — поведение не меняется, размер файла падает.

## 4. UI Products / Blog (вариант A — калька с Media)

- **Раскладка**: ряд фильтров (поиск + селекты + Clear) остаётся сверху на всю
  ширину; ниже — flex: `<aside>` с `FoldersPanel` (та же ширина `w-44 lg:w-52`) +
  колонка списка (`min-w-0 flex-1`).
- **Строка списка**: `<li>` становится flex-рядом:
  `Checkbox` (сосед ссылки — вложенные интерактивные элементы невалидны) +
  прежний `<Link>` (`flex-1`, вид не меняется) + иконка Move
  (`MoveToFolderMenu` с `currentFolderId` строки).
- **Bulk-бар** над списком при непустой видимой выборке: «N selected · Move to… ·
  Clear» — как в media.
- **Семантика фильтров**:
  - поиск — **сквозь все папки** (как в media: «нашёл — увидел»);
  - селекты (Brand/Category у products, Status у blog) — **AND с активной папкой**;
  - кнопка Clear чистит поиск и селекты, папку не трогает.
- **Empty state**: «This folder is empty.» — когда список пуст из-за папки и других
  фильтров нет; иначе прежние «No … match your filters.» / «No … yet.»
- **Счётчик в шапке** (`visible/total`) остаётся как есть (total — весь раздел).
- **Создание**: новые товары/посты попадают в Unsorted; выбора папки в редакторах
  и формах создания нет — перемещение только из списка (YAGNI).

## 5. Ошибки и краевые случаи

- Все паттерны из media сохраняются: дубль имени папки → тост «already exists»,
  пустое имя запрещено, ошибка загрузки папок не блокирует список (панель без
  скелетонов, отдельный текст ошибки), `effectiveFolder` против мёртвого id.
- Регресс-риск рефакторинга: media должен вести себя в точности как до него —
  проверяется по чек-листу media (сокращённому) в ручном e2e.

## 6. Проверка

- `npm run lint`, `npm run build`.
- Ручной e2e-чек-лист (войдёт в план): CRUD папок в products и blog, move одного и
  пачки, семантика поиск-сквозной/селекты-AND, «This folder is empty.», создание →
  Unsorted, регресс media после рефакторинга (панель/move/bulk/upload-в-папку),
  пикеры (ProductPickerDialog, ImagePickerDialog) работают как раньше.

## Вне скоупа

Drag&drop, вложенность, выбор папки в редакторах/при создании, bulk-операции кроме
move, папки для pages (набор страниц фиксирован — папки не нужны).
