# Avocado Kiss — дизайн v1 (сайт + схема БД)

> Дата: 2026-07-17 | Статус: одобрено пользователем | Реализация: фаза A (сайт + БД),
> фаза B (секции админки — отдельная спека)

Avocado Kiss — кулинарный журнал (рецепты, английский UI), третий проект платформы
workspace-one. Источник дизайна — два SingleFile-макета из Lovable
(`avocado.kiss/mockups/avocado-kiss-{home,recipe}.html`): главная и страница рецепта.
Сайт работает на общей БД (Supabase **base-one**) и управляется общей админкой
`web.admin`.

## Принятые решения

| Вопрос | Решение |
|---|---|
| Вёрстка | Как cozycorner: дизайн-токены в `globals.css` + CSS Modules, без Tailwind (макет на Tailwind — переписываем 1:1 по значениям) |
| Объём v1 | Главная + страница рецепта + страницы категорий |
| Главная | Курируется из админки через таблицу `home_slots` |
| Рассылка | Блок декоративный: тексты редактируются, submit никуда не пишет |
| Ингредиенты/шаги | Колонки `ingredients text[]` / `steps text[]` в `recipes` (порядок = порядок массива) |

## 1. Репозиторий и стек

- Next.js-проект в существующей папке `avocado.kiss/` (package name `avocado-kiss`);
  макеты переносятся в `avocado.kiss/mockups/`.
- Стек 1:1 как cozycorner: Next.js 16 (App Router, RSC, TypeScript), supabase-js
  (publishable key, `db: { schema: 'avocado_kiss' }`), GSAP для анимаций, без
  CSS-фреймворков, без react-markdown (в v1 нет markdown-полей).
- Структура: `app/` (тонкие `page.tsx`: загрузка данных + композиция), `components/`
  (компонент = файл + свой `.module.css`; SVG-иконки компонентами в `icons/`), `lib/`
  (весь доступ к данным, Supabase-клиенты в `lib/supabase/`),
  `supabase/migrations/` (схема `avocado_kiss` версионируется здесь).
- Шрифт Fira Sans (OFL) — self-hosted woff2 в `public/fonts` + `@font-face`
  в `globals.css` (regular, medium, semibold, bold).
- Все контентные страницы SSG + ISR `revalidate = 60`; никаких своих опций кэша
  в Supabase-fetch. Env: `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
- Деплой Vercel (пуш в `main` → авто-деплой) — настраивается при запуске.

## 2. Маршруты

| Маршрут | Содержимое | Данные |
|---|---|---|
| `/` | Главная по макету | `home_slots` + `recipes` + `categories` + `pages(home)` + `footer_settings` |
| `/recipes/[slug]` | Рецепт по макету | `recipes` |
| `/category/[slug]` | Хедер категории + сетка карточек рецептов (макета нет — собирается из существующих карточек) | `categories` + `recipes` |
| `sitemap.xml` | Главная + категории + опубликованные рецепты | — |
| `robots.txt`, 404 | Стандартно | — |

Поиск в шапке — в v1 нерабочий рендер (поле/иконка как в макете); функциональность —
отдельной задачей. Ссылки футера Magazine (About, Contributors, Contact, Issue
Archive) — в v1 без страниц-адресатов.

## 3. Перенос дизайна (структура макетов)

**Токены** (`:root` в `globals.css`): `--background:#fcfaf6`, `--foreground:#0b1c2c`,
`--paper:#f8f5ef`, `--muted:#f1eee9`, `--muted-foreground:#576574`,
`--accent:#467748`, `--accent-foreground:#fcfaf6`, `--rule:#cfcac2`,
`--border:#dbd7d0`, `--destructive:#cc2827`, `--radius:.25rem`,
`--tracking-eyebrow:.22em` + типографическая шкала макета. Правило cozycorner
«tokens first»: новых hardcode-значений нет, новое значение → новый токен.
Глобальная утилита `.eyebrow` (uppercase, letter-spacing .22em, 11px, 600).

**Главная** (сверху вниз):
1. Sticky-шапка с blur (`bg-background/90`, backdrop-blur): строка поиска слева
   (≥md), логотип «AVOCADO KISS» по центру, кнопки поиска/меню на мобильном;
   под ней — навигация категорий (8 ссылок, uppercase, hover → accent).
2. Hero-карусель: 3 слайда, aspect 4/3 (моб.) / 16/9 (≥md), градиент слева,
   eyebrow «Recipe of the week», заголовок `clamp(2rem,4.5vw,4.5rem)`, точки +
   стрелки-кнопки внизу по центру; весь слайд — ссылка на рецепт.
3. Секция «Recipes»: центрированный хедер (eyebrow «Section» + h2) →
   сетка 12 колонок: большая карточка (6 кол., aspect 5/4, описание), колонка
   из 2 средних карточек (3 кол., aspect 1/1), список из 5 текстовых ссылок
   (3 кол., слева border-l); ниже — 2 широкие карточки (фото 4/5 + текст рядом,
   автор + время через точку-разделитель).
4. «Editor's Picks» (фон `--paper`, border-t): слева нумерованный список 1–3
   (номер 35% opacity, двойной лейбл «Food | Drinks», заголовок, описание);
   справа sticky-карточка (фото 4/5 + двойной лейбл + крупный заголовок + текст).
5. Блок рассылки «The Culinary Dispatch»: инверсия (фон `--foreground`),
   центрированный, eyebrow + h2 + текст + форма (email + «Subscribe →»,
   underline-стиль).
6. Футер: бренд + tagline (2 кол.), колонки «Magazine» и «Follow»; нижняя строка —
   copyright + «Made with care · New York — Lisbon».

**Страница рецепта**: ссылка «‹ Back» → хедер (eyebrow, h1, excerpt, полоса меты
Serves/Time с иконками между border-y) → фото 16/10 (моб.) / 16/9 → две колонки
`1fr / 1.6fr`: Ingredients (буллеты-точки accent, разделители border-b) и Method
(номера 01–06 шрифтом accent + текст шага).

**Компоненты**: Header, CategoryNav, HeroCarousel, RecipeCard (варианты
large / medium / list / wide), EditorsPicks, NewsletterBlock, Footer, RecipeMeta,
IngredientsList, MethodSteps, MobileMenu. Каждый — свой `.module.css`.

**Анимации**: hero-карусель — клиентский компонент, кроссфейд GSAP, автоплей с
паузой при hover, точки + стрелки; hover-зум картинок (scale 1.03, 1200ms) и смена
цвета заголовков — чистый CSS как в макете; скролл-ревилы — сдержанно, по правилам
GSAP из design-system дока cozycorner.

**Картинки**: `next/image`, remotePatterns только `*.supabase.co` (внешние URL —
`unoptimized`, allowlist не расширять). Разбор путей — свой `resolveRecipeImage()`
в `lib/` по контракту §4 schema.md.

## 4. Схема БД `avocado_kiss` + бакет `avocado-kiss-photos`

Общие соглашения платформы: slug автогенерируется БД-триггером из `title`/`name`
(при insert не отправлять, если пуст); `updated_at` — триггером; пустые
optional-поля = `null`; пути картинок — относительные плоские ключи бакета или
внешние URL; чтение anon — только опубликованное, запись — `authenticated` при
`is_admin()` (общая функция из `public`).

### recipes

| Поле | Тип | Назначение |
|---|---|---|
| `id`, `created_at`, `updated_at` | uuid pk, timestamptz | служебные |
| `published_at` | timestamptz | дата публикации, сортировка лент |
| `is_published` | boolean (default false) | RLS: анону только опубликованные |
| `title` | text (not null) | название |
| `slug` | text (unique, not null) | для `/recipes/[slug]`; автоген из `title` |
| `excerpt` | text (nullable) | анонс (хедер рецепта, фолбэк описаний карточек) |
| `category` | text (nullable) | = `categories.name` (текст, FK нет — паттерн products; индекс) |
| `author_name` | text (nullable) | «by Mark P.» |
| `time_label` | text (nullable) | «45 minutes» — готовая строка |
| `servings_label` | text (nullable) | «4 servings» — готовая строка |
| `hero_image_path` | text (nullable) | ключ бакета или внешний URL |
| `ingredients` | text[] (not null, default '{}') | порядок массива = порядок списка |
| `steps` | text[] (not null, default '{}') | порядок массива = номера 01, 02, … |
| `seo_title` / `seo_description` | text (nullable) | пусто = фолбэк `title`/`excerpt` |
| `folder_id` | uuid (nullable, FK → admin_folders, on delete set null) | папка админки |

### categories

`id`, `created_at`, `name` (unique, not null), `slug` (unique, автоген из `name`),
`position` (integer, default 0 — порядок в навигации шапки), `seo_title`,
`seo_description` (nullable, фолбэк — автоформула из `name`). Публичное чтение —
все строки (навигация).

### home_slots — курируемая главная

| Поле | Тип | Назначение |
|---|---|---|
| `id`, timestamps | — | служебные |
| `slot` | text (not null, check) | `hero` \| `grid_large` \| `grid_medium` \| `grid_list` \| `wide` \| `pick` \| `pick_feature` |
| `position` | integer (not null, default 0) | порядок внутри слота |
| `recipe_id` | uuid (not null, FK → recipes, on delete cascade) | какой рецепт показывать |
| `eyebrow` | text (nullable) | оверрайд лейбла; пусто → категория рецепта |
| `eyebrow_secondary` | text (nullable) | второй лейбл в picks («Food \| Drinks») |
| `description` | text (nullable) | оверрайд описания; пусто → excerpt рецепта |
| `is_published` | boolean (default true) | RLS: анону только опубликованные |

Ёмкость слотов по макету (валидирует админка, сайт рендерит что есть): `hero` — 3,
`grid_large` — 1, `grid_medium` — 2, `grid_list` — 5, `wide` — 2, `pick` — 3,
`pick_feature` — 1. Сайт join'ит рецепты и скрывает слоты с неопубликованными
рецептами.

### pages

Фиксированный набор из 1 строки `home` (create/delete нет): `id`, timestamps,
`slug` (unique), `seo_title`, `seo_description`, `og_image_path`.

### footer_settings — singleton (одна строка)

`tagline`, `copyright`, `made_with`, `instagram_url`, `pinterest_url`,
`telegram_url`, `rss_url` + тексты блока рассылки: `newsletter_eyebrow`,
`newsletter_title`, `newsletter_text`. Все text nullable (кроме id/timestamps);
пусто → элемент не рендерится.

### media + admin_folders

По образцу cozycorner (миграции 0003/0021): `media` (`original_name`, `path`
unique, `mime`, `size`, `folder_id`) — RLS весь CRUD только `is_admin()`, без
публичного чтения; `admin_folders` (`section` check `'media' | 'recipes'`, `name`,
unique(section,name)) — admin-only.

### Миграции и настройка

- Файлы в `avocado.kiss/supabase/migrations/` (нумерация с 0001): схема + гранты
  по шаблону cozycorner 0017 (`GRANT USAGE`, `GRANT SELECT` anon, полный доступ
  authenticated под RLS, `ALTER DEFAULT PRIVILEGES`), бакет `avocado-kiss-photos`
  по шаблону 0018 (public read, запись `is_admin()`), таблицы + триггеры
  (slugify, updated_at) + RLS, сид: 8 категорий из макета, демо-рецепты
  (тексты рецепта zucchini и карточек главной из макетов), `home_slots` на все
  слоты, строки `pages.home` и `footer_settings`.
- Картинки-заглушки загружаются в бакет отдельным шагом (не миграцией).
- Ручные шаги (дашборд): добавить `avocado_kiss` в Exposed schemas.
- После применения: обновить `platform-docs/database/schema.md`.

## 5. Интеграция с админкой — фаза B (отдельная спека)

Схема выше спроектирована под существующие паттерны админки. Объём фазы B:
запись в `SITES` (`web.admin/src/config/sites.ts`, slug `avocado-kiss`), секции
Recipes (CRUD + редакторы-списки для `ingredients`/`steps`), Categories, Home
(пикер рецептов по слотам с лимитами), Pages, Footer, Media (переиспользование
готовых компонентов; `USAGE_SOURCES` — добавить `recipes.hero_image_path` и
`pages.og_image_path`). Сейчас фиксируется только контракт БД.

## 6. Верификация и документация

- `npm run build` (TS + prerender всех маршрутов) и `npm run lint` — зелёные.
- Визуальная сверка с макетами (`mockups/*.html` в браузере рядом с dev-сервером):
  токены, сетки, брейкпоинты, hover, карусель.
- Документация по чек-листу overview.md: создать `platform-docs/sites/avocado-kiss.md`,
  добавить сайт в таблицу overview.md, в `platform-docs/AGENTS.md` (Projects) и
  корневой AGENTS.md воркспейса; обновить `database/schema.md`; в репозитории
  сайта — AGENTS.md/CLAUDE.md по образцу cozycorner.

## Вне объёма v1

Рабочий поиск, сбор e-mail подписки (таблица subscribers), страницы About/
Contributors/Contact/Issue Archive, markdown-поля, фото у шагов рецепта,
мобильное выезжающее меню как отдельный диалог (v1 — кнопка + простое раскрытие).
