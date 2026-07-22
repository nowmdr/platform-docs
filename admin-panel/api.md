# Админ-панель — контракт доступа к данным

> Last updated: 2026-07-22 | Source project: web.admin (CLAUDE.md, docs/admin-app-spec.md §5–§9, docs/media-manager.md) — пути `src/…` относятся к репозиторию `web.admin/`

У админки нет своего сервера и REST API — «API» это прямой доступ к Supabase
(PostgREST + Storage) под RLS. Модель данных — [../database/schema.md](../database/schema.md).
Правила конкретных разделов — [media.md](media.md), [products.md](products.md),
[blog.md](blog.md), [pages.md](pages.md); UI — [components.md](components.md).

## 1. Реестр сайтов — `src/config/sites.ts`

Единственное место с подключениями. Publishable-ключи публичны — реестр коммитится.

```ts
type SiteConfig = {
  slug: string;        // короткий id, часть URL админки: /cozycorner/media
  label: string;       // имя в дашборде
  projectUrl: string;  // https://<ref>.supabase.co
  anonKey: string;     // publishable/anon ключ (публичный)
  schema: string;      // Postgres-схема сайта
  bucket: string;      // Storage-бакет сайта
};
```

Новый сайт = новая запись (+ свои `projectUrl`/`anonKey`, если он в другом
Supabase-проекте). Env-переменных нет — деплой подхватывает всё из кода.

## 2. Клиенты — `src/lib/supabase.ts`

**Актуальная реализация (отклонение от спеки §6)**: один supabase-клиент на проект
(schema `public` — сессия, rpc, storage) + `getDb(site)` = `.schema(site.schema)`
per-query для данных. Клиент создаётся per-project со своим `auth.storageKey`
(`sb-admin-<projectUrl>`) — сессии разных проектов не перетирают друг друга в
localStorage. Файлы — `storage.from(site.bucket)`.

## 3. Auth — `src/lib/auth.ts` + `components/RequireAuth.tsx`

- `signIn` → `auth.signInWithPassword`; `isAdmin` → `rpc("is_admin")` (функция в `public`).
- Guard `RequireAuth`: сессия есть? нет → redirect `/login` (с `state.from` — возврат
  после логина). `is_admin()` false → полноэкранное «Access denied» + Sign out.
- Известный баг первого логина (исправлен, не регрессировать): после signIn сессию
  кладут в кэш TanStack (`setQueryData(['auth', projectUrl])`) ДО navigate — иначе
  гард видит устаревшее «нет сессии» и возвращает на /login.
- Ложный «Access denied» при возврате во вкладку (исправлен 2026-07-22, не
  регрессировать): в фоне тормозятся таймеры, `autoRefreshToken` не успевает, и
  focus-refetch auth-запроса гонялся с обновлением токена → RPC `is_admin` с
  протухшим JWT → `false` → экран Access denied (перезагрузка чинила). Фикс:
  (1) `isAdmin` при ошибке RPC **бросает**, а не возвращает `false` (транзиентный
  сбой ≠ «не в whitelist»; при фоновом refetch RQ держит последнее хорошее значение);
  (2) auth-запрос в `RequireAuth` — `refetchOnWindowFocus: false` + `staleTime: Infinity`,
  свежесть только через `onAuthStateChange` (refetch на TOKEN_REFRESHED/SIGNED_IN/OUT —
  уже с валидным токеном).
- **Запись в БД/Storage работает только под сессией админа** (RLS + `is_admin()`).
  Service-role ключа в админке нет и не должно быть никогда.

## 4. Медиа/Storage — `src/lib/media.ts` (детали: [media.md](media.md))

- **Upload** `uploadImage(site, file, folderId = null)`: валидация (mime ∈ jpeg/png/webp/gif,
  ≤ 6 МБ) → плоский ключ `sanitize(имя)` (нижний регистр, `[^a-z0-9._-]→-`; коллизии —
  суффикс `-2`, `-3`… по `media.path`) → `storage.upload(key, { upsert: false })` →
  insert в `media`. **Откат**: insert упал → удалить объект из Storage (не плодить сирот).
- **List** `listImages(site)`: читает `media` (`created_at desc`) + собирает `usedBy`
  прямым равенством `path` по `USAGE_SOURCES`: `products.image_path`,
  `categories.image_path|hero_image_path`, `posts.cover_image_path`,
  `hero_sections.bg_image_path`, `pages.og_image_path`. Новые поля картинок в схеме →
  дополнить `USAGE_SOURCES`.
- **Rename** — меняет ТОЛЬКО `media.original_name`; `path`/Storage/URL не трогаются.
- **Delete** — объект из Storage → строка `media`; «используется» предупреждает, но
  **не блокирует** (осознанно — картинка на сайте станет битой).
- **Move** `moveToFolder(site, section, ids, folderId)` (`src/lib/folders.ts`) — generic
  для секций media/products/posts/seo_posts; меняет только `folder_id`.

## 5. Картинки в формах (контракт UI ↔ БД)

В БД — плоский ключ бакета или внешний URL ([schema.md §4](../database/schema.md)).
В UI-инпуте всегда **публичная ссылка**: чтение разворачивает `resolveImageUrl`,
сохранение сворачивает наш storage-URL обратно в ключ `toStoragePath(site, value)`
(`src/lib/images.ts`); внешние URL проходят как есть. Нарушать нельзя — иначе переезд
проекта побьёт ссылки. Выбор из галереи — общий `ImagePickerDialog` (упрощённо: пикер
папок не знает, upload из него идёт в Unsorted).

## 6. Общие паттерны слоя данных

- **Query-ключи**: список `['<entity>', site.slug]` (посты — `[..., postType]`), деталь
  `[..., id]`, папки `['folders', site.slug, section]`, медиа `['media', site.slug]` —
  пикеры используют ТОТ ЖЕ ключ, что раздел Media (общий кэш, свой не заводить).
  После Save: деталь — `setQueryData` свежими данными, список — инвалидация с
  `exact: true` (иначе сотрёт только что положенный детальный ключ).
- **Пустые optional-поля → `null`** (`orNull` в `toInput`), не пустые строки.
- **SEO-поля «живые»**: значение, равное фолбэку (title/description|excerpt) или пустое,
  пишется как `null` — SEO меняется вместе с сущностью, а не замораживается копией.
- **Diff-сохранение секций поста** (blog): insert новых → update существующих → delete
  пропавших в конце; после Save обязательна пересинхронизация формы свежими данными
  (иначе повторный Save продублирует новые секции) — детали и риски: [blog.md](blog.md) §1.
- **Update-only** (pages): insert/delete нет, повторный Save после падения безопасен.
- `slug` не отправлять (БД-триггер), `updated_at` не задавать (триггер),
  легаси `posts.content` и `pages.meta` не читать и не писать (явные списки колонок).

## 7. Деплой и окружение

- Vercel, preset **Vite** (build `npm run build`, output `dist`); SPA-rewrite всех путей
  на `/index.html` (`vercel.json`).
- Env: только `VITE_*` с publishable-ключом — либо значения прямо в `sites.ts`
  (текущий вариант). Никаких `SERVICE_ROLE` во фронте.
- Прод-URL админки добавить в Supabase → Auth → URL Configuration (Redirect URLs) —
  нужно для ссылок сброса пароля (обычному логину не мешает). Статус — [status.md](status.md).

## 8. Критичные правила (не нарушать)

1. Только publishable/anon ключ. Service-role — никогда.
2. Все мутации — после логина админа (RLS/`is_admin()`).
3. В полях картинок — относительный плоский путь, не абсолютный URL.
4. Новая схема сайта → Exposed schemas; бакет — public read + write через `is_admin()`
   (шаблон — миграции 0017/0018 сайта).
5. Схема БД меняется миграциями в репозитории сайта, не из админки; применённое через
   MCP — продублировать файлом (см. [schema.md §7](../database/schema.md)).
