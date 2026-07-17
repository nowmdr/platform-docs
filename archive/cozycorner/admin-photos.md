# Админка загрузки фото + bucket `photos`

> ⚠️ **УСТАРЕЛО (историческая справка).** Эта встроенная админка (`/admin`, `/api/admin`,
> `components/admin`, `lib/supabase/admin.ts`, `lib/supabase/storage.ts`, service-role)
> **удалена** из репозитория. Управление медиа вынесено в отдельное приложение (Vite React
> SPA, Supabase Auth + RLS через `is_admin()`), а картинки переехали в bucket
> `cozycorner-photos` с плоскими путями. Данные — в схеме `cozycorner`. Актуальный контекст —
> [../../platform-docs/database/schema.md](../../database/schema.md). Документ ниже
> описывает прежнюю реализацию и оставлен для истории.

История: внутренний MVP для управления медиа и консолидация хранилища картинок
товаров. Реализовано 2026-06-24.

## 1. Что сделано (кратко)

1. Страница `/admin` — загрузка фото в Supabase Storage, список с превью, удаление,
   понятные статусы.
2. Новый публичный bucket **`photos`** — единое хранилище для админки и для картинок
   товаров.
3. Перенос 9 картинок из старого bucket `product-images` в `photos` и переключение
   фронта на `photos`.
4. Таблица метаданных `public.media`: имя файла хранится как есть (`original_name`),
   уникальный `id` — отдельная колонка (и папка объекта в Storage).

## 2. Архитектура

Запись/удаление в Storage невозможны под `anon`-ключом (RLS даёт только чтение),
поэтому все операции идут через серверные Route Handlers с **service-role** ключом —
ключ живёт только на сервере и не попадает в браузер.

```
[Client UI /admin]  --fetch-->  [Route Handlers /api/admin/photos]  --service-role-->  [Storage: bucket "photos"]
  'use client'                    Web Request / FormData                lib/supabase/storage.ts
  состояния UI                    GET список / POST upload / DELETE      lib/supabase/admin.ts (server-only)
```

Стек — Next.js 16 App Router: Route Handlers на Web `Request`, динамический `params`
(`await`), GET без кеша, `FormData` через `await request.formData()`.

## 3. Файлы

| Файл | Роль |
|---|---|
| [lib/supabase/admin.ts](../../../cozycorner/lib/supabase/admin.ts) | `server-only` клиент на service-role (обходит RLS) |
| [lib/supabase/storage.ts](../../../cozycorner/lib/supabase/storage.ts) | `uploadImage` / `listImages` / `getImageUrl` / `deleteImage` + валидация |
| [app/api/admin/photos/route.ts](../../../cozycorner/app/api/admin/photos/route.ts) | `GET` список, `POST` загрузка |
| [app/api/admin/photos/[id]/route.ts](../../../cozycorner/app/api/admin/photos/%5Bid%5D/route.ts) | `DELETE` по `id` (uuid) |
| [app/admin/page.tsx](../../../cozycorner/app/admin/page.tsx) | RSC-страница (`noindex`, `force-dynamic`) |
| [components/admin/](../../../cozycorner/components/admin/) | `AdminPhotos` (контейнер), `PhotoUploader`, `PhotoGrid`, `PhotoCard`, `types.ts`, `admin.module.css` |
| [supabase/migrations/0002_photos_bucket.sql](../../../cozycorner/supabase/migrations/0002_photos_bucket.sql) | публичный bucket `photos` + read-политика |
| [supabase/migrations/0003_media_table.sql](../../../cozycorner/supabase/migrations/0003_media_table.sql) | таблица метаданных `public.media` (RLS, без публичных политик) |

## 4. Storage, метаданные и правила

- Bucket **`photos`** — публичный (политика `select` для `anon`/`authenticated`).
  Запись/удаление — только service-role (insert/update/delete политик нет).
- **Метаданные — в таблице `public.media`** (источник истины для списка админки):

  | Колонка | Тип | Назначение |
  |---|---|---|
  | `id` | uuid (pk) | идентификатор; он же папка объекта в Storage |
  | `created_at` | timestamptz | сортировка (новые первыми) |
  | `original_name` | text | **исходное имя файла — не меняется** |
  | `path` | text (unique) | ключ объекта в bucket |
  | `mime` | text | тип |
  | `size` | bigint | размер в байтах |

  RLS включён, публичных политик нет → доступ только через service-role.
- **Путь объекта:** `uploads/<id>/<sanitized-name>`. Уникальность ключа даёт `id`
  (поэтому два файла `headphones.png` не конфликтуют), а имя в UI берётся из
  `original_name` и остаётся нетронутым.
- Загрузка: объект в Storage + строка в `media` одной операцией; если insert падает —
  загруженный объект откатывается (удаляется), чтобы не плодить «сирот».
- Список — запрос к `media` (быстро, сортировка/пагинация дальше — тривиальны).
  Удаление — по `id`: убираем объект из Storage и строку из `media`.
- Валидация: mime ∈ {jpeg, png, webp, gif}, размер ≤ 6 МБ. Файлы > 6 МБ в будущем —
  через resumable/TUS upload.

### Целостность `products` ↔ `media`

`products.image_path` — это текст (URL или относительный путь), **не связанный FK** с
`media`. Поэтому «использование» вычисляется сопоставлением строк: `listImages()` тянет
`products` и для каждой записи `media` собирает `usedBy` — товары, где `image_path` равен
`media.path` (относительный) или его публичному URL (абсолютный). В админке такие карточки
помечены «используется: …».

⚠️ Удаление при этом **не блокируется** (осознанный выбор): пометка предупреждает, но
удалить используемое фото можно — тогда картинка товара станет битой. Жёсткая защита
(блокировка delete или FK) — задел на будущее.

## 5. Картинки товаров: `image_path` и консолидация на `photos`

`image_path` в `public.products` теперь принимает **два формата**, оба разворачивает
[publicImageUrl()](../../../cozycorner/lib/products.ts):

- абсолютный публичный URL (фото из `/admin`) → возвращается как есть;
- относительный путь внутри `photos` (напр. `product-1.png`) → достраивается до
  `…/storage/v1/object/public/photos/<path>`.

Выполненная миграция данных:

- 9 объектов `product-1.png … product-9.png` **скопированы** `product-images → photos`
  (разовым скриптом download+upload под service-role; оригиналы оставлены).
- fallback-bucket в `publicImageUrl()` переключён `product-images → photos`.
- Bucket `product-images` (миграция 0001) остаётся как **legacy** — фронтом не
  используется, можно удалить позднее.

Разовая правка целостности (2026-06-24): товар «Наушники» ссылался на удалённый объект
(битая картинка) — перепривязан к существующему `headphones.jpg` из `media`. Все объекты
bucket `photos` (включая `pled-main.webp` и `product-1..9.png`, загруженные до `media`)
**забэкфиллены** в `media` одним `INSERT … SELECT FROM storage.objects` — теперь они видны
и управляемы в админке. `product-1.png` — неиспользуемый «сирота» (можно удалить).

> Миграцию 0001_init.sql не редактировали (применённая история); смена bucket
> отражена в коде и docs.

## 6. Окружение

```
SUPABASE_SERVICE_ROLE_KEY=sb_secret_...   # ТОЛЬКО на сервере, без префикса NEXT_PUBLIC_!
```

Шаблон — в [.env.example](../../../cozycorner/.env.example). Нужен для записи/удаления в Storage.

## 7. Безопасность

⚠️ `/api/admin/*` и `/admin` сейчас **без авторизации** — для внутреннего MVP это
сознательно. Не выкладывать публично без защиты. Задел: один `middleware.ts` на
матчеры `/admin` и `/api/admin/:path*` (+ при желании `requireAdmin()` в хендлерах).

## 8. Проверка (выполнена)

- `npm run lint`, `npm run build` — без ошибок; `/admin` и API-роуты динамические.
- E2E через API: пустой список → upload (`201`) → список (`1`) → публичный URL (`200`)
  → отказ для не-картинки (`400`) → delete по `id` (`ok`) → список снова пуст.
- Имена/id: загрузка `headphones.png` → `name` остаётся `headphones.png`, `id` — отдельный
  uuid; повторная загрузка того же имени не конфликтует (другой `id`); delete несуществующего
  id → `400 «Файл не найден»`.
- Картинки товаров из `photos` отдаются `200`; `next/image` принимает абсолютные URL
  (хост `*.supabase.co` разрешён в [next.config.ts](../../../cozycorner/next.config.ts)).

## 9. Возможные расширения

Авторизация; кнопка «привязать загруженное фото к товару» в `/admin`; сортировка,
пагинация, множественная загрузка, drag&drop; приватный bucket + signed URL; удаление
legacy-bucket `product-images`.
