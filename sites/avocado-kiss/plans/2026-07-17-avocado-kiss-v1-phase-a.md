# Avocado Kiss v1 — Phase A Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Запустить сайт avocado.kiss (главная + рецепт + категории) на стеке cozycorner с собственной схемой БД `avocado_kiss` в Supabase base-one.

**Architecture:** Next.js 16 App Router (RSC, SSG + ISR 60s) читает контент из схемы `avocado_kiss` анонимным ключом под RLS; дизайн переносится 1:1 из Tailwind-макетов в дизайн-токены + CSS Modules; главная курируется таблицей `home_slots`. Спека: `platform-docs/sites/avocado-kiss/specs/2026-07-17-avocado-kiss-v1-design.md`.

**Tech Stack:** Next.js 16.2.9, React 19, TypeScript, @supabase/supabase-js, GSAP (+@gsap/react), CSS Modules, Fira Sans (self-hosted).

**Верификация вместо юнит-тестов:** в сайтах workspace нет тест-инфраструктуры (как в cozycorner); контроль качества = `npm run build` (TS + prerender), `npm run lint`, визуальная сверка с макетами `mockups/*.html`. Каждая задача заканчивается зелёным build.

**Рабочая директория всех задач:** `/Users/mdr/Documents/Projects/workspace-one/avocado.kiss`

---

## Контент из макетов (источник истины для сидов)

Категории навигации (position 0–7): Breakfast, Salads, Pasta, Baking, Desserts, Vegan, Seafood, Drinks.

Полный рецепт (макет recipe): «Roasted zucchini with white beans and kale», excerpt «A bright, hearty plate that turns a few pantry staples into a slow-cooked dinner worth lingering over.», Serves «4 servings», Time «45 minutes», 9 ингредиентов, 6 шагов (полные тексты — в Task 4).

Слоты главной: hero ×3 · grid_large ×1 · grid_medium ×2 · grid_list ×5 · wide ×2 · pick ×3 · pick_feature ×1 (тексты — в Task 4).

---

### Task 1: Скаффолд репозитория

**Files:**
- Move: `avocado-kiss-home.html`, `avocado-kiss-recipe.html` → `mockups/`
- Create: `package.json`, `next.config.ts`, `.env.local`, `app/layout.tsx`, `app/page.tsx`, `app/globals.css` (заглушка)
- Copy from `../cozycorner/`: `tsconfig.json`, `eslint.config.mjs`, `.gitignore`

- [ ] **Step 1: git init и перенос макетов**

```bash
cd /Users/mdr/Documents/Projects/workspace-one/avocado.kiss
git init -b main
mkdir mockups
mv avocado-kiss-home.html avocado-kiss-recipe.html mockups/
cp ../cozycorner/.gitignore ../cozycorner/tsconfig.json ../cozycorner/eslint.config.mjs .
```

- [ ] **Step 2: package.json** (стек cozycorner без react-markdown — в v1 нет markdown-полей)

```json
{
  "name": "avocado-kiss",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint"
  },
  "dependencies": {
    "@gsap/react": "^2.1.2",
    "@supabase/supabase-js": "^2.108.2",
    "gsap": "^3.15.0",
    "next": "16.2.9",
    "react": "19.2.4",
    "react-dom": "19.2.4",
    "server-only": "^0.0.1"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "eslint": "^9",
    "eslint-config-next": "16.2.9",
    "typescript": "^5"
  }
}
```

- [ ] **Step 3: next.config.ts** (контракт картинок §4 schema.md — allowlist не расширять)

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "*.supabase.co",
        pathname: "/storage/v1/object/public/**",
      },
    ],
  },
};

export default nextConfig;
```

- [ ] **Step 4: .env.local** (publishable ключ — публичный, защита RLS; `.env.local` в .gitignore)

```bash
cat > .env.local <<'EOF'
NEXT_PUBLIC_SUPABASE_URL=https://zwrkphynupdubevzwdzy.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=sb_publishable_Xilw_s7Be73d0Q2ryqxodA_1AeIqdKJ
EOF
```

- [ ] **Step 5: минимальный app/** — `app/globals.css` пока пустой файл; `app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";

// Шрифт проекта — Fira Sans (OFL), self-hosted: @font-face в globals.css (Task 2).

export const metadata: Metadata = {
  title: {
    default: "Avocado Kiss — a culinary magazine",
    template: "%s — Avocado Kiss",
  },
  description:
    "Seasonal recipes, chef stories, and the aesthetics of the home kitchen. A new magazine for those who cook without rush.",
  openGraph: {
    title: "Avocado Kiss — a culinary magazine",
    description:
      "Seasonal recipes, chef stories, and the aesthetics of the home kitchen.",
    type: "website",
  },
};

export default function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

`app/page.tsx` (временная, заменяется в Task 9):

```tsx
export default function HomePage() {
  return <main>Avocado Kiss</main>;
}
```

- [ ] **Step 6: установка и проверка**

```bash
npm install
npm run build && npm run lint
```
Expected: build успешен (2 маршрута: `/`, `/_not-found`), lint без ошибок.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "chore: scaffold avocado-kiss (Next 16 + cozycorner stack)"
```

---

### Task 2: Шрифты + дизайн-токены globals.css

**Files:**
- Create: `public/fonts/fira-sans-{400,500,600,700}.woff2`
- Modify: `app/globals.css`

- [ ] **Step 1: скачать Fira Sans (latin, OFL) из fontsource CDN**

```bash
cd /Users/mdr/Documents/Projects/workspace-one/avocado.kiss
mkdir -p public/fonts
for w in 400 500 600 700; do
  curl -fsSL "https://cdn.jsdelivr.net/fontsource/fonts/fira-sans@latest/latin-$w-normal.woff2" \
    -o "public/fonts/fira-sans-$w.woff2"
done
ls -la public/fonts   # 4 файла, каждый ≥ 10KB
```

- [ ] **Step 2: app/globals.css целиком** (токены = значения из макета; правило «tokens first» — новых hardcode-значений в компонентах нет)

```css
/* =============================================================================
   Avocado Kiss — Design System.
   Единый источник правды для цветов, типографики, отступов. Значения перенесены
   1:1 из макетов mockups/*.html (Tailwind-тема Lovable). Не хардкодим цвета и
   размеры в компонентах — только var(--token); новое значение → новый токен.
   ========================================================================== */

@font-face {
  font-family: "Fira Sans";
  src: url("/fonts/fira-sans-400.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}
@font-face {
  font-family: "Fira Sans";
  src: url("/fonts/fira-sans-500.woff2") format("woff2");
  font-weight: 500;
  font-style: normal;
  font-display: swap;
}
@font-face {
  font-family: "Fira Sans";
  src: url("/fonts/fira-sans-600.woff2") format("woff2");
  font-weight: 600;
  font-style: normal;
  font-display: swap;
}
@font-face {
  font-family: "Fira Sans";
  src: url("/fonts/fira-sans-700.woff2") format("woff2");
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

:root {
  /* --- Цвета (из :root макета) --- */
  --background: #fcfaf6;        /* фон страницы */
  --foreground: #0b1c2c;        /* основной текст; фон инверсных секций */
  --paper: #f8f5ef;             /* фон секции Editor's Picks */
  --muted: #f1eee9;             /* фон поля поиска */
  --muted-foreground: #576574;  /* вторичный текст */
  --accent: #467748;            /* зелёный: eyebrow, hover, буллеты, номера шагов */
  --accent-foreground: #fcfaf6; /* текст на accent */
  --rule: #cfcac2;              /* линейки-разделители (border-rule) */
  --border: #dbd7d0;            /* прочие границы */
  --destructive: #cc2827;

  /* --- Типографика --- */
  --font-sans: "Fira Sans", -apple-system, BlinkMacSystemFont, "SF Pro Text",
    "Segoe UI", Roboto, sans-serif;
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;
  --text-3xl: 1.875rem;
  --text-4xl: 2.25rem;
  --text-5xl: 3rem;
  --text-6xl: 3.75rem;
  --tracking-tight: -0.025em;
  --tracking-eyebrow: 0.22em;
  --tracking-logo: 0.18em;
  --leading-snug: 1.375;
  --leading-relaxed: 1.625;

  /* --- Геометрия --- */
  --radius: 0.25rem;            /* rounded-sm макета — все скругления картинок */
  --shell-max: 1320px;          /* контентные секции */
  --shell-wide-max: 1600px;     /* шапка и hero */

  /* --- Анимация --- */
  --dur-hover-zoom: 1200ms;     /* зум картинок в карточках */
  --ease-default: cubic-bezier(0.4, 0, 0.2, 1);
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html,
body {
  /* clip, не hidden — hidden ломает sticky-шапку (правило cozycorner) */
  overflow-x: clip;
}

body {
  min-height: 100vh;
  background: var(--background);
  color: var(--foreground);
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

a {
  color: inherit;
  text-decoration: none;
}

button {
  font: inherit;
  color: inherit;
  background: none;
  border: none;
  cursor: pointer;
}

ul,
ol {
  list-style: none;
}

img {
  max-width: 100%;
  display: block;
}

/* --- Глобальные утилиты (общие для многих компонентов) --- */

/* Лейбл-«бровь»: uppercase-надпись над заголовками (класс .eyebrow макета) */
.eyebrow {
  display: inline-block;
  font-size: 0.6875rem;
  font-weight: 600;
  letter-spacing: var(--tracking-eyebrow);
  text-transform: uppercase;
  color: var(--muted-foreground);
}

.eyebrowAccent {
  color: var(--accent);
}

/* Контентная обёртка 1320px (секции главной, футер) */
.shell {
  max-width: var(--shell-max);
  margin: 0 auto;
  padding-inline: 1rem;
}

/* Широкая обёртка 1600px (шапка, hero) */
.shellWide {
  max-width: var(--shell-wide-max);
  margin: 0 auto;
  padding-inline: 1rem;
}

@media (min-width: 768px) {
  .shell {
    padding-inline: 1.5rem;
  }
}

@media (min-width: 1024px) {
  .shell,
  .shellWide {
    padding-inline: 3rem;
  }
}
```

- [ ] **Step 3: проверка и commit**

```bash
npm run build && npm run lint
git add -A && git commit -m "feat: design tokens + Fira Sans self-hosted"
```

---

### Task 3: Миграция 0001 — схема avocado_kiss (таблицы, триггеры, RLS)

**Files:**
- Create: `supabase/migrations/0001_avocado_kiss_schema.sql`

Зависимости из public (уже существуют в base-one из миграций cozycorner): `public.slugify` (0007), `public.set_updated_at` (0004), `public.is_admin` (0016).

- [ ] **Step 1: файл миграции целиком**

```sql
-- Avocado Kiss: собственная схема сайта (мультисайт: схема на сайт, шаблон —
-- cozycorner 0017). Использует общие функции public.slugify / public.set_updated_at /
-- public.is_admin из миграций cozycorner 0004/0007/0016 (проект base-one).
--
-- РУЧНОЙ ШАГ после применения: добавить `avocado_kiss` в Exposed schemas
-- (Project Settings → Data API), иначе PostgREST → permission denied / 406.

create schema if not exists avocado_kiss;

-- Доступ к схеме (шаблон 0017): чтение под RLS — anon; запись под RLS —
-- authenticated; service_role — полный доступ.
grant usage on schema avocado_kiss to anon, authenticated, service_role;
alter default privileges in schema avocado_kiss grant select on tables to anon;
alter default privileges in schema avocado_kiss grant all on tables to authenticated, service_role;
alter default privileges in schema avocado_kiss grant usage, select on sequences to anon, authenticated;
alter default privileges in schema avocado_kiss grant all on sequences to service_role;

-- --- categories: категории навигации и страниц /category/[slug] -------------
create table if not exists avocado_kiss.categories (
  id              uuid primary key default gen_random_uuid(),
  created_at      timestamptz not null default now(),
  name            text not null unique,
  slug            text,                    -- автоген из name триггером
  position        integer not null default 0, -- порядок ссылок в шапке
  seo_title       text,
  seo_description text
);

create or replace function avocado_kiss.categories_set_slug()
returns trigger language plpgsql
set search_path = ''
as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.name), ''), 'category');
  end if;
  return new;
end;
$$;

drop trigger if exists categories_set_slug on avocado_kiss.categories;
create trigger categories_set_slug before insert or update on avocado_kiss.categories
  for each row execute function avocado_kiss.categories_set_slug();

alter table avocado_kiss.categories alter column slug set not null;
create unique index if not exists categories_slug_key on avocado_kiss.categories (slug);

-- --- admin_folders: папки админки (шаблон cozycorner 0021) -------------------
create table if not exists avocado_kiss.admin_folders (
  id         uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  section    text not null check (section in ('media', 'recipes')),
  name       text not null,
  unique (section, name)
);

-- --- recipes: рецепты --------------------------------------------------------
create table if not exists avocado_kiss.recipes (
  id              uuid primary key default gen_random_uuid(),
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  published_at    timestamptz not null default now(), -- сортировка лент (новые первыми)
  is_published    boolean not null default false,
  title           text not null,
  slug            text,                               -- автоген из title триггером
  excerpt         text,                               -- анонс; фолбэк описаний карточек
  category        text,                               -- = categories.name (текст, FK нет — паттерн products)
  author_name     text,                               -- «by Mark P.»
  time_label      text,                               -- «45 minutes» — готовая строка
  servings_label  text,                               -- «4 servings» — готовая строка
  hero_image_path text,                               -- плоский ключ бакета или внешний URL (§4 schema.md)
  ingredients     text[] not null default '{}',       -- порядок массива = порядок списка
  steps           text[] not null default '{}',       -- порядок массива = номера шагов 01, 02, …
  seo_title       text,                               -- null = фолбэк title
  seo_description text,                               -- null = фолбэк excerpt
  folder_id       uuid references avocado_kiss.admin_folders(id) on delete set null
);

create or replace function avocado_kiss.recipes_set_slug()
returns trigger language plpgsql
set search_path = ''
as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.title), ''), 'recipe');
  end if;
  return new;
end;
$$;

drop trigger if exists recipes_set_slug on avocado_kiss.recipes;
create trigger recipes_set_slug before insert or update on avocado_kiss.recipes
  for each row execute function avocado_kiss.recipes_set_slug();

drop trigger if exists recipes_set_updated_at on avocado_kiss.recipes;
create trigger recipes_set_updated_at before update on avocado_kiss.recipes
  for each row execute function public.set_updated_at();

alter table avocado_kiss.recipes alter column slug set not null;
create unique index if not exists recipes_slug_key on avocado_kiss.recipes (slug);
create index if not exists recipes_published_idx
  on avocado_kiss.recipes (is_published, published_at desc, id desc);
create index if not exists recipes_category_idx on avocado_kiss.recipes (category);
create index if not exists recipes_folder_idx on avocado_kiss.recipes (folder_id);

-- --- home_slots: курируемая главная ------------------------------------------
-- Ёмкость слотов по макету (валидирует админка, сайт рендерит что есть):
-- hero 3 · grid_large 1 · grid_medium 2 · grid_list 5 · wide 2 · pick 3 · pick_feature 1.
create table if not exists avocado_kiss.home_slots (
  id                uuid primary key default gen_random_uuid(),
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),
  slot              text not null check (slot in
    ('hero','grid_large','grid_medium','grid_list','wide','pick','pick_feature')),
  position          integer not null default 0,      -- порядок внутри слота
  recipe_id         uuid not null references avocado_kiss.recipes(id) on delete cascade,
  eyebrow           text,   -- оверрайд лейбла; null → категория рецепта
  eyebrow_secondary text,   -- второй лейбл в picks («Food | Drinks»)
  description       text,   -- оверрайд описания; null → excerpt рецепта
  is_published      boolean not null default true
);

drop trigger if exists home_slots_set_updated_at on avocado_kiss.home_slots;
create trigger home_slots_set_updated_at before update on avocado_kiss.home_slots
  for each row execute function public.set_updated_at();

create index if not exists home_slots_slot_idx on avocado_kiss.home_slots (slot, position);
create index if not exists home_slots_recipe_idx on avocado_kiss.home_slots (recipe_id);

-- --- pages: фиксированный набор страниц с SEO (v1 — только home) -------------
create table if not exists avocado_kiss.pages (
  id              uuid primary key default gen_random_uuid(),
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  slug            text not null unique,  -- ключ страницы, маршруты сайта read-only
  seo_title       text,
  seo_description text,
  og_image_path   text
);

drop trigger if exists pages_set_updated_at on avocado_kiss.pages;
create trigger pages_set_updated_at before update on avocado_kiss.pages
  for each row execute function public.set_updated_at();

-- --- footer_settings: singleton (одна строка) --------------------------------
create table if not exists avocado_kiss.footer_settings (
  id                 uuid primary key default gen_random_uuid(),
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),
  tagline            text,
  copyright          text,
  made_with          text,   -- «Made with care · New York — Lisbon»
  instagram_url      text,
  pinterest_url      text,
  telegram_url       text,
  rss_url            text,
  newsletter_eyebrow text,   -- «The Culinary Dispatch»
  newsletter_title   text,
  newsletter_text    text
);

drop trigger if exists footer_settings_set_updated_at on avocado_kiss.footer_settings;
create trigger footer_settings_set_updated_at before update on avocado_kiss.footer_settings
  for each row execute function public.set_updated_at();

-- --- media: метаданные файлов медиа-менеджера (шаблон 0003 + 0021) -----------
create table if not exists avocado_kiss.media (
  id            uuid primary key default gen_random_uuid(),
  created_at    timestamptz not null default now(),
  original_name text not null,
  path          text not null unique,  -- плоский ключ в бакете avocado-kiss-photos
  mime          text,
  size          bigint,
  folder_id     uuid references avocado_kiss.admin_folders(id) on delete set null
);
create index if not exists media_folder_idx on avocado_kiss.media (folder_id);

-- --- Гранты на созданные таблицы ---------------------------------------------
grant select on all tables in schema avocado_kiss to anon;
grant all on all tables in schema avocado_kiss to authenticated, service_role;

-- --- RLS ---------------------------------------------------------------------
alter table avocado_kiss.categories      enable row level security;
alter table avocado_kiss.recipes         enable row level security;
alter table avocado_kiss.home_slots      enable row level security;
alter table avocado_kiss.pages           enable row level security;
alter table avocado_kiss.footer_settings enable row level security;
alter table avocado_kiss.media           enable row level security;
alter table avocado_kiss.admin_folders   enable row level security;

-- Публичное чтение: категории, страницы, футер — все строки; рецепты и слоты —
-- только опубликованные. media и admin_folders публичного чтения НЕ имеют.
drop policy if exists "Public read categories" on avocado_kiss.categories;
create policy "Public read categories" on avocado_kiss.categories
  for select using (true);

drop policy if exists "Public read published recipes" on avocado_kiss.recipes;
create policy "Public read published recipes" on avocado_kiss.recipes
  for select using (is_published = true);

drop policy if exists "Public read published home_slots" on avocado_kiss.home_slots;
create policy "Public read published home_slots" on avocado_kiss.home_slots
  for select using (is_published = true);

drop policy if exists "Public read pages" on avocado_kiss.pages;
create policy "Public read pages" on avocado_kiss.pages
  for select using (true);

drop policy if exists "Public read footer_settings" on avocado_kiss.footer_settings;
create policy "Public read footer_settings" on avocado_kiss.footer_settings
  for select using (true);

-- Запись во все таблицы — только админ из whitelist (шаблон 0017);
-- для media/admin_folders это и единственный доступ (включая select).
do $$
declare t text;
begin
  foreach t in array array[
    'categories','recipes','home_slots','pages','footer_settings'
  ]
  loop
    execute format('drop policy if exists "Admin write %1$s" on avocado_kiss.%1$s;', t);
    execute format(
      'create policy "Admin write %1$s" on avocado_kiss.%1$s for all to authenticated using (public.is_admin()) with check (public.is_admin());',
      t
    );
  end loop;
end $$;

drop policy if exists "Admin all media" on avocado_kiss.media;
create policy "Admin all media" on avocado_kiss.media
  for all to authenticated
  using (public.is_admin()) with check (public.is_admin());

drop policy if exists "Admin all admin_folders" on avocado_kiss.admin_folders;
create policy "Admin all admin_folders" on avocado_kiss.admin_folders
  for all to authenticated
  using (public.is_admin()) with check (public.is_admin());
```

- [ ] **Step 2: применить к base-one через Supabase MCP**

Вызвать `mcp__claude_ai_Supabase__apply_migration` (project ref `zwrkphynupdubevzwdzy`) с `name: "avocado_kiss_schema"` и содержимым файла. Файл уже лежит в репозитории — правило «файл миграции сразу в репо» соблюдено.

- [ ] **Step 3: проверить** — `mcp__claude_ai_Supabase__list_tables` (schema `avocado_kiss`): 7 таблиц.

- [ ] **Step 4: РУЧНОЙ ШАГ (пользователь, блокирует Task 5+)** — дашборд base-one → Project Settings → Data API → Exposed schemas → добавить `avocado_kiss`.

- [ ] **Step 5: Commit**

```bash
git add supabase && git commit -m "feat: avocado_kiss schema (recipes, categories, home_slots, pages, footer, media)"
```

---

### Task 4: Миграции 0002 (бакет) и 0003 (сиды) + картинки из макетов

**Files:**
- Create: `supabase/migrations/0002_storage_avocado_kiss_photos.sql`
- Create: `supabase/migrations/0003_seed_demo_content.sql`
- Create: `mockups/seed-images/*.jpg` (извлечение из макетов, в git не коммитить — добавить `mockups/seed-images/` в .gitignore)

- [ ] **Step 1: 0002_storage_avocado_kiss_photos.sql** (шаблон cozycorner 0018)

```sql
-- Storage Avocado Kiss: бакет на сайт. Чтение публичное; запись/изменение/удаление —
-- только админ из whitelist (public.is_admin()). Пути объектов — плоские ключи.

insert into storage.buckets (id, name, public)
values ('avocado-kiss-photos', 'avocado-kiss-photos', true)
on conflict (id) do update set public = true;

drop policy if exists "Public read avocado-kiss-photos" on storage.objects;
create policy "Public read avocado-kiss-photos"
  on storage.objects for select
  to anon, authenticated
  using (bucket_id = 'avocado-kiss-photos');

drop policy if exists "Admin insert avocado-kiss-photos" on storage.objects;
create policy "Admin insert avocado-kiss-photos"
  on storage.objects for insert
  to authenticated
  with check (bucket_id = 'avocado-kiss-photos' and public.is_admin());

drop policy if exists "Admin update avocado-kiss-photos" on storage.objects;
create policy "Admin update avocado-kiss-photos"
  on storage.objects for update
  to authenticated
  using (bucket_id = 'avocado-kiss-photos' and public.is_admin())
  with check (bucket_id = 'avocado-kiss-photos' and public.is_admin());

drop policy if exists "Admin delete avocado-kiss-photos" on storage.objects;
create policy "Admin delete avocado-kiss-photos"
  on storage.objects for delete
  to authenticated
  using (bucket_id = 'avocado-kiss-photos' and public.is_admin());
```

- [ ] **Step 2: 0003_seed_demo_content.sql** — демо-контент из макетов. Пути картинок = плоские ключи, файлы для загрузки готовит Step 4. Slug'и не задаём (автоген). Ключ картинки = slug рецепта + `.jpg`.

```sql
-- Демо-сид v1: контент из макетов mockups/*.html. Картинки загружаются в бакет
-- отдельно (ключи = <slug>.jpg). Один полный рецепт (zucchini) + карточки главной.

-- Категории навигации (порядок из шапки макета)
insert into avocado_kiss.categories (name, position) values
  ('Breakfast', 0), ('Salads', 1), ('Pasta', 2), ('Baking', 3),
  ('Desserts', 4), ('Vegan', 5), ('Seafood', 6), ('Drinks', 7)
on conflict (name) do nothing;

-- Полный рецепт из макета recipe
insert into avocado_kiss.recipes
  (title, excerpt, category, author_name, time_label, servings_label,
   hero_image_path, ingredients, steps, is_published)
values
  ('Roasted zucchini with white beans and kale',
   'A bright, hearty plate that turns a few pantry staples into a slow-cooked dinner worth lingering over.',
   'Vegan', 'Anna I.', '45 minutes', '4 servings',
   'roasted-zucchini-with-white-beans-and-kale.jpg',
   array[
     '3 medium zucchini, sliced into 1cm rounds',
     '400 g cooked white beans (cannellini), drained',
     '1 bunch curly kale, stems removed, torn',
     '3 cloves garlic, thinly sliced',
     '3 tbsp extra-virgin olive oil',
     '1 tbsp lemon juice',
     '1 tsp chili flakes',
     'Sea salt and freshly ground black pepper',
     'Shaved pecorino, to serve'
   ],
   array[
     'Preheat the oven to 220°C. Toss the zucchini with 2 tbsp olive oil, salt and pepper. Arrange in a single layer on a tray.',
     'Roast for 18–20 minutes, turning halfway, until deeply golden at the edges.',
     'Meanwhile, warm the remaining olive oil in a wide pan over medium heat. Add the garlic and chili flakes, cook 1 minute until fragrant.',
     'Add the kale and a splash of water. Cover and cook 4–5 minutes until just wilted.',
     'Stir in the white beans, season generously, and warm through for another 2 minutes.',
     'Fold the roasted zucchini into the pan, finish with lemon juice, and top with shaved pecorino before serving.'
   ],
   true);

-- Остальные рецепты-карточки главной (только данные для карточек; slug автоген,
-- hero_image_path = slugify(title).jpg — тем же правилом, что и ключи файлов)
insert into avocado_kiss.recipes
  (title, excerpt, category, author_name, time_label, hero_image_path, is_published)
values
  ('Our 20 best vegetarian recipes for the grill',
   'Because grilling isn''t just for sausages and burgers.',
   'Vegan', null, null, 'our-20-best-vegetarian-recipes-for-the-grill.jpg', true),
  ('Fridge full of carrots? Here''s what to cook',
   null, 'Salads', null, null, 'fridge-full-of-carrots-here-s-what-to-cook.jpg', true),
  ('Your April vegetable playbook starts here',
   null, 'Pasta', null, null, 'your-april-vegetable-playbook-starts-here.jpg', true),
  ('20 pies our readers can''t stop making', null, 'Baking', null, null, null, true),
  ('14 recipes that will make you love prunes', null, 'Desserts', null, null, null, true),
  ('20 spring dishes for longer evenings', null, 'Salads', null, null, null, true),
  ('What to bake this spring, straight from the test kitchen', null, 'Baking', null, null, null, true),
  ('Too many lemons? 35 recipes to make right now', null, 'Drinks', null, null, null, true),
  ('Wild sourdough loaf', null, 'Baking', 'Mark P.', '18 hours',
   'wild-sourdough-loaf.jpg', true),
  ('Citrus salad with fennel and mint', null, 'Salads', 'Anna I.', '10 minutes',
   'citrus-salad-with-fennel-and-mint.jpg', true),
  ('The best non-alcoholic wines, according to the experts',
   'What wine shop owners and sommeliers across the country recommend.',
   'Drinks', null, null, null, true),
  ('The best pasta salad recipes for a picnic or party',
   'From mayo-free versions to how much to make for a crowd — it''s all here.',
   'Pasta', null, null, null, true),
  ('Which dessert actually pairs with rosé?',
   'From fruit tarts to truffles — how to find a sweet match for summer wine.',
   'Desserts', null, null, null, true),
  ('22 sides for the grill that will steal the show',
   'Mac and cheese is the best part of any barbecue, and we''ll defend that.',
   'Vegan', null, null, '22-sides-for-the-grill-that-will-steal-the-show.jpg', true);

-- Слоты главной. Хелпер выбора id по slug — подзапросом.
insert into avocado_kiss.home_slots (slot, position, recipe_id, eyebrow, eyebrow_secondary, description)
select v.slot, v.position,
       (select id from avocado_kiss.recipes where slug = v.slug),
       v.eyebrow, v.eyebrow_secondary, v.description
from (values
  -- hero: 3 слайда (в макете один рецепт — берём три разных для живой карусели)
  ('hero', 0, 'roasted-zucchini-with-white-beans-and-kale', 'Recipe of the week', null, null),
  ('hero', 1, 'wild-sourdough-loaf', 'Recipe of the week', null, null),
  ('hero', 2, 'citrus-salad-with-fennel-and-mint', 'Recipe of the week', null, null),
  -- сетка Recipes
  ('grid_large', 0, 'our-20-best-vegetarian-recipes-for-the-grill', 'Grill', null, null),
  ('grid_medium', 0, 'fridge-full-of-carrots-here-s-what-to-cook', 'Seasonal', null, null),
  ('grid_medium', 1, 'your-april-vegetable-playbook-starts-here', 'Dinner', null, null),
  ('grid_list', 0, '20-pies-our-readers-can-t-stop-making', 'Baking', null, null),
  ('grid_list', 1, '14-recipes-that-will-make-you-love-prunes', 'Seasonal', null, null),
  ('grid_list', 2, '20-spring-dishes-for-longer-evenings', 'Spring', null, null),
  ('grid_list', 3, 'what-to-bake-this-spring-straight-from-the-test-kitchen', 'Test Kitchen', null, null),
  ('grid_list', 4, 'too-many-lemons-35-recipes-to-make-right-now', 'Citrus', null, null),
  -- широкие карточки
  ('wide', 0, 'wild-sourdough-loaf', 'Bread', null, null),
  ('wide', 1, 'citrus-salad-with-fennel-and-mint', 'Salads', null, null),
  -- Editor's Picks
  ('pick', 0, 'the-best-non-alcoholic-wines-according-to-the-experts', 'Life', 'Community', null),
  ('pick', 1, 'the-best-pasta-salad-recipes-for-a-picnic-or-party', 'Food', 'Trends & Season', null),
  ('pick', 2, 'which-dessert-actually-pairs-with-ros', 'Food', 'Drinks', null),
  ('pick_feature', 0, '22-sides-for-the-grill-that-will-steal-the-show', 'Food', 'Menu Planning', null)
) as v(slot, position, slug, eyebrow, eyebrow_secondary, description)
where exists (select 1 from avocado_kiss.recipes where slug = v.slug);

-- SEO главной
insert into avocado_kiss.pages (slug, seo_title, seo_description) values
  ('home',
   'Avocado Kiss — a culinary magazine',
   'Seasonal recipes, chef stories, and the aesthetics of the home kitchen. A new magazine for those who cook without rush.')
on conflict (slug) do nothing;

-- Футер + блок рассылки
insert into avocado_kiss.footer_settings
  (tagline, copyright, made_with,
   instagram_url, pinterest_url, telegram_url, rss_url,
   newsletter_eyebrow, newsletter_title, newsletter_text)
select
  'A culinary magazine about slow food, seasonal produce, and the aesthetics of the home kitchen. Founded in 2024.',
  '© 2024 Avocado Kiss Magazine',
  'Made with care · New York — Lisbon',
  'https://instagram.com', 'https://pinterest.com', 'https://telegram.org', '/rss',
  'The Culinary Dispatch',
  'Inspiration for the table — every Saturday.',
  'Recipes, chef stories, and seasonal finds — straight to your inbox, no noise or banners.'
where not exists (select 1 from avocado_kiss.footer_settings);
```

⚠️ Slug `which-dessert-actually-pairs-with-ros` — результат `public.slugify('Which dessert actually pairs with rosé?')`: функция режет не-латиницу, `é` выпадает. После применения сида проверить фактические slug'и запросом (Step 3) и, если какой-то отличается, поправить значения в `home_slots` тем же запросом по реальному slug.

- [ ] **Step 3: применить 0002 и 0003 через MCP** (`apply_migration`, names: `storage_avocado_kiss_photos`, `seed_demo_content`), затем проверить:

```sql
select slug from avocado_kiss.recipes order by created_at;
select slot, position, recipe_id is not null as ok from avocado_kiss.home_slots order by slot, position;
```
Expected: 15 рецептов, 17 слотов, все `ok = true`. Если слотов меньше — поправить slug'и в values-списке по фактическим и перезалить слоты.

- [ ] **Step 4: извлечь картинки из макетов** (скрипт одноразовый, файлы в git не коммитим):

```bash
cd /Users/mdr/Documents/Projects/workspace-one/avocado.kiss
echo "mockups/seed-images/" >> .gitignore
python3 - <<'EOF'
import re, base64, os, pathlib
out = pathlib.Path("mockups/seed-images"); out.mkdir(exist_ok=True)
def slugify(s):
    return re.sub(r'[^a-z0-9]+', '-', s.lower()).strip('-')
for name in ["mockups/avocado-kiss-home.html", "mockups/avocado-kiss-recipe.html"]:
    src = open(name, encoding='utf-8', errors='replace').read()
    # CSS-переменные SingleFile: --sf-img-N -> base64
    vars = dict(re.findall(r'--sf-img-(\d+):\s*url\("data:image/(?:jpeg|png|webp);base64,([^"]+)"\)', src))
    for m in re.finditer(r'<img[^>]*>', src):
        tag = m.group(0)
        alt = re.search(r'alt="?([^">]+)"?', tag)
        if not alt: continue
        fname = slugify(alt.group(1)) + ".jpg"
        data = None
        v = re.search(r'var\(--sf-img-(\d+)\)', tag)
        if v and v.group(1) in vars: data = vars[v.group(1)]
        else:
            s = re.search(r'src="?data:image/(?:jpeg|png|webp);base64,([^"\' >]+)', tag)
            if s: data = s.group(1)
        if data and not (out / fname).exists():
            (out / fname).write_bytes(base64.b64decode(data))
            print(fname)
EOF
ls mockups/seed-images
```
Expected: ~8 jpg-файлов, имена совпадают с `hero_image_path` из сида (сверить; расхождения — переименовать файл под значение в БД).

- [ ] **Step 5: РУЧНОЙ ШАГ (пользователь)** — дашборд → Storage → `avocado-kiss-photos` → загрузить все файлы из `mockups/seed-images/` в корень бакета (плоские ключи). До этого сайт рендерится с пустыми картинками — не блокирует разработку.

- [ ] **Step 6: Commit**

```bash
git add supabase .gitignore && git commit -m "feat: storage bucket + demo seed from mockups"
```

---

### Task 5: lib — типы, клиенты Supabase, загрузчики

**Files:**
- Create: `lib/types.ts`, `lib/supabase/client.ts`, `lib/supabase/server.ts`, `lib/images.ts`, `lib/content.ts`

- [ ] **Step 1: lib/types.ts**

```ts
export type Recipe = {
  id: string;
  created_at: string;
  updated_at: string;
  published_at: string; // сортировка лент (новые первыми)
  is_published: boolean;
  title: string;
  slug: string; // уникальный kebab-slug для /recipes/[slug] (автоген из title)
  excerpt: string | null; // анонс; фолбэк описаний карточек и seo_description
  category: string | null; // = categories.name (текст, без FK — паттерн products)
  author_name: string | null; // «by Mark P.»
  time_label: string | null; // «45 minutes» — готовая строка
  servings_label: string | null; // «4 servings» — готовая строка
  hero_image_path: string | null; // плоский ключ бакета или внешний URL
  ingredients: string[]; // порядок массива = порядок списка
  steps: string[]; // порядок массива = номера шагов 01, 02, …
  seo_title: string | null; // null = фолбэк title
  seo_description: string | null; // null = фолбэк excerpt
};

export type Category = {
  id: string;
  created_at: string;
  name: string; // отображаемое имя; рецепты ссылаются через recipes.category
  slug: string; // kebab-slug для /category/[slug] (автоген из name)
  position: number; // порядок ссылок в навигации шапки
  seo_title: string | null; // null = автоформула из name
  seo_description: string | null;
};

export const HOME_SLOTS = [
  "hero",
  "grid_large",
  "grid_medium",
  "grid_list",
  "wide",
  "pick",
  "pick_feature",
] as const;
export type HomeSlotName = (typeof HOME_SLOTS)[number];

export type HomeSlot = {
  id: string;
  slot: HomeSlotName;
  position: number;
  recipe_id: string;
  eyebrow: string | null; // null → категория рецепта
  eyebrow_secondary: string | null; // второй лейбл в picks («Food | Drinks»)
  description: string | null; // null → excerpt рецепта
  is_published: boolean;
  recipe: Recipe; // embed через FK home_slots.recipe_id → recipes
};

export type PageSeo = {
  id: string;
  slug: string;
  seo_title: string | null;
  seo_description: string | null;
  og_image_path: string | null;
};

export type FooterSettings = {
  id: string;
  tagline: string | null;
  copyright: string | null;
  made_with: string | null;
  instagram_url: string | null;
  pinterest_url: string | null;
  telegram_url: string | null;
  rss_url: string | null;
  newsletter_eyebrow: string | null;
  newsletter_title: string | null;
  newsletter_text: string | null;
};
```

- [ ] **Step 2: lib/supabase/client.ts и server.ts** (паттерн cozycorner, схема `avocado_kiss`)

```ts
// lib/supabase/client.ts
import { createClient } from "@supabase/supabase-js";

// Браузерный клиент Supabase. Публичный (publishable/anon) ключ безопасно отдавать
// в браузер — доступ ограничен RLS. Данные сайта — в схеме avocado_kiss.
export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  { db: { schema: "avocado_kiss" } },
);

// Тип клиента, настроенного на схему avocado_kiss (браузерный и серверный одинаковы).
export type DbClient = typeof supabase;
```

```ts
// lib/supabase/server.ts
import { createClient } from "@supabase/supabase-js";

// Серверный клиент для рендеринга в Server Components.
// Без сессий/куки — читаем только публичные данные под RLS.
export function createServerClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { auth: { persistSession: false }, db: { schema: "avocado_kiss" } },
  );
}
```

- [ ] **Step 3: lib/images.ts** (контракт §4 schema.md, аналог `resolveProductImage`)

```ts
/**
 * Путь картинки в БД — относительный плоский ключ бакета avocado-kiss-photos
 * ИЛИ внешний URL. Относительный ключ → публичный Storage-URL (оптимизируется
 * next/image по remotePatterns); внешний URL отдаётся с unoptimized: true.
 */
export function resolveRecipeImage(
  path: string | null,
): { url: string; unoptimized: boolean } | null {
  if (!path) return null;

  const base = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const publicPrefix = `${base}/storage/v1/object/public/`;

  if (!/^https?:\/\//i.test(path)) {
    return { url: `${publicPrefix}avocado-kiss-photos/${path}`, unoptimized: false };
  }

  const internal = Boolean(base) && path.startsWith(publicPrefix);
  return { url: path, unoptimized: !internal };
}
```

- [ ] **Step 4: lib/content.ts** — все загрузчики (SQL только здесь, не в компонентах)

```ts
import "server-only";
import type { DbClient } from "./supabase/client";
import type {
  Category,
  FooterSettings,
  HomeSlot,
  HomeSlotName,
  PageSeo,
  Recipe,
} from "./types";

/** Категории для навигации шапки и страниц категорий (порядок — position). */
export async function fetchCategories(client: DbClient): Promise<Category[]> {
  const { data, error } = await client
    .from("categories")
    .select("*")
    .order("position", { ascending: true })
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Category[];
}

export async function fetchCategoryBySlug(
  client: DbClient,
  slug: string,
): Promise<Category | null> {
  const { data, error } = await client
    .from("categories")
    .select("*")
    .eq("slug", slug)
    .maybeSingle();
  if (error) throw error;
  return data as Category | null;
}

/**
 * Слоты главной с embed-рецептом. !inner + eq по embed-таблице отбрасывает слоты,
 * чей рецепт не опубликован (RLS вернула бы null-embed, слот остался бы пустым).
 */
export async function fetchHomeSlots(
  client: DbClient,
): Promise<Record<HomeSlotName, HomeSlot[]>> {
  const { data, error } = await client
    .from("home_slots")
    .select("*, recipe:recipes!inner(*)")
    .eq("is_published", true)
    .eq("recipe.is_published", true)
    .order("position", { ascending: true });
  if (error) throw error;

  const grouped: Record<HomeSlotName, HomeSlot[]> = {
    hero: [], grid_large: [], grid_medium: [], grid_list: [],
    wide: [], pick: [], pick_feature: [],
  };
  for (const row of (data ?? []) as HomeSlot[]) grouped[row.slot]?.push(row);
  return grouped;
}

export async function fetchRecipeBySlug(
  client: DbClient,
  slug: string,
): Promise<Recipe | null> {
  const { data, error } = await client
    .from("recipes")
    .select("*")
    .eq("slug", slug)
    .maybeSingle();
  if (error) throw error;
  return data as Recipe | null;
}

/** Опубликованные рецепты категории, новые первыми (вторичный ключ id — детерминизм). */
export async function fetchRecipesByCategory(
  client: DbClient,
  categoryName: string,
): Promise<Recipe[]> {
  const { data, error } = await client
    .from("recipes")
    .select("*")
    .eq("category", categoryName)
    .order("published_at", { ascending: false })
    .order("id", { ascending: false });
  if (error) throw error;
  return (data ?? []) as Recipe[];
}

/** Все опубликованные slug'и — для generateStaticParams и sitemap. */
export async function fetchRecipeSlugs(
  client: DbClient,
): Promise<Pick<Recipe, "slug" | "updated_at">[]> {
  const { data, error } = await client
    .from("recipes")
    .select("slug, updated_at");
  if (error) throw error;
  return (data ?? []) as Pick<Recipe, "slug" | "updated_at">[];
}

export async function fetchPageSeo(
  client: DbClient,
  slug: string,
): Promise<PageSeo | null> {
  const { data, error } = await client
    .from("pages")
    .select("*")
    .eq("slug", slug)
    .maybeSingle();
  if (error) throw error;
  return data as PageSeo | null;
}

/** Singleton футера (одна строка). null — рендерим без текстов (безопасный фолбэк). */
export async function fetchFooterSettings(
  client: DbClient,
): Promise<FooterSettings | null> {
  const { data, error } = await client
    .from("footer_settings")
    .select("*")
    .limit(1)
    .maybeSingle();
  if (error) throw error;
  return data as FooterSettings | null;
}
```

- [ ] **Step 5: проверка и commit**

```bash
npm run build && npm run lint
git add lib && git commit -m "feat: types, supabase clients, content loaders"
```

---

### Task 6: Иконки, Header + CategoryNav + MobileMenu, Footer, layout

**Files:**
- Create: `components/icons/{MenuIcon,SearchIcon,ChevronLeftIcon,ChevronRightIcon,UsersIcon,ClockIcon}.tsx`
- Create: `components/Header.tsx` + `Header.module.css`
- Create: `components/MobileMenu.tsx` + `MobileMenu.module.css`
- Create: `components/Footer.tsx` + `Footer.module.css`
- Modify: `app/layout.tsx`

- [ ] **Step 1: иконки** — SVG-пути lucide из макетов, каждый компонент по одному образцу (все 6 файлов одинаковой формы, меняются только paths):

```tsx
// components/icons/SearchIcon.tsx
export default function SearchIcon({ className }: { className?: string }) {
  return (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none"
      stroke="currentColor" strokeWidth={1.5} strokeLinecap="round"
      strokeLinejoin="round" className={className} aria-hidden>
      <path d="m21 21-4.34-4.34" />
      <circle cx={11} cy={11} r={8} />
    </svg>
  );
}
```

Остальные — тот же каркас со своими путями:
- `MenuIcon`: `<path d="M4 5h16" /><path d="M4 12h16" /><path d="M4 19h16" />`
- `ChevronLeftIcon`: `<path d="m15 18-6-6 6-6" />`
- `ChevronRightIcon`: `<path d="m9 18 6-6-6-6" />`
- `UsersIcon`: `<path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2" /><path d="M16 3.128a4 4 0 0 1 0 7.744" /><path d="M22 21v-2a4 4 0 0 0-3-3.87" /><circle cx={9} cy={7} r={4} />`
- `ClockIcon`: `<circle cx={12} cy={12} r={10} /><path d="M12 6v6l4 2" />`

- [ ] **Step 2: components/Header.tsx** (RSC; категории приходят пропом из layout)

```tsx
import Link from "next/link";
import type { Category } from "@/lib/types";
import SearchIcon from "@/components/icons/SearchIcon";
import MobileMenu from "@/components/MobileMenu";
import styles from "./Header.module.css";

// Sticky-шапка из макета: поиск слева (≥md), логотип по центру, под шапкой —
// навигация категорий. Поиск в v1 декоративный (без обработчика).
export default function Header({ categories }: { categories: Category[] }) {
  return (
    <header className={styles.header}>
      <div className="shellWide">
        <div className={styles.bar}>
          <div className={styles.left}>
            <MobileMenu categories={categories} />
            <label className={styles.search}>
              <SearchIcon className={styles.searchIcon} />
              <input
                type="text"
                placeholder="Search"
                className={styles.searchInput}
              />
            </label>
          </div>
          <Link href="/" className={styles.logo}>
            AVOCADO KISS
          </Link>
          <div className={styles.right}>
            <button aria-label="Search" className={styles.iconButton}>
              <SearchIcon className={styles.iconSmall} />
            </button>
          </div>
        </div>
        <nav className={styles.nav}>
          {categories.map((c) => (
            <Link key={c.id} href={`/category/${c.slug}`} className={styles.navLink}>
              {c.name}
            </Link>
          ))}
        </nav>
      </div>
    </header>
  );
}
```

- [ ] **Step 3: components/Header.module.css**

```css
.header {
  position: sticky;
  top: 0;
  z-index: 40;
  border-bottom: 1px solid var(--rule);
  background: color-mix(in srgb, var(--background) 90%, transparent);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
}

.bar {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  height: 4rem;
}

.left {
  display: flex;
  align-items: center;
  justify-content: flex-start;
}

.right {
  display: flex;
  align-items: center;
  justify-content: flex-end;
}

.search {
  display: none;
}

.logo {
  white-space: nowrap;
  text-align: center;
  font-size: var(--text-base);
  font-weight: 700;
  letter-spacing: var(--tracking-logo);
}

.iconButton {
  display: grid;
  place-items: center;
  height: 2.25rem;
  width: 2.25rem;
  border-radius: 9999px;
}

.iconSmall {
  height: 1.25rem;
  width: 1.25rem;
}

.nav {
  display: none;
}

@media (min-width: 640px) {
  .logo {
    font-size: var(--text-lg);
  }
}

@media (min-width: 768px) {
  .bar {
    height: 5rem;
  }
  .logo {
    font-size: var(--text-3xl);
  }
  .iconButton {
    display: none;
  }
  .search {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    height: 2.5rem;
    width: 100%;
    max-width: 260px;
    padding-inline: 1rem;
    border-radius: 9999px;
    background: var(--muted);
    color: color-mix(in srgb, var(--foreground) 60%, transparent);
  }
  .searchIcon {
    height: 1rem;
    width: 1rem;
    flex-shrink: 0;
  }
  .searchInput {
    width: 100%;
    background: transparent;
    border: none;
    outline: none;
    font-size: var(--text-sm);
    color: var(--foreground);
  }
  .searchInput::placeholder {
    color: color-mix(in srgb, var(--foreground) 50%, transparent);
  }
  .nav {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 2.5rem;
    height: 3rem;
    border-top: 1px solid var(--rule);
  }
  .navLink {
    white-space: nowrap;
    font-size: 13px;
    text-transform: uppercase;
    letter-spacing: var(--tracking-logo);
    color: color-mix(in srgb, var(--foreground) 80%, transparent);
    transition: color 0.15s var(--ease-default);
  }
  .navLink:hover {
    color: var(--accent);
  }
}
```

- [ ] **Step 4: components/MobileMenu.tsx + module.css** (клиентский; простое раскрытие — выезжающий диалог вне объёма v1 по спеке)

```tsx
"use client";

import { useState } from "react";
import Link from "next/link";
import type { Category } from "@/lib/types";
import MenuIcon from "@/components/icons/MenuIcon";
import styles from "./MobileMenu.module.css";

// Мобильная навигация: кнопка-бургер раскрывает список категорий под шапкой.
export default function MobileMenu({ categories }: { categories: Category[] }) {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button
        aria-label="Open menu"
        aria-expanded={open}
        className={styles.button}
        onClick={() => setOpen((v) => !v)}
      >
        <MenuIcon className={styles.icon} />
      </button>
      {open && (
        <nav className={styles.panel}>
          {categories.map((c) => (
            <Link
              key={c.id}
              href={`/category/${c.slug}`}
              className={styles.link}
              onClick={() => setOpen(false)}
            >
              {c.name}
            </Link>
          ))}
        </nav>
      )}
    </>
  );
}
```

```css
/* components/MobileMenu.module.css */
.button {
  display: grid;
  place-items: center;
  height: 2.25rem;
  width: 2.25rem;
  border-radius: 9999px;
}

.icon {
  height: 1.25rem;
  width: 1.25rem;
}

.panel {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 1.25rem 1rem 1.5rem;
  background: var(--background);
  border-bottom: 1px solid var(--rule);
}

.link {
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: var(--tracking-logo);
  color: color-mix(in srgb, var(--foreground) 80%, transparent);
}

@media (min-width: 768px) {
  .button,
  .panel {
    display: none;
  }
}
```

`.panel` позиционируется от шапки: в `Header.module.css` `.header` уже sticky (containing block — добавить `position: relative` не нужно, panel растянется от header из-за absolute + top:100% — проверить визуально; если панель легла неверно, добавить `position: relative` на `.header`).

- [ ] **Step 5: components/Footer.tsx + module.css**

```tsx
import Link from "next/link";
import type { FooterSettings } from "@/lib/types";
import styles from "./Footer.module.css";

// Футер из макета: бренд + tagline (2 колонки), «Magazine» (v1 — без адресатов,
// ссылки на главную), «Follow» (URL из настроек), нижняя строка © + made with.
export default function Footer({ settings }: { settings: FooterSettings | null }) {
  const follow = [
    { label: "Instagram", href: settings?.instagram_url },
    { label: "Pinterest", href: settings?.pinterest_url },
    { label: "Telegram", href: settings?.telegram_url },
    { label: "RSS", href: settings?.rss_url },
  ].filter((l): l is { label: string; href: string } => Boolean(l.href));

  return (
    <footer className={styles.footer}>
      <div className={`shell ${styles.grid}`}>
        <div className={styles.brand}>
          <span className={styles.logo}>
            Avocado Kiss<span className={styles.dot}>.</span>
          </span>
          {settings?.tagline && <p className={styles.tagline}>{settings.tagline}</p>}
        </div>
        <div>
          <span className="eyebrow">Magazine</span>
          <ul className={styles.links}>
            {["About", "Contributors", "Contact", "Issue Archive"].map((l) => (
              <li key={l}>
                <Link href="/" className={styles.link}>{l}</Link>
              </li>
            ))}
          </ul>
        </div>
        {follow.length > 0 && (
          <div>
            <span className="eyebrow">Follow</span>
            <ul className={styles.links}>
              {follow.map((l) => (
                <li key={l.label}>
                  <a href={l.href} className={styles.link}>{l.label}</a>
                </li>
              ))}
            </ul>
          </div>
        )}
      </div>
      <div className={styles.bottomRule}>
        <div className={`shell ${styles.bottom}`}>
          {settings?.copyright && <span>{settings.copyright}</span>}
          {settings?.made_with && <span>{settings.made_with}</span>}
        </div>
      </div>
    </footer>
  );
}
```

```css
/* components/Footer.module.css */
.footer {
  border-top: 1px solid var(--rule);
  background: var(--background);
}

.grid {
  display: grid;
  gap: 2.5rem;
  padding-block: 3rem;
}

.logo {
  font-size: var(--text-2xl);
  letter-spacing: var(--tracking-tight);
}

.dot {
  color: var(--accent);
}

.tagline {
  margin-top: 1rem;
  max-width: 24rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: var(--muted-foreground);
}

.links {
  margin-top: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  font-size: var(--text-sm);
}

.link:hover {
  color: var(--accent);
}

.bottomRule {
  border-top: 1px solid var(--rule);
}

.bottom {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding-block: 1.25rem;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: var(--tracking-eyebrow);
  color: var(--muted-foreground);
}

@media (min-width: 768px) {
  .grid {
    gap: 3rem;
    padding-block: 4rem;
  }
  .bottom {
    flex-direction: row;
    font-size: 11px;
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(4, 1fr);
  }
  .brand {
    grid-column: span 2;
  }
  .logo {
    font-size: var(--text-3xl);
  }
}
```

- [ ] **Step 6: layout подключает Header/Footer с данными**

```tsx
// app/layout.tsx — заменить целиком
import type { Metadata } from "next";
import Header from "@/components/Header";
import Footer from "@/components/Footer";
import { createServerClient } from "@/lib/supabase/server";
import { fetchCategories, fetchFooterSettings } from "@/lib/content";
import "./globals.css";

// Шрифт проекта — Fira Sans (OFL), self-hosted через @font-face в globals.css.

export const revalidate = 60; // ISR: правки контента из админки видны в течение минуты

export const metadata: Metadata = {
  title: {
    default: "Avocado Kiss — a culinary magazine",
    template: "%s — Avocado Kiss",
  },
  description:
    "Seasonal recipes, chef stories, and the aesthetics of the home kitchen. A new magazine for those who cook without rush.",
  openGraph: {
    title: "Avocado Kiss — a culinary magazine",
    description:
      "Seasonal recipes, chef stories, and the aesthetics of the home kitchen.",
    type: "website",
  },
};

export default async function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  const client = createServerClient();
  const [categories, footer] = await Promise.all([
    fetchCategories(client),
    fetchFooterSettings(client),
  ]);

  return (
    <html lang="en">
      <body>
        <Header categories={categories} />
        {children}
        <Footer settings={footer} />
      </body>
    </html>
  );
}
```

- [ ] **Step 7: проверка** — `npm run dev`, открыть http://localhost:3000: шапка с 8 категориями из БД, футер с текстами из БД. Затем:

```bash
npm run build && npm run lint
git add -A && git commit -m "feat: header, category nav, mobile menu, footer"
```

---

### Task 7: Reveal (GSAP) + NewsletterBlock

**Files:**
- Create: `components/Reveal.tsx` (порт из cozycorner как есть)
- Create: `components/NewsletterBlock.tsx` + `NewsletterBlock.module.css`

- [ ] **Step 1: components/Reveal.tsx** — скопировать 1:1 из `../cozycorner/components/Reveal.tsx` (scroll-reveal с уважением prefers-reduced-motion; контент рендерится на сервере, GSAP анимирует после гидрации).

```bash
cp ../cozycorner/components/Reveal.tsx components/Reveal.tsx
```

- [ ] **Step 2: components/NewsletterBlock.tsx** (форма декоративная по спеке: submit гасится, никуда не пишет)

```tsx
import type { FooterSettings } from "@/lib/types";
import NewsletterForm from "@/components/NewsletterForm";
import styles from "./NewsletterBlock.module.css";

// Инверсный блок рассылки «The Culinary Dispatch». Тексты — из footer_settings;
// пустые поля не рендерятся. Форма в v1 декоративная (отдельный клиентский компонент).
export default function NewsletterBlock({
  settings,
}: {
  settings: FooterSettings | null;
}) {
  if (!settings?.newsletter_title) return null;
  return (
    <section className={styles.section}>
      <div className={styles.inner}>
        {settings.newsletter_eyebrow && (
          <span className={styles.eyebrow}>{settings.newsletter_eyebrow}</span>
        )}
        <h2 className={styles.title}>{settings.newsletter_title}</h2>
        {settings.newsletter_text && (
          <p className={styles.text}>{settings.newsletter_text}</p>
        )}
        <NewsletterForm />
      </div>
    </section>
  );
}
```

`components/NewsletterForm.tsx` (клиентский, только preventDefault):

```tsx
"use client";

import styles from "./NewsletterBlock.module.css";

// v1: форма декоративная — сабмит никуда не пишет (сбор e-mail — отдельной задачей).
export default function NewsletterForm() {
  return (
    <form className={styles.form} onSubmit={(e) => e.preventDefault()}>
      <input
        type="email"
        required
        placeholder="you@email.com"
        className={styles.input}
      />
      <button type="submit" className={styles.submit}>
        Subscribe →
      </button>
    </form>
  );
}
```

- [ ] **Step 3: components/NewsletterBlock.module.css**

```css
.section {
  background: var(--foreground);
  color: var(--background);
}

.inner {
  max-width: 48rem;
  margin: 0 auto;
  padding: 5rem 1rem;
  text-align: center;
}

.eyebrow {
  display: inline-block;
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.3em;
  color: color-mix(in srgb, var(--background) 60%, transparent);
}

.title {
  margin-top: 1rem;
  font-size: var(--text-4xl);
  font-weight: 400;
  line-height: 1.05;
  letter-spacing: var(--tracking-tight);
}

.text {
  margin: 1rem auto 0;
  max-width: 28rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: color-mix(in srgb, var(--background) 65%, transparent);
}

.form {
  margin: 2rem auto 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  max-width: 32rem;
  border-bottom: 1px solid color-mix(in srgb, var(--background) 25%, transparent);
  padding-bottom: 0.75rem;
}

.input {
  flex: 1;
  width: 100%;
  background: transparent;
  border: none;
  outline: none;
  font-size: var(--text-base);
  color: var(--background);
}

.input::placeholder {
  color: color-mix(in srgb, var(--background) 40%, transparent);
}

.submit {
  flex-shrink: 0;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: var(--tracking-eyebrow);
  transition: color 0.15s var(--ease-default);
}

.submit:hover {
  color: var(--accent);
}

@media (min-width: 768px) {
  .inner {
    padding: 7rem 1.5rem;
  }
  .eyebrow {
    font-size: 11px;
  }
  .title {
    margin-top: 1.5rem;
    font-size: var(--text-5xl);
  }
  .text {
    margin-top: 1.5rem;
  }
  .form {
    margin-top: 3rem;
    flex-direction: row;
  }
}

@media (min-width: 1024px) {
  .inner {
    padding-block: 9rem;
  }
  .title {
    font-size: var(--text-6xl);
  }
}
```

- [ ] **Step 4: проверка и commit**

```bash
npm run build && npm run lint
git add components && git commit -m "feat: reveal animation + newsletter block"
```

---

### Task 8: HeroCarousel (GSAP)

**Files:**
- Create: `components/HeroCarousel.tsx` + `HeroCarousel.module.css`

- [ ] **Step 1: components/HeroCarousel.tsx**

```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import Image from "next/image";
import Link from "next/link";
import gsap from "gsap";
import ChevronLeftIcon from "@/components/icons/ChevronLeftIcon";
import ChevronRightIcon from "@/components/icons/ChevronRightIcon";
import styles from "./HeroCarousel.module.css";

export type HeroSlide = {
  id: string;
  title: string;
  href: string;
  eyebrow: string;
  image: { url: string; unoptimized: boolean } | null;
};

const AUTOPLAY_MS = 6000;

/**
 * Hero-карусель главной: слайды растянуты друг под другом (absolute), активный
 * проявляется GSAP-кроссфейдом. Автоплей с паузой при hover; стрелки + точки.
 * prefers-reduced-motion — мгновенное переключение без анимации.
 */
export default function HeroCarousel({ slides }: { slides: HeroSlide[] }) {
  const [index, setIndex] = useState(0);
  const refs = useRef<(HTMLDivElement | null)[]>([]);
  const hovered = useRef(false);

  const goTo = (next: number) => {
    setIndex((next + slides.length) % slides.length);
  };

  useEffect(() => {
    const reduced = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
    refs.current.forEach((el, i) => {
      if (!el) return;
      gsap.to(el, {
        autoAlpha: i === index ? 1 : 0,
        duration: reduced ? 0 : 0.9,
        ease: "power2.inOut",
        overwrite: true,
      });
    });
  }, [index, slides.length]);

  useEffect(() => {
    if (slides.length < 2) return;
    const t = setInterval(() => {
      if (!hovered.current) setIndex((i) => (i + 1) % slides.length);
    }, AUTOPLAY_MS);
    return () => clearInterval(t);
  }, [slides.length]);

  if (slides.length === 0) return null;

  return (
    <section className={`shellWide ${styles.section}`}>
      <div
        className={styles.frame}
        onMouseEnter={() => (hovered.current = true)}
        onMouseLeave={() => (hovered.current = false)}
      >
        <div className={styles.viewport}>
          {slides.map((s, i) => (
            <div
              key={s.id}
              ref={(el) => { refs.current[i] = el; }}
              className={styles.slide}
              style={i === 0 ? undefined : { opacity: 0, visibility: "hidden" }}
            >
              <Link href={s.href} aria-label={s.title} className={styles.slideLink} />
              {s.image && (
                <Image
                  src={s.image.url}
                  alt={s.title}
                  fill
                  priority={i === 0}
                  sizes="(min-width: 1600px) 1504px, 100vw"
                  unoptimized={s.image.unoptimized}
                  className={styles.image}
                />
              )}
              <div className={styles.gradient} />
              <div className={styles.caption}>
                <div className={styles.captionInner}>
                  <span className={styles.eyebrow}>{s.eyebrow}</span>
                  <h1 className={styles.title}>{s.title}</h1>
                </div>
              </div>
            </div>
          ))}
        </div>
        {slides.length > 1 && (
          <div className={styles.controls}>
            <button aria-label="Previous" className={styles.arrow} onClick={() => goTo(index - 1)}>
              <ChevronLeftIcon className={styles.arrowIcon} />
            </button>
            <div className={styles.dots}>
              {slides.map((s, i) => (
                <button
                  key={s.id}
                  aria-label={`Slide ${i + 1}`}
                  className={i === index ? styles.dotActive : styles.dot}
                  onClick={() => goTo(i)}
                />
              ))}
            </div>
            <button aria-label="Next" className={styles.arrow} onClick={() => goTo(index + 1)}>
              <ChevronRightIcon className={styles.arrowIcon} />
            </button>
          </div>
        )}
      </div>
    </section>
  );
}
```

- [ ] **Step 2: components/HeroCarousel.module.css**

```css
.section {
  padding-top: 1.5rem;
}

.frame {
  position: relative;
  overflow: hidden;
  border-radius: var(--radius);
}

.viewport {
  position: relative;
  aspect-ratio: 4 / 3;
  width: 100%;
}

.slide {
  position: absolute;
  inset: 0;
}

.slideLink {
  position: absolute;
  inset: 0;
  z-index: 10;
}

.image {
  object-fit: cover;
}

.gradient {
  display: none;
}

.caption {
  display: none;
}

.controls {
  position: absolute;
  bottom: 0.75rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 20;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.arrow {
  display: grid;
  place-items: center;
  height: 1.75rem;
  width: 1.75rem;
  border-radius: 9999px;
  background: color-mix(in srgb, var(--background) 80%, transparent);
  color: var(--foreground);
  transition: background 0.15s var(--ease-default);
}

.arrow:hover {
  background: var(--background);
}

.arrowIcon {
  height: 0.875rem;
  width: 0.875rem;
}

.dots {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.dot,
.dotActive {
  height: 0.375rem;
  border-radius: 9999px;
  transition: all 0.15s var(--ease-default);
}

.dot {
  width: 0.375rem;
  background: color-mix(in srgb, var(--background) 60%, transparent);
}

.dotActive {
  width: 1.5rem;
  background: var(--background);
}

@media (min-width: 768px) {
  .section {
    padding-top: 2rem;
  }
  .viewport {
    aspect-ratio: 16 / 9;
  }
  .gradient {
    display: block;
    position: absolute;
    inset: 0;
    background: linear-gradient(
      to right,
      rgb(0 0 0 / 0.45),
      rgb(0 0 0 / 0.15) 50%,
      transparent
    );
  }
  .caption {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: flex-end;
  }
  .captionInner {
    width: 100%;
    max-width: 42rem;
    padding: 0 2rem 4rem;
    color: var(--background);
  }
  .eyebrow {
    display: inline-block;
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 0.28em;
    color: color-mix(in srgb, var(--background) 85%, transparent);
  }
  .title {
    margin-top: 1.25rem;
    font-size: clamp(2rem, 4.5vw, 4.5rem);
    font-weight: 500;
    line-height: 1.02;
    letter-spacing: var(--tracking-tight);
    text-wrap: balance;
  }
  .controls {
    bottom: 1.5rem;
    gap: 1rem;
  }
  .arrow {
    height: 2rem;
    width: 2rem;
  }
  .arrowIcon {
    height: 1rem;
    width: 1rem;
  }
}

@media (min-width: 1024px) {
  .captionInner {
    padding: 0 4rem 5rem;
  }
}
```

- [ ] **Step 3: проверка и commit** (компонент подключается в Task 9; сейчас — только компиляция)

```bash
npm run build && npm run lint
git add components && git commit -m "feat: hero carousel with GSAP crossfade"
```

---

### Task 9: Карточки, Editor's Picks, сборка главной

**Files:**
- Create: `components/RecipeCard.tsx` + `RecipeCard.module.css`
- Create: `components/EditorsPicks.tsx` + `EditorsPicks.module.css`
- Create: `app/page.module.css`
- Modify: `app/page.tsx`

- [ ] **Step 1: components/RecipeCard.tsx** — один компонент, 4 варианта карточек главной (large / medium / list / wide). Фолбэки слота: eyebrow → категория, description → excerpt.

```tsx
import Image from "next/image";
import Link from "next/link";
import type { Recipe } from "@/lib/types";
import { resolveRecipeImage } from "@/lib/images";
import styles from "./RecipeCard.module.css";

export type RecipeCardVariant = "large" | "medium" | "list" | "wide";

type Props = {
  recipe: Recipe;
  variant: RecipeCardVariant;
  eyebrow?: string | null; // оверрайд из слота; null → категория рецепта
  description?: string | null; // оверрайд из слота; null → excerpt
};

export default function RecipeCard({ recipe, variant, eyebrow, description }: Props) {
  const image = resolveRecipeImage(recipe.hero_image_path);
  const href = `/recipes/${recipe.slug}`;
  const label = eyebrow ?? recipe.category;

  if (variant === "list") {
    return (
      <li className={styles.listItem}>
        <Link href={href} className={styles.listLink}>
          {label && <span className="eyebrow eyebrowAccent">{label}</span>}
          <h3 className={styles.listTitle}>{recipe.title}</h3>
        </Link>
      </li>
    );
  }

  const picture = (
    <div className={styles.imageWrap}>
      {image && (
        <Image
          src={image.url}
          alt={recipe.title}
          width={960}
          height={variant === "medium" ? 960 : variant === "wide" ? 1200 : 768}
          unoptimized={image.unoptimized}
          className={`${styles.image} ${
            variant === "large" ? styles.ratioLarge
            : variant === "medium" ? styles.ratioSquare
            : styles.ratioPortrait
          }`}
        />
      )}
    </div>
  );

  if (variant === "wide") {
    return (
      <article className={styles.card}>
        <Link href={href} className={styles.wideGrid}>
          {picture}
          <div>
            {label && <span className="eyebrow eyebrowAccent">{label}</span>}
            <h3 className={styles.wideTitle}>{recipe.title}</h3>
            <div className={styles.meta}>
              {recipe.author_name && <span>by {recipe.author_name}</span>}
              {recipe.author_name && recipe.time_label && (
                <span className={styles.metaDot} />
              )}
              {recipe.time_label && <span>{recipe.time_label}</span>}
            </div>
          </div>
        </Link>
      </article>
    );
  }

  if (variant === "medium") {
    return (
      <article className={styles.card}>
        <Link href={href}>
          {picture}
          {label && (
            <span className={`eyebrow eyebrowAccent ${styles.mediumEyebrow}`}>
              {label}
            </span>
          )}
          <h3 className={styles.mediumTitle}>{recipe.title}</h3>
        </Link>
      </article>
    );
  }

  return (
    <article className={styles.card}>
      <Link href={href}>
        {picture}
        <div className={styles.largeBody}>
          {label && <span className="eyebrow eyebrowAccent">{label}</span>}
          <h3 className={styles.largeTitle}>{recipe.title}</h3>
          {description && <p className={styles.largeText}>{description}</p>}
        </div>
      </Link>
    </article>
  );
}
```

- [ ] **Step 2: components/RecipeCard.module.css**

```css
.card {
  cursor: pointer;
}

.imageWrap {
  overflow: hidden;
  border-radius: var(--radius);
}

.image {
  width: 100%;
  object-fit: cover;
  transition: transform var(--dur-hover-zoom) var(--ease-default);
}

.card:hover .image {
  transform: scale(1.03);
}

.ratioLarge {
  aspect-ratio: 5 / 4;
}

.ratioSquare {
  aspect-ratio: 1 / 1;
}

.ratioPortrait {
  aspect-ratio: 4 / 5;
}

/* --- large: большая карточка сетки, центрированный текст --- */
.largeBody {
  margin-top: 1.25rem;
  text-align: center;
}

.largeTitle {
  margin-top: 0.5rem;
  font-size: var(--text-xl);
  font-weight: 400;
  line-height: 1.05;
  letter-spacing: var(--tracking-tight);
  text-wrap: balance;
}

.largeText {
  margin: 0.75rem auto 0;
  max-width: 28rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: var(--muted-foreground);
}

/* --- medium: квадратная карточка --- */
.mediumEyebrow {
  display: block;
  margin-top: 1rem;
}

.mediumTitle {
  margin-top: 0.25rem;
  font-size: var(--text-lg);
  font-weight: 400;
  line-height: var(--leading-snug);
  letter-spacing: var(--tracking-tight);
}

/* --- list: текстовый пункт с линейкой --- */
.listItem {
  border-bottom: 1px solid var(--rule);
}

.listItem:last-child {
  border-bottom: none;
}

.listLink {
  display: block;
  padding-block: 1rem;
}

.listTitle {
  margin-top: 0.25rem;
  font-size: var(--text-lg);
  font-weight: 400;
  line-height: var(--leading-snug);
  letter-spacing: var(--tracking-tight);
  transition: color 0.15s var(--ease-default);
}

.listLink:hover .listTitle {
  color: var(--accent);
}

/* --- wide: фото + текст рядом --- */
.wideGrid {
  display: grid;
  gap: 1rem;
}

.wideTitle {
  margin-top: 0.5rem;
  font-size: var(--text-xl);
  font-weight: 400;
  line-height: 1.25;
  letter-spacing: var(--tracking-tight);
}

.meta {
  margin-top: 1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  font-size: var(--text-xs);
  color: var(--muted-foreground);
}

.metaDot {
  height: 0.25rem;
  width: 0.25rem;
  border-radius: 9999px;
  background: color-mix(in srgb, var(--muted-foreground) 60%, transparent);
}

@media (min-width: 640px) {
  .wideGrid {
    grid-template-columns: 1.1fr 1fr;
    align-items: center;
  }
}

@media (min-width: 768px) {
  .largeBody {
    margin-top: 1.75rem;
  }
  .largeTitle {
    margin-top: 0.75rem;
    font-size: 2rem;
  }
  .largeText {
    margin-top: 1rem;
  }
  .mediumEyebrow {
    margin-top: 1.25rem;
  }
  .mediumTitle {
    margin-top: 0.5rem;
    font-size: var(--text-xl);
  }
  .listLink {
    padding-block: 1.5rem;
  }
  .listTitle {
    margin-top: 0.5rem;
    font-size: var(--text-xl);
  }
  .wideGrid {
    gap: 1.5rem;
  }
  .wideTitle {
    margin-top: 0.75rem;
    font-size: var(--text-2xl);
  }
  .meta {
    margin-top: 1.5rem;
    gap: 1.5rem;
  }
}

@media (min-width: 1024px) {
  .largeTitle {
    font-size: 2.4rem;
  }
}
```

- [ ] **Step 3: components/EditorsPicks.tsx**

```tsx
import Image from "next/image";
import Link from "next/link";
import type { HomeSlot } from "@/lib/types";
import { resolveRecipeImage } from "@/lib/images";
import styles from "./EditorsPicks.module.css";

// Секция Editor's Picks: слева нумерованный список (pick), справа sticky-карточка
// (pick_feature). Двойной лейбл «Food | Drinks» — eyebrow + eyebrow_secondary слота.
export default function EditorsPicks({
  picks,
  feature,
}: {
  picks: HomeSlot[];
  feature: HomeSlot | null;
}) {
  if (picks.length === 0 && !feature) return null;
  const featureImage = feature
    ? resolveRecipeImage(feature.recipe.hero_image_path)
    : null;

  return (
    <section className={styles.section}>
      <div className={`shell ${styles.grid}`}>
        <div>
          <h2 className={styles.heading}>Editor&apos;s Picks</h2>
          <ol className={styles.list}>
            {picks.map((p, i) => (
              <li key={p.id} className={styles.item}>
                <Link href={`/recipes/${p.recipe.slug}`} className={styles.itemGrid}>
                  <span className={styles.number}>{i + 1}</span>
                  <div>
                    <Labels slot={p} />
                    <h3 className={styles.itemTitle}>{p.recipe.title}</h3>
                    {(p.description ?? p.recipe.excerpt) && (
                      <p className={styles.itemText}>
                        {p.description ?? p.recipe.excerpt}
                      </p>
                    )}
                  </div>
                </Link>
              </li>
            ))}
          </ol>
        </div>
        {feature && (
          <div className={styles.feature}>
            <Link href={`/recipes/${feature.recipe.slug}`}>
              <div className={styles.featureImageWrap}>
                {featureImage && (
                  <Image
                    src={featureImage.url}
                    alt={feature.recipe.title}
                    width={1200}
                    height={1500}
                    unoptimized={featureImage.unoptimized}
                    className={styles.featureImage}
                  />
                )}
              </div>
              <div className={styles.featureBody}>
                <Labels slot={feature} />
                <h3 className={styles.featureTitle}>{feature.recipe.title}</h3>
                {(feature.description ?? feature.recipe.excerpt) && (
                  <p className={styles.featureText}>
                    {feature.description ?? feature.recipe.excerpt}
                  </p>
                )}
              </div>
            </Link>
          </div>
        )}
      </div>
    </section>
  );
}

function Labels({ slot }: { slot: HomeSlot }) {
  const primary = slot.eyebrow ?? slot.recipe.category;
  if (!primary) return null;
  return (
    <div className={styles.labels}>
      <span>{primary}</span>
      {slot.eyebrow_secondary && (
        <>
          <span className={styles.labelDivider}>|</span>
          <span className={styles.labelSecondary}>{slot.eyebrow_secondary}</span>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 4: components/EditorsPicks.module.css**

```css
.section {
  border-top: 1px solid var(--rule);
  background: var(--paper);
}

.grid {
  display: grid;
  gap: 2.5rem;
  padding-block: 4rem;
}

.heading {
  font-size: var(--text-4xl);
  font-weight: 400;
  letter-spacing: var(--tracking-tight);
}

.list {
  margin-top: 2.5rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.item {
  border-top: 1px solid var(--rule);
  cursor: pointer;
}

.itemGrid {
  display: grid;
  grid-template-columns: 40px 1fr;
  gap: 1rem;
  padding-top: 1.5rem;
}

.number {
  font-size: var(--text-3xl);
  color: color-mix(in srgb, var(--foreground) 35%, transparent);
}

.labels {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: var(--tracking-eyebrow);
}

.labelDivider {
  color: color-mix(in srgb, var(--foreground) 30%, transparent);
}

.labelSecondary {
  color: var(--muted-foreground);
}

.itemTitle {
  margin-top: 0.75rem;
  font-size: var(--text-xl);
  font-weight: 400;
  line-height: 1.15;
  letter-spacing: var(--tracking-tight);
  transition: color 0.15s var(--ease-default);
}

.item:hover .itemTitle {
  color: var(--accent);
}

.itemText {
  margin-top: 0.5rem;
  max-width: 28rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: var(--muted-foreground);
}

.featureImageWrap {
  overflow: hidden;
  border-radius: var(--radius);
}

.featureImage {
  width: 100%;
  aspect-ratio: 4 / 5;
  object-fit: cover;
}

.featureBody {
  margin-top: 1.75rem;
}

.featureBody .labels {
  font-size: 10px;
}

.featureTitle {
  margin-top: 1rem;
  font-size: var(--text-3xl);
  font-weight: 400;
  line-height: 1.1;
  letter-spacing: var(--tracking-tight);
}

.featureText {
  margin-top: 1rem;
  max-width: 28rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: var(--muted-foreground);
}

@media (min-width: 768px) {
  .grid {
    gap: 4rem;
    padding-block: 6rem;
  }
  .heading {
    font-size: var(--text-5xl);
  }
  .list {
    margin-top: 3.5rem;
    gap: 3rem;
  }
  .itemGrid {
    grid-template-columns: 72px 1fr;
    gap: 1.5rem;
    padding-top: 2.5rem;
  }
  .number {
    font-size: var(--text-5xl);
  }
  .labels {
    gap: 0.75rem;
    font-size: 10px;
  }
  .itemTitle {
    margin-top: 1rem;
    font-size: var(--text-2xl);
  }
  .itemText {
    margin-top: 1rem;
  }
  .featureTitle {
    font-size: 2.2rem;
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: 1fr 1fr;
    gap: 5rem;
    padding-block: 8rem;
  }
  .heading {
    font-size: var(--text-6xl);
  }
  .itemTitle {
    font-size: 1.7rem;
  }
  .feature {
    position: sticky;
    top: 6rem;
    align-self: start;
  }
}
```

- [ ] **Step 5: app/page.tsx — сборка главной**

```tsx
import type { Metadata } from "next";
import RecipeCard from "@/components/RecipeCard";
import HeroCarousel, { type HeroSlide } from "@/components/HeroCarousel";
import EditorsPicks from "@/components/EditorsPicks";
import NewsletterBlock from "@/components/NewsletterBlock";
import Reveal from "@/components/Reveal";
import { createServerClient } from "@/lib/supabase/server";
import { fetchFooterSettings, fetchHomeSlots, fetchPageSeo } from "@/lib/content";
import { resolveRecipeImage } from "@/lib/images";
import styles from "./page.module.css";

export const revalidate = 60;

export async function generateMetadata(): Promise<Metadata> {
  const seo = await fetchPageSeo(createServerClient(), "home");
  if (!seo) return {};
  return {
    title: seo.seo_title ?? undefined,
    description: seo.seo_description ?? undefined,
  };
}

export default async function HomePage() {
  const client = createServerClient();
  const [slots, footer] = await Promise.all([
    fetchHomeSlots(client),
    fetchFooterSettings(client),
  ]);

  const heroSlides: HeroSlide[] = slots.hero.map((s) => ({
    id: s.id,
    title: s.recipe.title,
    href: `/recipes/${s.recipe.slug}`,
    eyebrow: s.eyebrow ?? "Recipe of the week",
    image: resolveRecipeImage(s.recipe.hero_image_path),
  }));

  return (
    <main>
      <HeroCarousel slides={heroSlides} />

      <section className={`shell ${styles.recipes}`}>
        <Reveal>
          <header className={styles.sectionHeader}>
            <span className="eyebrow eyebrowAccent">Section</span>
            <h2 className={styles.sectionTitle}>Recipes</h2>
          </header>
        </Reveal>
        <div className={styles.grid}>
          <div className={styles.gridLarge}>
            {slots.grid_large[0] && (
              <RecipeCard
                recipe={slots.grid_large[0].recipe}
                variant="large"
                eyebrow={slots.grid_large[0].eyebrow}
                description={slots.grid_large[0].description}
              />
            )}
          </div>
          <div className={styles.gridMedium}>
            {slots.grid_medium.map((s) => (
              <RecipeCard
                key={s.id}
                recipe={s.recipe}
                variant="medium"
                eyebrow={s.eyebrow}
              />
            ))}
          </div>
          <ul className={styles.gridList}>
            {slots.grid_list.map((s) => (
              <RecipeCard
                key={s.id}
                recipe={s.recipe}
                variant="list"
                eyebrow={s.eyebrow}
              />
            ))}
          </ul>
        </div>
        <div className={styles.wideRow}>
          {slots.wide.map((s, i) => (
            <Reveal key={s.id} delay={i * 0.1}>
              <RecipeCard recipe={s.recipe} variant="wide" eyebrow={s.eyebrow} />
            </Reveal>
          ))}
        </div>
      </section>

      <EditorsPicks picks={slots.pick} feature={slots.pick_feature[0] ?? null} />
      <NewsletterBlock settings={footer} />
    </main>
  );
}
```

- [ ] **Step 6: app/page.module.css**

```css
.recipes {
  padding-block: 4rem;
}

.sectionHeader {
  margin-bottom: 2.5rem;
  text-align: center;
}

.sectionTitle {
  margin-top: 1rem;
  font-size: var(--text-4xl);
  font-weight: 400;
  letter-spacing: var(--tracking-tight);
}

.grid {
  display: grid;
  gap: 2.5rem;
}

.gridMedium {
  display: flex;
  flex-direction: column;
  gap: 2.5rem;
}

.wideRow {
  margin-top: 4rem;
  display: grid;
  gap: 2.5rem;
}

@media (min-width: 768px) {
  .recipes {
    padding-block: 6rem;
  }
  .sectionHeader {
    margin-bottom: 4rem;
  }
  .sectionTitle {
    font-size: var(--text-5xl);
  }
  .grid,
  .gridMedium {
    gap: 3rem;
  }
  .wideRow {
    margin-top: 6rem;
    gap: 3rem;
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .recipes {
    padding-block: 8rem;
  }
  .sectionTitle {
    font-size: var(--text-6xl);
  }
  .grid {
    grid-template-columns: repeat(12, 1fr);
    gap: 2.5rem;
  }
  .gridLarge {
    grid-column: span 6;
  }
  .gridMedium {
    grid-column: span 3;
  }
  .gridList {
    grid-column: span 3;
    border-left: 1px solid var(--rule);
    padding-left: 2.5rem;
  }
}
```

- [ ] **Step 7: визуальная проверка** — `npm run dev`, сравнить http://localhost:3000 с `mockups/avocado-kiss-home.html` (открыть файл в браузере рядом): hero, сетка, picks, newsletter, футер; hover-зум и кроссфейд карусели работают.

- [ ] **Step 8: build, lint, commit**

```bash
npm run build && npm run lint
git add -A && git commit -m "feat: home page (hero carousel, curated grid, editor's picks)"
```

---

### Task 10: Страница рецепта /recipes/[slug]

**Files:**
- Create: `app/recipes/[slug]/page.tsx` + `page.module.css`

- [ ] **Step 1: app/recipes/[slug]/page.tsx**

```tsx
import type { Metadata } from "next";
import Image from "next/image";
import Link from "next/link";
import { notFound } from "next/navigation";
import ChevronLeftIcon from "@/components/icons/ChevronLeftIcon";
import UsersIcon from "@/components/icons/UsersIcon";
import ClockIcon from "@/components/icons/ClockIcon";
import { createServerClient } from "@/lib/supabase/server";
import { fetchRecipeBySlug, fetchRecipeSlugs } from "@/lib/content";
import { resolveRecipeImage } from "@/lib/images";
import styles from "./page.module.css";

export const revalidate = 60;

export async function generateStaticParams() {
  const slugs = await fetchRecipeSlugs(createServerClient());
  return slugs.map(({ slug }) => ({ slug }));
}

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const recipe = await fetchRecipeBySlug(createServerClient(), slug);
  if (!recipe) return {};
  return {
    title: recipe.seo_title ?? recipe.title,
    description: recipe.seo_description ?? recipe.excerpt ?? undefined,
  };
}

export default async function RecipePage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const recipe = await fetchRecipeBySlug(createServerClient(), slug);
  if (!recipe) notFound();

  const image = resolveRecipeImage(recipe.hero_image_path);

  return (
    <main>
      <article className={styles.article}>
        <Link href="/" className={styles.back}>
          <ChevronLeftIcon className={styles.backIcon} />
          Back
        </Link>

        <header className={styles.header}>
          <span className="eyebrow eyebrowAccent">
            {recipe.category ?? "Recipe"}
          </span>
          <h1 className={styles.title}>{recipe.title}</h1>
          {recipe.excerpt && <p className={styles.excerpt}>{recipe.excerpt}</p>}
          {(recipe.servings_label || recipe.time_label) && (
            <div className={styles.meta}>
              {recipe.servings_label && (
                <div className={styles.metaItem}>
                  <UsersIcon className={styles.metaIcon} />
                  <span className={`eyebrow ${styles.metaLabel}`}>Serves</span>
                  <span className={styles.metaValue}>{recipe.servings_label}</span>
                </div>
              )}
              {recipe.time_label && (
                <div className={styles.metaItem}>
                  <ClockIcon className={styles.metaIcon} />
                  <span className={`eyebrow ${styles.metaLabel}`}>Time</span>
                  <span className={styles.metaValue}>{recipe.time_label}</span>
                </div>
              )}
            </div>
          )}
        </header>

        {image && (
          <div className={styles.imageWrap}>
            <Image
              src={image.url}
              alt={recipe.title}
              width={1920}
              height={1080}
              priority
              unoptimized={image.unoptimized}
              className={styles.image}
            />
          </div>
        )}

        <div className={styles.content}>
          {recipe.ingredients.length > 0 && (
            <section>
              <h2 className={styles.subheading}>Ingredients</h2>
              <ul className={styles.ingredients}>
                {recipe.ingredients.map((item, i) => (
                  <li key={i} className={styles.ingredient}>
                    <span className={styles.bullet} />
                    <span>{item}</span>
                  </li>
                ))}
              </ul>
            </section>
          )}
          {recipe.steps.length > 0 && (
            <section>
              <h2 className={styles.subheading}>Method</h2>
              <ol className={styles.steps}>
                {recipe.steps.map((step, i) => (
                  <li key={i} className={styles.step}>
                    <span className={styles.stepNumber}>
                      {String(i + 1).padStart(2, "0")}
                    </span>
                    <p className={styles.stepText}>{step}</p>
                  </li>
                ))}
              </ol>
            </section>
          )}
        </div>
      </article>
    </main>
  );
}
```

- [ ] **Step 2: app/recipes/[slug]/page.module.css**

```css
.article {
  max-width: 1100px;
  margin: 0 auto;
  padding: 2rem 1rem 5rem;
}

.back {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  font-size: var(--text-xs);
  text-transform: uppercase;
  letter-spacing: 0.2em;
  color: color-mix(in srgb, var(--foreground) 60%, transparent);
  transition: color 0.15s var(--ease-default);
}

.back:hover {
  color: var(--accent);
}

.backIcon {
  height: 0.875rem;
  width: 0.875rem;
}

.header {
  margin-top: 2rem;
  max-width: 48rem;
}

.title {
  margin-top: 1rem;
  font-size: var(--text-3xl);
  font-weight: 400;
  line-height: 1.05;
  letter-spacing: var(--tracking-tight);
  text-wrap: balance;
}

.excerpt {
  margin-top: 1.25rem;
  max-width: 42rem;
  font-size: var(--text-base);
  line-height: var(--leading-relaxed);
  color: var(--muted-foreground);
}

.meta {
  margin-top: 1.75rem;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  column-gap: 2rem;
  row-gap: 0.75rem;
  border-block: 1px solid var(--rule);
  padding-block: 1rem;
  font-size: var(--text-sm);
}

.metaItem {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: color-mix(in srgb, var(--foreground) 80%, transparent);
}

.metaIcon {
  height: 1rem;
  width: 1rem;
  color: var(--accent);
}

.metaLabel {
  color: color-mix(in srgb, var(--foreground) 60%, transparent);
}

.metaValue {
  font-weight: 500;
}

.imageWrap {
  margin-top: 2.5rem;
  overflow: hidden;
  border-radius: var(--radius);
}

.image {
  width: 100%;
  aspect-ratio: 16 / 10;
  object-fit: cover;
}

.content {
  margin-top: 3rem;
  display: grid;
  gap: 3rem;
}

.subheading {
  font-size: var(--text-2xl);
  font-weight: 400;
  letter-spacing: var(--tracking-tight);
}

.ingredients {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.ingredient {
  display: flex;
  gap: 0.75rem;
  border-bottom: 1px solid var(--rule);
  padding-bottom: 0.75rem;
  font-size: var(--text-sm);
  line-height: var(--leading-relaxed);
  color: color-mix(in srgb, var(--foreground) 85%, transparent);
}

.ingredient:last-child {
  border-bottom: none;
}

.bullet {
  margin-top: 0.5rem;
  height: 0.375rem;
  width: 0.375rem;
  flex-shrink: 0;
  border-radius: 9999px;
  background: var(--accent);
}

.steps {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.75rem;
}

.step {
  display: flex;
  align-items: flex-start;
  gap: 1.25rem;
}

.stepNumber {
  font-size: var(--text-2xl);
  font-weight: 500;
  line-height: 1;
  color: var(--accent);
}

.stepText {
  margin-top: -0.25rem;
  font-size: var(--text-base);
  line-height: var(--leading-relaxed);
  color: color-mix(in srgb, var(--foreground) 90%, transparent);
}

@media (min-width: 768px) {
  .article {
    padding: 3rem 1.5rem 7rem;
  }
  .header {
    margin-top: 3rem;
  }
  .title {
    margin-top: 1.25rem;
    font-size: var(--text-5xl);
  }
  .excerpt {
    margin-top: 1.5rem;
    font-size: var(--text-lg);
  }
  .meta {
    margin-top: 2.5rem;
    padding-block: 1.25rem;
  }
  .imageWrap {
    margin-top: 3.5rem;
  }
  .image {
    aspect-ratio: 16 / 9;
  }
  .content {
    margin-top: 5rem;
    gap: 4rem;
  }
  .subheading {
    font-size: var(--text-3xl);
  }
  .ingredients,
  .steps {
    margin-top: 2rem;
  }
  .steps {
    gap: 2.25rem;
  }
  .step {
    gap: 1.75rem;
  }
  .stepNumber {
    font-size: var(--text-3xl);
  }
  .stepText {
    font-size: var(--text-lg);
  }
  .ingredient {
    font-size: var(--text-base);
  }
}

@media (min-width: 1024px) {
  .article {
    padding-inline: 3rem;
  }
  .title {
    font-size: var(--text-6xl);
  }
  .content {
    grid-template-columns: 1fr 1.6fr;
    gap: 5rem;
  }
}
```

- [ ] **Step 3: проверка** — `npm run dev`, открыть `/recipes/roasted-zucchini-with-white-beans-and-kale`, сравнить с `mockups/avocado-kiss-recipe.html`; несуществующий slug → 404.

- [ ] **Step 4: build, lint, commit**

```bash
npm run build && npm run lint
git add -A && git commit -m "feat: recipe page (/recipes/[slug])"
```

---

### Task 11: Страница категории, 404, sitemap, robots

**Files:**
- Create: `app/category/[slug]/page.tsx` + `page.module.css`
- Create: `app/not-found.tsx` + `not-found.module.css`
- Create: `app/sitemap.ts`, `app/robots.ts`

- [ ] **Step 1: app/category/[slug]/page.tsx** (макета нет — хедер как у секции Recipes + сетка medium-карточек с временем)

```tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import RecipeCard from "@/components/RecipeCard";
import Reveal from "@/components/Reveal";
import { createServerClient } from "@/lib/supabase/server";
import {
  fetchCategories,
  fetchCategoryBySlug,
  fetchRecipesByCategory,
} from "@/lib/content";
import styles from "./page.module.css";

export const revalidate = 60;

export async function generateStaticParams() {
  const categories = await fetchCategories(createServerClient());
  return categories.map(({ slug }) => ({ slug }));
}

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const category = await fetchCategoryBySlug(createServerClient(), slug);
  if (!category) return {};
  return {
    title: category.seo_title ?? `${category.name} recipes`,
    description:
      category.seo_description ??
      `${category.name} recipes from Avocado Kiss — a culinary magazine about slow food and seasonal produce.`,
  };
}

export default async function CategoryPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const client = createServerClient();
  const category = await fetchCategoryBySlug(client, slug);
  if (!category) notFound();

  const recipes = await fetchRecipesByCategory(client, category.name);

  return (
    <main className={`shell ${styles.page}`}>
      <Reveal>
        <header className={styles.header}>
          <span className="eyebrow eyebrowAccent">Category</span>
          <h1 className={styles.title}>{category.name}</h1>
        </header>
      </Reveal>
      {recipes.length === 0 ? (
        <p className={styles.empty}>No recipes here yet — check back soon.</p>
      ) : (
        <div className={styles.grid}>
          {recipes.map((r) => (
            <RecipeCard key={r.id} recipe={r} variant="medium" />
          ))}
        </div>
      )}
    </main>
  );
}
```

- [ ] **Step 2: app/category/[slug]/page.module.css**

```css
.page {
  padding-block: 4rem;
}

.header {
  margin-bottom: 2.5rem;
  text-align: center;
}

.title {
  margin-top: 1rem;
  font-size: var(--text-4xl);
  font-weight: 400;
  letter-spacing: var(--tracking-tight);
}

.empty {
  text-align: center;
  color: var(--muted-foreground);
}

.grid {
  display: grid;
  gap: 2.5rem;
}

@media (min-width: 640px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 768px) {
  .page {
    padding-block: 6rem;
  }
  .header {
    margin-bottom: 4rem;
  }
  .title {
    font-size: var(--text-5xl);
  }
  .grid {
    gap: 3rem;
  }
}

@media (min-width: 1024px) {
  .title {
    font-size: var(--text-6xl);
  }
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

- [ ] **Step 3: app/not-found.tsx + not-found.module.css**

```tsx
import Link from "next/link";
import styles from "./not-found.module.css";

export default function NotFound() {
  return (
    <main className={styles.main}>
      <span className="eyebrow eyebrowAccent">404</span>
      <h1 className={styles.title}>This page has left the kitchen</h1>
      <Link href="/" className={styles.link}>
        ← Back to the magazine
      </Link>
    </main>
  );
}
```

```css
/* app/not-found.module.css */
.main {
  min-height: 50vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  padding: 4rem 1rem;
  text-align: center;
}

.title {
  font-size: var(--text-3xl);
  font-weight: 400;
  letter-spacing: var(--tracking-tight);
}

.link {
  font-size: var(--text-sm);
  text-transform: uppercase;
  letter-spacing: var(--tracking-eyebrow);
  color: var(--muted-foreground);
  transition: color 0.15s var(--ease-default);
}

.link:hover {
  color: var(--accent);
}
```

- [ ] **Step 4: app/sitemap.ts и app/robots.ts** (`NEXT_PUBLIC_SITE_URL` добавить в `.env.local` после появления прод-URL; фолбэк — localhost)

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { createServerClient } from "@/lib/supabase/server";
import { fetchCategories, fetchRecipeSlugs } from "@/lib/content";

const BASE = process.env.NEXT_PUBLIC_SITE_URL ?? "http://localhost:3000";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const client = createServerClient();
  const [recipes, categories] = await Promise.all([
    fetchRecipeSlugs(client),
    fetchCategories(client),
  ]);
  return [
    { url: BASE, changeFrequency: "daily", priority: 1 },
    ...categories.map((c) => ({
      url: `${BASE}/category/${c.slug}`,
      changeFrequency: "weekly" as const,
      priority: 0.7,
    })),
    ...recipes.map((r) => ({
      url: `${BASE}/recipes/${r.slug}`,
      lastModified: r.updated_at,
      changeFrequency: "monthly" as const,
      priority: 0.8,
    })),
  ];
}
```

```ts
// app/robots.ts
import type { MetadataRoute } from "next";

const BASE = process.env.NEXT_PUBLIC_SITE_URL ?? "http://localhost:3000";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: "*", allow: "/" },
    sitemap: `${BASE}/sitemap.xml`,
  };
}
```

- [ ] **Step 5: build, lint, commit**

```bash
npm run build && npm run lint
git add -A && git commit -m "feat: category pages, 404, sitemap, robots"
```

---

### Task 12: Финальная верификация

- [ ] **Step 1:** `npm run build` — prerender всех маршрутов зелёный (`/`, `/recipes/[slug]` ×15, `/category/[slug]` ×8, sitemap, robots, 404).
- [ ] **Step 2:** `npm run lint` — чисто.
- [ ] **Step 3:** `npm run dev` + браузер: сверка главной и рецепта с макетами на ширинах ~375 / 768 / 1440: sticky-шапка с blur, мобильное меню, карусель (кроссфейд, точки, стрелки, автоплей, пауза при hover), hover-зумы, sticky pick_feature, инверсный newsletter, футер; категории и 404.
- [ ] **Step 4:** проверить, что при отсутствии загруженных картинок (Step 5 Task 4 не выполнен) страницы рендерятся без ошибок (пустые imageWrap).
- [ ] **Step 5:** зафиксировать найденные визуальные расхождения и поправить CSS точечно; каждый фикс — отдельный маленький коммит `fix: …`.

---

### Task 13: Документация workspace

**Files:**
- Create: `platform-docs/sites/avocado-kiss.md`
- Create: `avocado.kiss/AGENTS.md`, `avocado.kiss/CLAUDE.md`, `avocado.kiss/README.md`
- Modify: `platform-docs/sites/overview.md` (таблица сайтов), `platform-docs/AGENTS.md` (Projects), `platform-docs/database/schema.md` (схема avocado_kiss)

- [ ] **Step 1: platform-docs/sites/avocado-kiss.md** — архитектура сайта по образцу cozycorner.md: маршруты, ISR, схема/бакет, слоты главной (таблица ёмкостей), контракт картинок, деплой (Vercel — pending до первого деплоя).

- [ ] **Step 2: platform-docs/database/schema.md** — добавить раздел «Таблицы схемы avocado_kiss» (7 таблиц с полями — перенести из спеки), строку в §1 (схема/бакет), пункт в §6-аналог об истории миграций avocado.kiss (0001–0003), упомянуть зависимость от public.slugify/set_updated_at/is_admin.

- [ ] **Step 3: platform-docs/sites/overview.md** — строка в таблицу «Действующие сайты»: Avocado Kiss | `avocado.kiss/` | `avocado_kiss` / `avocado-kiss-photos` | прод pending | [avocado-kiss.md](avocado-kiss.md).

- [ ] **Step 4: platform-docs/AGENTS.md** — `avocado.kiss/` в «Projects in this workspace» + строка в Documentation index (avocado-kiss.md).

- [ ] **Step 5: avocado.kiss/AGENTS.md** — по образцу cozycorner/AGENTS.md (блок «NOT the Next.js you know», стек, структура, конвенции, схема `avocado_kiss`, бакет, таблица shared docs); `CLAUDE.md` = `# Claude Code — avocado-kiss\n\n@AGENTS.md`; короткий README.md (что за проект, dev-команды, env).

- [ ] **Step 6: Commits**

```bash
cd /Users/mdr/Documents/Projects/workspace-one/avocado.kiss
git add -A && git commit -m "docs: AGENTS/CLAUDE/README"
cd ../platform-docs
git add -A && git commit -m "docs: register avocado.kiss (site doc, schema, overview, index)"
```

- [ ] **Step 7: напомнить пользователю про отложенные ручные шаги**, если не сделаны: Exposed schemas (Task 3 Step 4), загрузка seed-картинок в бакет (Task 4 Step 5). Vercel-деплой и запись в `SITES` админки — фаза B.
