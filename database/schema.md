# База данных — Supabase (все сайты)

> Last updated: 2026-08-01 | Source project: cozycorner (CLAUDE.md, docs/admin-app-spec.md, docs/page-content.md, docs/multisite-migration.md) + web.admin

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
| Схема сайта Avocado Kiss | `avocado_kiss` (добавлена в Exposed schemas) |
| Storage-бакет Avocado Kiss | `avocado-kiss-photos` (плоские ключи) |

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
  НЕТ (весь CRUD только под админом). Колонка `posts.preview_token` НЕ выдаётся `anon`:
  у роли снят табличный `select` на `posts` и выдан column-level `select` на все колонки
  КРОМЕ `preview_token` (0031). ⚠️ Урок: column-level `revoke select (col)` НЕ
  перекрывает табличный `grant select` (0030 оказалась пустышкой) — чтобы скрыть колонку,
  надо снять табличный грант и выдать column-level на остальные. Из-за этого анон-выборки
  постов используют явный список колонок (`PUBLIC_POST_COLUMNS`), а не `select("*")`
  (иначе анонный `select *` упал бы на `preview_token`). Доступ к токену — только
  service-role (превью) и authenticated-админ (RLS).
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
| `slug` | text (unique, not null) | kebab-slug для `/product/[slug]`; автоген из `title` |
| `description` | text (nullable) | **Markdown** (подмножество: абзацы пустой строкой, списки `-`/`1.`, `**bold**`, `*italic*`; без ссылок/заголовков/кода). Старый плоский текст валиден как есть |
| `image_style` | text (not null, default `photo`) | `photo` — «живое» фото, cover во всю плитку; `cutout` — товар без фона/на белом, contain с паддингами на белой плитке |
| `seo_title` / `seo_description` | text (nullable) | SEO-оверрайды; пусто = фолбэк на `title`/`description` |
| `folder_id` | uuid (nullable, FK → admin_folders) | папка админки (§5 admin_folders); сайт поле не читает |

Категория(и) товара — **many-to-many** через `product_categories` (см. ниже). Прежней
колонки `products.category` больше нет (миграция 0028); товар может быть в 0..N категориях.

### categories — категории каталога

Источник карточек `/shop` и страниц `/shop/[slug]`. Товары связаны с категориями
**many-to-many** через `product_categories` (по `category_id`, см. ниже).

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at` | uuid pk, timestamptz | служебные |
| `name` | text (unique, not null) | отображаемое имя категории |
| `slug` | text (unique, not null) | для `/shop/[category]`; автоген из `name` |
| `image_path` | text (nullable) | **всегда cutout** (PNG с прозрачным фоном) — карточка на `/shop` |
| `position` | integer (not null, default 0) | порядок карточек |
| `hero_badge` / `hero_title` / `hero_description` | text (nullable) | hero страницы категории; пусто — фолбэки в коде (`Category` / `name` / автоформула) |
| `hero_image_path` | text (nullable) | фон hero; пусто — фолбэк на фон hero `/shop` |
| `seo_title` / `seo_description` | text (nullable) | SEO; пусто — автоформула из `name` |

### product_categories — связь товар↔категория (M2M)

Членство товаров в категориях (миграция 0027) — пришла на смену текстовой
`products.category`: товар может принадлежать нескольким категориям.

| Поле | Тип | Назначение |
|---|---|---|
| `product_id` | uuid (FK → products, on delete cascade) | товар |
| `category_id` | uuid (FK → categories, on delete cascade) | категория |

PK — `(product_id, category_id)`; индекс по `category_id` (выборка товаров категории).
RLS: public read (anon+authenticated), запись — `authenticated` + `public.is_admin()` —
как у products/categories. Удаление товара или категории чистит связи каскадом (FK), поэтому
переименование/удаление категории в админке больше не каскадит в товары (членство — по id).
Фронт `/shop/[category]` и связанные выборки фильтруют через inner-join
(`product_categories!inner`); админка правит связи в редакторе товара (чипы категорий) и
на странице категории (add/remove).

### brands — бренды каталога

Справочник брендов (как categories, но без картинки/hero/seo). Товары привязаны по имени:
`products.brand = brands.name`. FK нет сознательно — как у категорий.

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at` | uuid pk, timestamptz | служебные |
| `name` | text (unique, not null) | отображаемое имя; ключ связи с товарами |
| `slug` | text (not null) | автоген из `name` (триггер `brands_set_slug`, функция в `public`) |
| `position` | integer (not null, default 0) | порядок в списках |

RLS: `Public read brands` (SELECT anon+authenticated) + `Admin write brands` (ALL
authenticated, `public.is_admin()`) — как у categories. Миграция `0023_brands` засевает
таблицу уникальными `products.brand`. Переименование/удаление бренда в админке каскадит
в `products.brand` на уровне приложения (FK нет).

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
| `preview_token` | uuid (not null, default `gen_random_uuid()`) | capability-токен превью черновика (0029). НЕ флаг видимости. Анону НЕ выдаётся (column-grant, 0031) — читается только service-role клиентом на роуте `/preview/blog/[slug]` |

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

### header_settings — настройки шапки (singleton, одна строка)

Сайт-глобальные параметры шапки (миграция 0024, сразу в схеме `cozycorner`).
`id`, `created_at`, `updated_at`, `instagram_url` (nullable). Сайт читает первую
строку (`fetchHeaderSettings`, `lib/content.ts`) и передаёт `instagram_url` в
`<Header>` через `app/layout.tsx`; пусто/ошибка → иконка Instagram не рендерится
(безопасный фолбэк, как у footer). ⚠️ `app/layout.tsx` держит `export const
revalidate = 60` — иначе значение запекается в статический шелл и правки из админки
не видны до редеплоя (см. [../sites/cozycorner.md](../sites/cozycorner.md) §2).
RLS: публичное чтение + запись `is_admin()`.
Правится в админке разделом Pages → строка «Header» (`web.admin`, `src/lib/header.ts`
+ `features/pages/HeaderEditPage.tsx`). Таблица заведена с прицелом на рост — новые
параметры шапки = новые колонки.

### about_content — контент страницы /about (singleton, одна строка)

Управляемый контент статической страницы `/about` (миграция 0026, схема `cozycorner`).
Страница — фиксированный набор неповторяющихся секций (intro / story / три
карточки-ценности), поэтому это **singleton-таблица «колонка на поле»** (как
`footer_settings`/`header_settings`), а НЕ типизированные `*_sections` (те — под
повторяющиеся блоки). Новая секция/поле About = новая колонка, не новая строка.

Поля (все `text`, nullable — пусто → сайт показывает встроенный фолбэк секции):
`id`, `created_at`, `updated_at`, `eyebrow`, `heading`, `intro` (секция 1);
`story_heading`, `story_body_1`, `story_body_2`, `story_image_path` (секция 2, путь —
контракт §4); `card1_icon`/`card1_title`/`card1_text` … `card3_*` (три карточки,
`*_icon` — эмодзи-строка). RLS: публичное чтение + запись `is_admin()`; `updated_at`
триггером; сид одной строкой с дефолтным контентом. Сайт читает первую строку
(`fetchAboutContent`, `cozycorner/lib/content.ts`) и рендерит секции
`components/about/*` с фолбэками. Правится в админке разделом Pages → страница About
(`web.admin`, `src/lib/about.ts` + `features/pages/AboutContentEditor.tsx`, встроен в
`PageEditPage` — один Save на SEO + hero + контент About). Новое поле `story_image_path`
учтено в `USAGE_SOURCES` админки.

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

### subscribers — подписчики рассылки

Реальный сбор email из формы подписки в футере (`NewsletterForm`, миграция 0025).
`id`, `email` (`unique`, регистронезависимо — сервер пишет в lower-case),
`created_at`. Данные минимальны (GDPR): без IP/имени. Запись — ТОЛЬКО серверным
действием `lib/newsletter.ts` под `service_role` (обходит RLS) ПОСЛЕ проверки
Cloudflare Turnstile (режим Invisible), поэтому запись реально гейтится, а не
только UI. Ключи: `SUPABASE_SERVICE_ROLE_KEY` (server-only, вернулся ради этого
действия), `NEXT_PUBLIC_TURNSTILE_SITE_KEY`, `TURNSTILE_SECRET_KEY`. RLS:
публичных политик нет (анон не видит и не пишет — дефолтный `select` анону снят
явным `revoke`), единственная политика `Admin manage subscribers` (`is_admin()`)
— просмотр/удаление/экспорт CSV в админке (раздел Subscribers,
[../admin-panel/subscribers.md](../admin-panel/subscribers.md); право на забвение).
Privacy Policy (`pages.body`,
slug=`privacy`) ссылается на Turnstile Privacy Addendum — обязательное условие
Invisible-режима.

## 6. История миграций (репозиторий cozycorner, `supabase/migrations/`)

0001 init (products, legacy bucket) · 0002 bucket photos (legacy) · 0003 media ·
0004 pages + hero_sections · 0005 products.brand/category · 0006 footer_settings ·
0007 products.slug/description + slugify() · 0008 hero align + сид shop · 0009 posts ·
0010 секции постов · 0011–0012 демо-сиды · 0013 products.image_style · 0014 categories ·
0015 hero/seo-поля (categories/products/posts/pages-сиды) · 0016 admin_users + is_admin ·
0017 схема cozycorner (перенос 9 таблиц из public + гранты + write-RLS) ·
0018 bucket cozycorner-photos · 0019 pages.body · 0020 posts.post_type ·
0021 admin_folders + media.folder_id · 0022 products/posts.folder_id ·
0023 brands (справочник, сид из products.brand; RLS public read + admin write) ·
0024 header_settings (singleton настроек шапки, поле instagram_url; RLS public read +
admin write; сид одной пустой строкой) ·
0025 subscribers (email подписки на рассылку; `unique` email; RLS без публичных
политик — запись под service_role после проверки Turnstile, чтение/удаление только
admin) ·
0026 about_content (singleton-контент страницы /about: intro/story/три карточки,
колонка на поле; RLS public read + admin write; сид одной строкой с дефолтным
контентом) ·
0027 product_categories (связь товар↔категория M2M: PK `(product_id, category_id)`,
FK on delete cascade, RLS public read + admin write; бэкфилл из `products.category` по
имени — 50 связей) ·
0028 drop products.category (модель категорий стала M2M — текстовая колонка удалена) ·
0029 posts.preview_token (capability-токен превью черновика; `uuid not null default
gen_random_uuid`, бэкфилл существующих строк автоматом) ·
0030 revoke select(preview_token) от anon — **нерабочая попытка**: column-level revoke
НЕ перекрывает табличный grant select, эффекта нет (has_column_privilege остался true) ·
0031 корректный отзыв: `revoke select on posts` от anon + `grant select` на все колонки
КРОМЕ preview_token (применён ПОСЛЕ деплоя явных колонок в анон-выборках — иначе анонный
select упал бы; проверено: `has_column_privilege(anon, preview_token) = false`).

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

## 9. Таблицы схемы `avocado_kiss`

Второй сайт платформы (репозиторий `avocado.kiss/`, миграции —
`avocado.kiss/supabase/migrations/`). Схема **зависит от общих функций
`public.slugify`, `public.set_updated_at`, `public.is_admin`**, введённых
миграциями cozycorner 0007/0004/0016 — это тот же проект base-one, функции
уже есть, отдельно заводить не нужно. Те же общие соглашения, что в §5:
`slug` автогенерируется триггером из `title`/`name` (при insert не отправлять,
если пуст), `updated_at` — триггером, пустые optional-поля — `null`, пути
картинок — контракт §4. Фаза B (секции админки `web.admin` под эту схему) не
реализована — сейчас правка данных только напрямую в БД.

### recipes — рецепты

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at`, `updated_at` | uuid pk, timestamptz | служебные |
| `published_at` | timestamptz | дата публикации, сортировка лент |
| `is_published` | boolean (default false) | RLS: анону только опубликованные |
| `title` | text (not null) | название |
| `slug` | text (unique, not null) | для `/recipes/[slug]`; автоген из `title` |
| `excerpt` | text (nullable) | анонс (хедер рецепта, фолбэк описаний карточек) |
| `category` | text (nullable) | = `categories.name` (текст, FK нет — паттерн products/cozycorner; индекс) |
| `author_name` | text (nullable) | «by Mark P.» |
| `time_label` | text (nullable) | «45 minutes» — готовая строка |
| `servings_label` | text (nullable) | «4 servings» — готовая строка |
| `hero_image_path` | text (nullable) | ключ бакета `avocado-kiss-photos` или внешний URL (§4) |
| `ingredients` | text[] (not null, default `{}`) | порядок массива = порядок списка (не отдельная таблица) |
| `steps` | text[] (not null, default `{}`) | порядок массива = номера шагов 01, 02, … (не отдельная таблица) |
| `seo_title` / `seo_description` | text (nullable) | пусто = фолбэк `title`/`excerpt` |
| `folder_id` | uuid (nullable, FK → admin_folders, on delete set null) | папка админки |

### recipes — рейтинги (звёзды)

Оценки рецепта хранятся прямо на `recipes` как агрегаты + отдельная таблица
голосов; отображаемое среднее — GENERATED-колонки (миграция `0012`).

Новые колонки `recipes`:

| Поле | Тип | Назначение |
|---|---|---|
| `ratings_count` | integer (not null, default 0) | число реальных голосов (read-only агрегат, растит только RPC) |
| `ratings_sum` | integer (not null, default 0) | сумма реальных голосов (read-only агрегат) |
| `seed_count` | integer (not null, default 0) | админский baseline: число «затравочных» голосов |
| `seed_sum` | integer (not null, default 0) | админский baseline: сумма «затравочных» голосов |
| `min_display_rating` | numeric(2,1) (default 4.1) | пер-рецептный пол отображаемой оценки |
| `display_count` | integer GENERATED | `seed_count + ratings_count` |
| `display_avg` | numeric GENERATED | `greatest(min_display_rating, round((seed_sum + ratings_sum) / display_count, 1))` — с полом; при `display_count = 0` → `min_display_rating` |

`display_avg`/`display_count` — вычисляемые (STORED), напрямую не пишутся; сайт
читает только их (`Recipe.display_avg`/`display_count`).

### recipe_ratings — голоса за рецепт

Одна строка на голос. `id`, `recipe_id` (FK → recipes), `value` (int, check
`1..5`), `client_id` (идентификатор браузера для дедупа), `created_at`. RLS:
только админ (read/write) — **публичного/анонимного доступа нет** (анон не читает
и не пишет сырые голоса; сайт видит агрегаты через колонки `recipes`).

### rate_recipe() — приём голоса (RPC)

`avocado_kiss.rate_recipe(p_slug text, p_value int, p_client_id text)` —
`security definer`; EXECUTE выдан **только** роли `service_role`. Вставляет голос
в `recipe_ratings`, инкрементит агрегаты (`ratings_count`/`ratings_sum`) и
возвращает `(display_avg, display_count)`. Вызывается серверным Route Handler
`POST /api/recipes/[slug]/rate` под service-role ключом (см.
[../sites/avocado-kiss.md](../sites/avocado-kiss.md)) — первая мутация в репозитории.

Миграция: `avocado.kiss/supabase/migrations/0012_recipe_ratings.sql`.

### categories — категории навигации

`id`, `created_at`, `name` (**unique**, not null — единственный уникальный ключ,
на нём `on conflict`), `slug` (автоген из `name`; НЕ unique-constraint),
`position` (integer, default 0 — порядок ссылок в шапке), `show_in_nav`
(boolean, not null, default true — `false` убирает категорию из навигации шапки,
но страница `/category/<slug>`, поиск и главная остаются доступны),
`seo_title`, `seo_description` (nullable, фолбэк — автоформула из `name`).
Публичное чтение — все строки. Навигация шапки фильтрует по `show_in_nav = true`
(`fetchNavCategories`); страницы категорий и sitemap берут полный список
(`fetchCategories`). Рецепты привязаны по имени: `recipes.category =
categories.name` (без FK). Миграция `0011` добавила `show_in_nav` + категорию
`Dinner`, скрыла `Seafood` из навигации.

### home_slots — курируемая главная

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at`, `updated_at` | — | служебные |
| `slot` | text (not null, check) | `hero` \| `grid_large` \| `grid_medium` \| `grid_list` \| `wide` \| `pick` \| `pick_feature` \| `mosaic_banner` |
| `position` | integer (not null, default 0) | порядок внутри слота |
| `recipe_id` | uuid (nullable, FK → recipes, on delete cascade) | ссылка на рецепт (полиморфизм — см. ниже) |
| `post_id` | uuid (nullable, FK → posts, on delete cascade) | ссылка на пост блога (для Editor's Picks) |
| `eyebrow` | text (nullable) | оверрайд лейбла; пусто → категория рецепта |
| `eyebrow_secondary` | text (nullable) | устар.: до v2 — второй лейбл picks; теперь метки Editor's Picks берутся из тегов элемента |
| `description` | text (nullable) | оверрайд описания; пусто → excerpt рецепта |
| `is_published` | boolean (default true) | RLS: анону только опубликованные |

Ёмкость слотов по макету (валидирует админка фазы B, сайт рендерит что есть):
`hero` — 3, `grid_large` — 1, `grid_medium` — 2, `grid_list` — 5, `wide` — 2,
`pick` — 3, `pick_feature` — 1, `mosaic_banner` — 1 (широкий баннер мозаики
главной; слот допускает несколько строк — задел под слайдер, сайт рендерит
первую). Загрузчик сайта джойнит рецепты через
`recipes!inner` embed + двойной фильтр `is_published` (на слот и на embed) —
слот со скрытым рецептом отбрасывается целиком, а не рендерится пустым.
Подробности курирования — [../sites/avocado-kiss.md](../sites/avocado-kiss.md) §3.

**Полиморфизм (v2):** слот ссылается на **рецепт ИЛИ пост** — `recipe_id` и
`post_id` оба nullable, `check (num_nonnulls(recipe_id, post_id) = 1)` требует
ровно одну ссылку. Мозаика главной использует только рецептные слоты; секция
Editor's Picks (`pick` / `pick_feature`) — любой из двух типов. Загрузчик
`fetchEditorsPicks` разрешает рецепт/пост и берёт теги у самого элемента.

### tags — сквозные метки контента

Единая таксономия для всех типов контента (recipe/post/product). `id`,
timestamps, `name` (unique), `slug` (автоген из name, unique), `position`.
**Отличие от `categories`:** категории — раздел рецепта (несколько, в карточках
как SEASONAL); теги — сквозные метки: журнальные (LIFE, COMMUNITY) и дескрипторы
блюда (Vegetarian, Quick, Comfort Food, Healthy, Baking, Sweet, Brunch,
Weeknight, Summer, Spring, No-Cook, Vegan — сид 0006). Выводятся через «\|»
(`TagLabels`) в Editor's Picks и в теле карточки на страницах категорий (там
теги заменяют дублирующую категорию). Публичное чтение — все строки; запись — админ.

### recipe_tags / post_tags — связи контент↔теги

Join-таблицы many-to-many: `(recipe_id|post_id, tag_id)` PK + `position`
(порядок вывода тегов), `on delete cascade` с обеих сторон, индекс по `tag_id`.
`product_tags` появится вместе с товарами. Публичное чтение — все строки
(видимость гейтит сам рецепт/пост).

### posts — посты блога

2-й тип контента. `id`, timestamps, `published_at`, `is_published`, `title`,
`slug` (автоген из title, unique), `excerpt`, `hero_image_path` (тот же контракт
картинок). Миграция 0008 добавила: `template` (check `essay|interview|roundup`,
default essay — управляет вариантом hero/лэйаута), `author_id` (FK → authors),
`subtitle` (dek), `read_minutes` (int, «N min read»), `seo_title`,
`seo_description`. Миграция 0009 добавила `hero_caption` (подпись hero-фигуры для
roundup). Публичное чтение — только опубликованные; запись — админ.
Архив `/blog`, single `/blog/[slug]`. Загрузчики — `lib/blog.ts`.

### authors — авторы постов (миграция 0008)

`id`, `created_at`, `name`, `slug` (автоген из name, unique — задел под
`/author/[slug]`), `avatar_path` (контракт картинок), `bio`. Публичное чтение —
все; запись — админ. `posts.author_id` → authors.

### post_sections — блоки тела поста (миграция 0008)

**Единая таблица** динамических переупорядочиваемых блоков (в отличие от модели
«таблица на тип» у cozycorner — здесь один `type` + типизированные nullable
колонки, один `position`). Общие: `id`, timestamps, `post_id` (FK → posts, on
delete cascade), `position`, `is_published`, `type` (check
`text|quote|image|recipe_card|qa|list_item`). Колонки по типам: text — `body`,
`text_variant` (check lead|body); quote — `quote`, `quote_attribution`; image —
`image_path`, `caption`, `credit`; recipe_card — `recipe_id` (FK → recipes, on
delete set null), `card_eyebrow`; qa — `question`, `answer`; list_item — `rank`,
`heading` (+ переиспользует `body` и `recipe_id`/`card_eyebrow`). Индексы
`(post_id, position)` и `(recipe_id)`. Публичное чтение — `is_published`; запись
— админ. Загрузка — `fetchPostSections` (один запрос + гидрация рецептов одним
`.in()`), диспетчер `PostSections`. Реализованы essay-блоки
(text/quote/image/recipe_card); qa/list_item — задел под interview/roundup.

### post_related — «Read also» (миграция 0008)

Ручные пины связанного чтения, полиморфно: `id`, `post_id` (FK → posts, on
delete cascade), `position`, `recipe_id` (FK → recipes), `related_post_id` (FK →
posts) с check `num_nonnulls(recipe_id, related_post_id) = 1`. Публичное чтение —
все; запись — админ. `fetchRelatedReading` = пины сверху + авто-добор
(посты по общему тегу/свежести, затем рецепты) до 3 карточек.

### pages — страницы + SEO

Фиксированный набор из 3 строк (`home`, `shop`, `blog`) — create/delete нет.
`id`, timestamps, `slug` (unique), `seo_title`, `seo_description`,
`og_image_path` (§4). Строка `shop` — миграция 0007, `blog` — 0010. Миграция
0010 добавила **hero-баннер** для страниц `/shop` и `/blog`: `hero_eyebrow`,
`hero_title`, `hero_description`, `hero_image_path` (плоский ключ бакета или URL;
null → плейсхолдер) — раньше текст был захардкожен в `app/shop|blog/page.tsx`
(`HUB_HERO`/`HERO`), теперь страницы читают строку `pages` с фолбэком на эти
константы. У строки `home` баннера нет (мозаика). Markdown-полей/`body` по-прежнему
нет.

### footer_settings — текст футера (singleton, одна строка)

`tagline`, `copyright`, `made_with` («Made with care · New York — Lisbon»),
`instagram_url`, `pinterest_url`, `telegram_url`, `rss_url` + тексты
декоративного блока рассылки: `newsletter_eyebrow`, `newsletter_title`,
`newsletter_text`. Все text nullable (кроме id/timestamps); пусто — элемент
не рендерится. Форма рассылки ничего никуда не пишет (нет таблицы
`subscribers` в v1).

### media — метаданные загруженных файлов

По образцу cozycorner: источник истины для медиа-менеджера (не Storage).
`id`, `created_at`, `original_name`, `path` (unique, плоский ключ в бакете
`avocado-kiss-photos`), `mime`, `size`, `folder_id` (nullable, FK →
admin_folders). RLS: весь CRUD — только `is_admin()`, публичного чтения нет
(тот же нюанс диагностики, что и у cozycorner — см. §8).

### admin_folders — папки админки

`id`, `created_at`, `section` (check `'media' | 'recipes'`), `name`,
`unique(section, name)`; RLS admin-only. `recipes.folder_id` /
`media.folder_id` — nullable FK, `on delete set null` — удаление папки не
удаляет элементы. Сайт таблицу не читает.

### shop_categories — категории Curated Shop

Таксономия товаров магазина, **независима от рецептных `categories`** (другая
сенсибильность). `id`, `created_at`, `slug` (unique — `/shop/[slug]`; **без
триггера автогенерации** — задаётся явно при insert), `name`, `item_count`
(integer, default 0 — «31 items» на карточке хаба; держится согласованным с
фактическим числом товаров категории при посеве), `position` (порядок сетки
хаба), `hero_eyebrow`/`hero_title`/`hero_description`/`hero_image_path`
(nullable — hero страницы категории), `seo_title`/`seo_description`. Публичное
чтение — все строки; запись — `is_admin()`.

### products — товары Curated Shop

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at` | uuid pk, timestamptz | служебные |
| `slug` | text (unique, not null) | `/product/[slug]`; **без автоген-триггера** — задаётся явно |
| `title` | text (not null) | название |
| `brand` | text (nullable) | эйброу карточки; текст, без справочника брендов |
| `price` | numeric(10,2) (not null) | цена; форматируется `formatPrice()` |
| `description` | text (nullable) | тело страницы товара |
| `image_path` | text (nullable) | плоский ключ `avocado-kiss-photos` или внешний URL (§4); тестовый сид → null |
| `referral_url` | text (not null) | «Buy from …» — внешняя ссылка |
| `category` | text (nullable) | = `shop_categories.slug` (**текст, FK нет** — паттерн `recipes.category`; индекс) |

Индексы: `(category)` и `(created_at desc, id desc)` — детерминированная
пагинация `.range()` (как cozycorner). `category_name` в типе `Product` —
**вычисляемое** поле (lookup к `shop_categories.name` по slug), не колонка.
Товары **не тегируются** (нет `product_tags` — фильтры каталога статичны в v1).
Публичное чтение — все строки; запись — `is_admin()`.

### shop_editors_picks / product_pairings / product_reading — курация магазина

Три join-таблицы по образцу `home_slots` (ручной выбор редакции). Все: `id`,
`created_at`, `position` (порядок вывода), публичное чтение `using (true)`,
запись — `is_admin()`.

- **shop_editors_picks** — «Editors' picks this month» на `/shop` (4 товара).
  `product_id` (FK → products, on delete cascade), `unique(product_id)`.
- **product_pairings** — «Pairs well with» на странице товара (товар → товар, по
  3). `product_id` + `paired_product_id` (оба FK → products, cascade),
  `unique(product_id, paired_product_id)`, `check (product_id <> paired_product_id)`
  (без само-ссылок). Embed неоднозначен (2 FK на products) — загрузчик указывает
  явный хинт `products!product_pairings_paired_product_id_fkey`.
- **product_reading** — «Related reading» на странице товара (товар → **реальный
  рецепт**, по 3). `product_id` (FK → products) + `recipe_id` (FK → **recipes**,
  cascade), `unique(product_id, recipe_id)`. RLS даёт `using (true)`, но
  загрузчик embed'ит `recipes!inner` + `eq('recipe.is_published', true)` (плюс
  RLS на recipes пускает только опубликованные) — неопубликованные рецепты в
  блок не протекают (паттерн `fetchHomeSlots`). Проекция → `ReadingItem`
  (`href=/recipes/{slug}`, `eyebrow=category`, `imagePath=hero_image_path`).

Строка `pages.shop` (SEO хаба) добавлена миграцией 0007 — см. `pages` выше.

## 10. История миграций (репозиторий avocado.kiss, `supabase/migrations/`)

0001 схема `avocado_kiss` (categories, admin_folders, recipes, home_slots,
pages, footer_settings, media + гранты + RLS, шаблон cozycorner 0017) ·
0002 bucket `avocado-kiss-photos` (public read, запись `is_admin()`, шаблон
0018) · 0003 демо-сид (8 категорий из макета, демо-рецепты — тексты рецепта
zucchini и карточки главной из мокапов, `home_slots` на все слоты, строки
`pages.home` и `footer_settings`) · 0004 слот `mosaic_banner` (расширение
check-констрейнта `home_slots.slot` + демо-строка баннера; редизайн главной
v2) · 0005 модель контента: `tags` + `recipe_tags` + `posts` + `post_tags`,
полиморфные слоты (`home_slots.recipe_id` XOR `post_id`, check
`num_nonnulls=1`), сид тегов picks (редизайн главной v2, Editor's Picks) ·
0006 больше контента категорий: теги-дескрипторы блюд + новые рецепты
(Breakfast/Seafood были пусты) + бэкфилл hero-картинок (внешние URL) +
привязка тегов к рецептам категорий · 0007 Curated Shop: `shop_categories`,
`products` (текстовая `category`, без FK), курация `shop_editors_picks` +
`product_pairings` + `product_reading` (по образцу `home_slots`; RLS/гранты
шаблона 0001) + строка `pages.shop`. Посев (6 категорий, 36 товаров
`image_path=null`, 4 picks, по 3 пары/чтения на товар → опубликованные рецепты)
— через `avocado-content-ops`, не миграцией. · 0008 блог (Article): `authors`,
расширение `posts` (`template`/`author_id`/`subtitle`/`read_minutes`/seo-поля),
единая таблица блоков тела `post_sections` (все 6 типов в check), `post_related`
(Read also, полиморфно) + RLS/гранты шаблона 0005 + сид demo-essay «The quiet art
of the weeknight table» (8 секций). Роуты `/blog`, `/blog/[slug]`; nav «Article». ·
0009 `posts.hero_caption` + demo-посты interview («Recipes are just letters to
strangers», блоки qa) и roundup («20 pies…», блоки list_item) + теги
Interview/People/Test Kitchen + автор «The Test Kitchen». · 0010 hero-баннер в
`pages` (`hero_eyebrow`/`hero_title`/`hero_description`/`hero_image_path` для
`/shop` и `/blog`) + строка `pages.blog` + сид текущей копии.

Ручной шаг после 0001: схема `avocado_kiss` добавлена в **Exposed schemas**
(готово). Картинки-заглушки для сида загружаются в бакет отдельным шагом
(не миграцией).
