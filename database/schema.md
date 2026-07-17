# База данных — Supabase (все сайты)

> Last updated: 2026-07-17 | Source project: cozycorner (CLAUDE.md, docs/admin-app-spec.md, docs/page-content.md, docs/multisite-migration.md) + web.admin

Единый источник правды о структуре БД и контрактах с данными. Схема **версионируется
миграциями в репозитории `cozycorner/supabase/migrations/`** — менять её из других мест
нельзя (см. §7).

## 1. Проект и подключение

| Параметр | Значение |
|---|---|
| Supabase-проект | **base-one** |
| Project ref | `zwrkphynupdubevzwdzy` |
| Project URL | `https://zwrkphynupdubevzwdzy.supabase.co` |
| Publishable (anon) key | `sb_publishable_Xilw_s7Be73d0Q2ryqxodA_1AeIqdKJ` (публичный, защита — RLS) |
| Схема сайта CozyCorner | `cozycorner` (добавлена в Exposed schemas) |
| Storage-бакет CozyCorner | `cozycorner-photos` (плоские ключи) |

- Переезд со старого проекта `nkaobsivfzsjqypuaamw` выполнен 2026-07-16: миграции
  среплеены, данные и Storage перенесены, md5-суммы таблиц сверены, RLS проверен.
  Экспорт-снапшот и импорт-скрипт — `~/Documents/Projects/cozycorner-migration/`.
- Старый проект живёт только как откат. Финализация (этап G, по команде пользователя):
  удалить старый проект, его MCP-сервер `supabase` из `~/.claude.json` и бэкап
  `~/.claude.json.bak-migration`; папку `cozycorner-migration/` архивировать.
  Ротация secret key, использованного для импорта, — на пользователе.
- **Service-role ключ не нужен ни сайту, ни админке.** Никогда не класть `sb_secret_…`
  в переменные с префиксом `NEXT_PUBLIC_*` / `VITE_*` — он уедет в браузер.

## 2. Мультисайт-архитектура

Один Supabase-проект держит несколько сайтов (экономия на подписке; расселение по
отдельным проектам — когда упрёмся в лимиты Free: 500 MB БД / 1 GB Storage / 5 GB egress).

- **Схема на сайт**: `cozycorner.*`, `<site2>.*` — внутри обычные имена таблиц
  (`products`, `posts`, …) без префиксов. Принадлежность данных сайту читается из имени
  схемы (в т.ч. для Supabase MCP/коннектора Claude).
- **Бакет на сайт**: `cozycorner-photos`, `<site2>-photos`. Чтение публичное, запись —
  политики на `storage.objects` через `is_admin()`.
- **Общее для админки — в `public`**: `admin_users` (whitelist) + `is_admin()`.
- **Настройка новой схемы** (иначе PostgREST → permission denied / 406):
  1. добавить схему в **Exposed schemas** (Project Settings → Data API) — ручной шаг;
  2. в миграции: `GRANT USAGE ON SCHEMA` + `GRANT SELECT` для `anon`, полный доступ
     `authenticated` (под RLS) + `ALTER DEFAULT PRIVILEGES`;
  3. supabase-js: сайт — `db: { schema: '<site>' }`; админка — `.schema('<site>')` per-query.
- Мультипроект (на будущее): у каждого Supabase-проекта своя Auth (JWT непереносимы) —
  админ-логин заводится в каждом проекте, клиент per-project со своим `auth.storageKey`;
  каждому проекту нужен свой Supabase-коннектор/MCP (project ref).

## 3. Модель доступа (Auth + RLS)

- `public.admin_users(user_id uuid pk → auth.users, email, created_at)` — whitelist
  админов (миграция 0016). `public.is_admin()` → boolean (SECURITY DEFINER): true, если
  текущий `auth.uid()` есть в `admin_users`.
- **Чтение контента**: публичный `select` для `anon` (исключения ниже). **Запись**:
  `authenticated` при `is_admin()` — или service role / дашборд. Storage-бакет сайта:
  публичное чтение; запись/удаление — `is_admin()`.
- Ограничения чтения: `posts` — анону только `is_published = true`; секции постов и
  hero-секции — только опубликованные; `media` и `admin_folders` — публичного чтения
  НЕТ (весь CRUD только под админом).
  ⚠️ Диагностика: anon-запросы к `media` ВСЕГДА возвращают 0 строк, даже когда данные
  есть, — проверять содержимое через SQL Editor / MCP / Management API (роль postgres).
- Auth: Email-провайдер включён, публичная регистрация закрыта; 2 админа заведены и
  внесены в whitelist (eugeniusz.lipko@gmail.com, yury.brankovsky@gmail.com).
- Security advisor: осознанные WARN — public-bucket listing; `is_admin` executable
  (для анона всегда false); рекомендация включить leaked password protection.

## 4. Контракт путей картинок (критично, общий для сайта и админки)

Все поля `*image_path*` хранят **относительный плоский ключ** внутри бакета сайта
(например `product-1.png`, без вложенных папок) ИЛИ **внешний URL** (Amazon, Unsplash —
как есть). **Абсолютные URL нашего Storage как формат хранения запрещены** — тогда
переезд на другой Supabase-проект = смена одной переменной, ссылки не бьются.

- Публичный URL строится на лету: `${projectUrl}/storage/v1/object/public/${bucket}/${path}`.
- Разбор: сайт — `resolveProductImage()` (`cozycorner/lib/products.ts`); админка —
  `resolveImageUrl` / `toStoragePath` (`web.admin/src/lib/images.ts`).
- При записи из админки в БД попадает **только относительный ключ**, никогда полный URL.

## 5. Таблицы схемы `cozycorner`

Общие соглашения (везде): `slug` автогенерируется БД-триггером из `title`/`name`, если
пуст — при insert slug не отправлять; `updated_at` обновляется триггером — вручную не
задавать; пустые optional-поля пишутся как `null`, не пустой строкой.

### products — товары

| Поле | Тип | Назначение |
|---|---|---|
| `id` | uuid (pk) | идентификатор |
| `created_at` | timestamptz | сортировка (новые первыми) |
| `title` | text | название товара |
| `price` | numeric(10,2) | цена (сайт форматирует `formatPrice`, USD) |
| `image_path` | text | ключ в бакете или внешний URL (§4) |
| `referral_url` | text | реферальная ссылка «где купить» |
| `brand` | text (nullable) | бренд — фильтрация (индекс) |
| `category` | text (nullable) | категория = `categories.name` (текст, FK нет; индекс) |
| `slug` | text (unique, not null) | kebab-slug для `/product/[slug]`; автоген из `title` |
| `description` | text (nullable) | **Markdown** (подмножество: абзацы пустой строкой, списки `-`/`1.`, `**bold**`, `*italic*`; без ссылок/заголовков/кода). Старый плоский текст валиден как есть |
| `image_style` | text (not null, default `photo`) | `photo` — «живое» фото, cover во всю плитку; `cutout` — товар без фона/на белом, contain с паддингами на белой плитке |
| `seo_title` / `seo_description` | text (nullable) | SEO-оверрайды; пусто = фолбэк на `title`/`description` |
| `folder_id` | uuid (nullable, FK → admin_folders) | папка админки (§5 admin_folders); сайт поле не читает |

### categories — категории каталога

Источник карточек `/shop` и страниц `/shop/[slug]`. Товары привязаны по имени:
`products.category = categories.name`.

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at` | uuid pk, timestamptz | служебные |
| `name` | text (unique, not null) | отображаемое имя; ключ связи с товарами |
| `slug` | text (unique, not null) | для `/shop/[category]`; автоген из `name` |
| `image_path` | text (nullable) | **всегда cutout** (PNG с прозрачным фоном) — карточка на `/shop` |
| `position` | integer (not null, default 0) | порядок карточек |
| `hero_badge` / `hero_title` / `hero_description` | text (nullable) | hero страницы категории; пусто — фолбэки в коде (`Category` / `name` / автоформула) |
| `hero_image_path` | text (nullable) | фон hero; пусто — фолбэк на фон hero `/shop` |
| `seo_title` / `seo_description` | text (nullable) | SEO; пусто — автоформула из `name` |

### posts — блог и SEO-посты

Архив `/blog`; single `/blog/[slug]`. RLS чтения: анону только опубликованные.

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at`, `updated_at` | — | служебные |
| `published_at` | timestamptz | дата публикации: сортировка архива |
| `is_published` | boolean | черновики скрыты от анона |
| `slug` | text (unique, not null) | автоген из `title` |
| `post_type` | text (not null, default `blog`) | `blog` — обычный пост; `seo` — programmatic-SEO-пост: скрыт из лент и поиска, открывается по прямому URL, включён в sitemap. **Без cloaking**: редиректы / UA-логика / noindex запрещены |
| `title` / `excerpt` / `content` | text | заголовок / анонс / **легаси-поле `content` не читать и не писать** (тело поста — секции) |
| `cover_image_path` | text (nullable) | обложка (§4) |
| `seo_title` / `seo_description` | text (nullable) | пусто = фолбэк `title`/`excerpt` |
| `folder_id` | uuid (nullable) | папка админки |

### post_text_sections + post_product_sections — секции поста

Модель «таблица на тип секции». Общие поля: `post_id` (fk → posts, on delete cascade),
`position`, `is_published` (RLS отдаёт только опубликованные), timestamps.
**`position` — сквозная нумерация через ОБЕ таблицы**: сайт сливает их и сортирует.

- `post_text_sections`: `heading` (h2, nullable), `body` (Markdown, абзацы пустой строкой).
- `post_product_sections`: `heading` (nullable), `columns` (2|3, default 3),
  `product_ids uuid[]` — реальные id из `products`, порядок массива = порядок карточек.

### pages — страницы + SEO

Фиксированный набор из 7 строк (`home`, `shop`, `blog`, `terms`, `privacy`, `about`,
`featured`) — задан кодом сайта; create/delete страниц нет.

| Поле | Тип | Назначение |
|---|---|---|
| `id`, timestamps | — | служебные |
| `slug` | text (unique) | ключ страницы (на нём маршруты сайта — read-only) |
| `seo_title` / `seo_description` | text | `<title>`+og:title / meta+og:description |
| `og_image_path` | text | картинка для соцсетей (§4) |
| `body` | text (nullable) | **Markdown-тело**, сейчас только `terms`/`privacy`; NULL → сайт показывает встроенный текст (безопасный фолбэк) |
| `meta` | jsonb | произвольные мета-теги; **контракта нет — не читать и не писать** (везде `{}`) |

### hero_sections — hero статических страниц

`page_id` (fk → pages, cascade), `position`, `is_published`, `badge` (nullable),
`title` (not null), `description` (nullable), `bg_image_path` (nullable, §4),
`align` (`left`|`right`, default `right`) — положение текстовой карточки.
Набор секций управляется миграциями; админка делает только update существующих.

### footer_settings — текст футера (singleton, одна строка)

`tagline`, `newsletter_title`, `newsletter_subtitle`, `disclaimer`, `copyright`.
⚠️ `disclaimer` содержит обязательную по Amazon Associates формулировку «As an Amazon
Associate we earn from qualifying purchases» — она должна оставаться на каждой странице
с реф-ссылками (футер глобальный — условие выполняется).

### media — метаданные загруженных файлов

Источник истины для списка в медиа-менеджере админки (не Storage).
`id`, `created_at`, `original_name` (исходное имя, меняется только оно при rename),
`path` (unique, плоский ключ в бакете), `mime`, `size`, `folder_id` (nullable).
RLS: весь CRUD — только `is_admin()`, публичного чтения нет. FK с `products` нет:
«использование» вычисляется сравнением строк `path` ↔ поля `*image_path*`.

### admin_folders — папки админки (организационная сущность админки, не сайта)

`id`, `created_at`, `section` (`'media' | 'products' | 'posts' | 'seo_posts'`), `name`;
`unique(section, name)`; RLS admin-only. В контентных таблицах — nullable
`folder_id uuid references admin_folders(id) on delete set null` + индекс: удаление
папки не удаляет элементы (уходят в Unsorted через FK). Сайт таблицу не читает; у
products/posts `folder_id` виден anon-выборкам (просто uuid — принято осознанно).
Секция `seo_posts` в `SECTION_TABLES` указывает на таблицу `posts` (имя секции ≠ таблице).

## 6. История миграций (репозиторий cozycorner, `supabase/migrations/`)

0001 init (products, legacy bucket) · 0002 bucket photos (legacy) · 0003 media ·
0004 pages + hero_sections · 0005 products.brand/category · 0006 footer_settings ·
0007 products.slug/description + slugify() · 0008 hero align + сид shop · 0009 posts ·
0010 секции постов · 0011–0012 демо-сиды · 0013 products.image_style · 0014 categories ·
0015 hero/seo-поля (categories/products/posts/pages-сиды) · 0016 admin_users + is_admin ·
0017 схема cozycorner (перенос 9 таблиц из public + гранты + write-RLS) ·
0018 bucket cozycorner-photos · 0019 pages.body · 0020 posts.post_type ·
0021 admin_folders + media.folder_id · 0022 products/posts.folder_id.

Нюанс хронологии: 0021 (`add_admin_folders`) и 0022 (`add_content_folders`) применены
к base-one 2026-07-16 через MCP из админки **до** 0020 (`add_seo_post_type`) — файлы
добавлены в репозиторий задним числом (2026-07-17), когда номера 0019/0020 уже были
заняты. Взаимный порядок 0021 → 0022 соответствует порядку применения; все три файла
идемпотентны, реплей безопасен.

## 7. Изменение схемы — workflow

1. Новый файл миграции в `cozycorner/supabase/migrations/` (таблицы контента — в схеме
   сайта; общее для админки — в `public`). Из админки/чата схему можно применить через
   Supabase MCP `apply_migration` — но файл миграции ОБЯЗАТЕЛЬНО продублировать в
   репозитории сайта сразу же (прецедент отложенных дублей — 0021/0022, §6).
2. Применить к проекту (MCP / дашборд / CLI).
3. Синхронизировать типы сайта `cozycorner/lib/types.ts` (Product/Category/Post).
4. Новые поля `*image_path*` → добавить в `USAGE_SOURCES` админки
   (`web.admin/src/lib/media.ts`), чтобы «used by» оставался точным.
5. Обновить этот файл.

Как добавить новый тип секции страницы/поста: `create table <name>_sections`
(`id`, timestamps, `page_id`|`post_id` fk, `position`, `is_published`, типизированные
поля) + индекс по fk + updated_at-триггер + RLS публичного чтения опубликованных;
тип и загрузка — `cozycorner/lib/content.ts` (или `lib/posts.ts`), компонент секции,
подключение в page.tsx по `position`; обновить этот файл.

## 8. Диагностика (после переезда / при странностях)

1. Media-раздел пуст или «used by» не видит фото → RLS: `media` admin-only, anon видит
   0 строк — это норма, проверять от postgres.
2. Логин в админку не проходит → учётки существуют только в base-one; сброс пароля
   требует прод-URL админки в Auth → URL Configuration (Redirect URLs) — pending.
3. Картинки не отображаются → пути в БД должны быть относительными; если в данных
   всплыл ref старого проекта — это баг данных, чинить путь на относительный.
4. MCP отвечает про старый проект → проверить `--project-ref` в `~/.claude.json`;
   после правки — перезапуск сессии.
5. PostgREST «permission denied» / пустая схема → схема в Exposed schemas + гранты (0017).
6. Прямой SQL без MCP: Management API
   `POST https://api.supabase.com/v1/projects/zwrkphynupdubevzwdzy/database/query`
   с personal access token. Нюанс: python-urllib режется Cloudflare (403/1010) — curl.
7. Быстрая проверка, из какой БД собран прод сайта:
   `curl -sL <прод-URL> | grep -o '<project-ref>'` — в HTML зашиты Storage-URL.
