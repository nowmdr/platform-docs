# Папки в Media — дизайн (v1)

Дата: 2026-07-16. Статус: утверждено пользователем (вариант A: панель папок + Move to,
плоские папки, элемент ровно в одной папке).

## Цель и рамки

Дать админу возможность упорядочивать картинки по папкам **внутри админки**. Папки —
чисто организационная сущность админ-панели: сайт cozycorner о них не знает, контракт
фронта не меняется. Начинаем с раздела Media; после проверки логики тот же механизм
распространим на posts/products (отдельные итерации).

Ключевое следствие: **папка виртуальна** — это `folder_id` в БД, а не префикс пути.
Перемещение файла между папками не трогает `media.path`, ключ в Storage и публичный
URL; ссылки на сайте не ломаются. Пути в бакете остаются плоскими.

## 1. Модель данных

Одна generic-таблица на схему сайта + колонка в `media`:

```sql
create table cozycorner.admin_folders (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  section text not null,            -- 'media'; позже 'posts' | 'products'
  name text not null,
  unique (section, name)
);

alter table cozycorner.media
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;
```

- Плоская структура: без `parent_id`. Вложенность и ручной порядок (`position`) —
  сознательно вне скоупа; при необходимости добавляются миграцией позже.
- Сортировка папок в UI — по алфавиту (`order by name`).
- `on delete set null`: удаление папки никогда не удаляет и не «теряет» файлы — они
  становятся «Unsorted» (folder_id = null).
- RLS на `admin_folders` — как у `media`: все операции (select/insert/update/delete)
  только под `public.is_admin()`; публичного чтения нет (анониму — 0 строк).
- Миграция применяется через MCP к проекту base-one; файл миграции обязательно
  продублировать в репозитории сайта cozycorner (как делали с `add_pages_body`).
- Для будущих секций: posts/products получат свою колонку `folder_id` отдельной
  миграцией; таблица папок общая, секции разделяются полем `section`.

## 2. Слой данных

- Новый `src/lib/folders.ts` — generic, сразу с параметром `section`:
  - `FolderRow = { id, created_at, section, name }`;
  - `listFolders(site, section)` — `order by name`;
  - `createFolder(site, section, name)`;
  - `renameFolder(site, id, name)`;
  - `deleteFolder(site, id)`;
  - имя всегда `trim()`, пустое — ошибка (как в `renameImage`).
- Query-ключ папок: `['folders', site.slug, section]`.
- `src/lib/media.ts`:
  - `MediaRow` дополняется `folder_id: string | null`;
  - новая `moveImages(site, ids: string[], folderId: string | null)` —
    один `update media set folder_id where id in (…)`.
- Счётчики папок (All/Unsorted/по папкам) считаются на клиенте из уже загруженного
  `listImages` — дополнительных запросов нет.

## 3. UI (`src/features/media/`)

- **Панель папок** — узкая колонка слева внутри контента раздела Media (левое меню
  сайта из `SiteLayout` не меняется). Содержимое сверху вниз:
  `All (N)`, `Unsorted (N)`, папки по алфавиту с счётчиками, внизу «+ New folder».
  Клик по строке — фильтр грида. Активная папка — локальное состояние страницы
  (сбрасывается при уходе со страницы — приемлемо для v1).
- **Управление папкой** — меню (⋯) у строки папки:
  - Rename — инлайн-инпут (Enter/Esc), паттерн как у rename картинки в модалке;
  - Delete — AlertDialog с текстом вида «N images will be moved to Unsorted».
- **Перемещение файла**:
  - на карточке — кнопка-иконка папки с dropdown «Move to…» (Unsorted + список папок);
  - то же действие в модалке деталей (`MediaDetailsDialog`).
- **Multi-select** — чекбокс в углу превью карточки; при выборе ≥1 появляется бар
  «N selected · Move to ▾ · Clear». Bulk-операция v1 — только Move (без bulk delete).
- **Upload** — загруженный файл получает `folder_id` открытой сейчас папки
  (в All/Unsorted → null).
- **Search** — ищет по всем папкам, игнорируя выбранную (нашёл — увидел файл, в какой
  бы папке он ни лежал).
- Маркеры на превью, «used by», предупреждения при удалении файла — без изменений.
- Текст интерфейса — на английском (правило проекта).

## 4. Ошибки и краевые случаи

- Дубль имени папки (unique violation по `(section, name)`) → понятный тост
  «Folder with this name already exists».
- Пустое имя папки запрещено (валидация до запроса).
- Мутации move/create/rename/delete инвалидируют оба ключа: `['media', slug]` и
  `['folders', slug, section]`.
- «Битый» `folder_id` невозможен по FK (`on delete set null`).

## 5. Проверка

- `npm run lint`, `npm run build`.
- Ручной e2e-чек-лист (войдёт в план реализации): создать/переименовать/удалить папку
  (файлы уходят в Unsorted), move одного файла и пачки, upload в открытую папку,
  поиск сквозь папки, счётчики, анон не видит `admin_folders` (RLS).

## Вне скоупа v1 (сознательно)

- Drag&drop (вариант C) — добавляется поверх этой архитектуры после проверки логики.
- Вложенность папок, ручной порядок папок.
- Папки для posts/products — следующие итерации на той же таблице `admin_folders`.
- Bulk delete файлов.
