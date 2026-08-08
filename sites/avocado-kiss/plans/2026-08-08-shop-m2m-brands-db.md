# Avocado Shop — Фундамент БД (M2M-категории, бренды, папки, триггеры) — план

> **For agentic workers:** REQUIRED SUB-SKILL: используй superpowers:subagent-driven-development (рекомендуется) или superpowers:executing-plans, чтобы выполнять план по задачам. Шаги помечены чекбоксами (`- [ ]`).
>
> ⚠️ **Правило workspace:** не коммитить без явного разрешения пользователя. Шаги «Commit» ниже выполнять только после «ок» от пользователя.
>
> ⚠️ Миграции применяются к **живому** проекту base-one (`zwrkphynupdubevzwdzy`) через Supabase MCP `apply_migration`, а файл миграции в `avocado.kiss/supabase/migrations/` — source of truth (правило `avocado.kiss/AGENTS.md`). Миграция **аддитивная и идемпотентная** (`if not exists`) — прод-сайт продолжает работать на `products.category`.

**Goal:** Довести схему `avocado_kiss` до модели cozy для Curated Shop: M2M `product_categories`, справочник `brands`, `products.folder_id`, продуктовый SEO, slug-триггеры и авто-`item_count` — без поломки живого сайта.

**Architecture:** Одна аддитивная миграция `0013_shop_m2m_brands.sql`. Собирается по секциям (Задачи 1–6), применяется атомарно один раз (Задача 7), затем сверяется SQL-проверками. `products.category` НЕ трогаем (удалим в Плане 2 после переключения сайта). Референс-DDL: cozy `0023_brands.sql`, `0027_product_categories.sql`, `0022_content_folders.sql`.

**Tech Stack:** Postgres (Supabase), схема `avocado_kiss`, общая `public.is_admin()` / `public.slugify()`; применение через MCP `apply_migration` + execute_sql для проверок.

**Базовые данные (сверено 2026-08-08):** products = 48 (все с category и brand), distinct brand = 20, shop_categories = 6, orphan-по-категории = 0. Эти числа — ожидаемые результаты проверок ниже.

---

## Структура файлов

- Create: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` — вся миграция фазы 1.
- Modify: `platform-docs/database/schema.md` §9 (таблицы avocado_kiss) + §10 (история миграций) — после применения.
- Проверки: `mcp__supabase__execute_sql` (project_id `zwrkphynupdubevzwdzy`).

Порядок секций внутри файла (важно для FK и посева):
1. `brands` (+ trigger + grants/RLS + seed)
2. slug-триггеры `products` и `shop_categories`
3. `product_categories` (+ grants/RLS + backfill)
4. функция+триггер `item_count` + первичный пересчёт
5. `products.folder_id` + расширение CHECK `admin_folders`
6. `products.seo_title` / `seo_description`

---

## Task 1: Секция BRANDS в файле миграции

**Files:**
- Create/append: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`

- [ ] **Step 1: Создать файл миграции с шапкой и секцией brands**

Записать в `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`:

```sql
-- Avocado Kiss Curated Shop — фаза 1 (паритет с cozy): M2M-категории, справочник
-- брендов, папки товаров, продуктовый SEO, slug-триггеры, авто-item_count.
-- Аддитивно/идемпотентно. products.category СОХРАНЯЕТСЯ (drop — в отдельной
-- миграции после переключения сайта). Референс: cozy 0023/0027/0022.
-- Спека: platform-docs/sites/avocado-kiss/specs/2026-08-08-shop-m2m-brands-admin-design.md

-- ============ 1. BRANDS (справочник; products.brand остаётся text) ============
create table if not exists avocado_kiss.brands (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  name text not null unique,
  slug text not null,
  position integer not null default 0
);
create unique index if not exists brands_slug_key on avocado_kiss.brands (slug);

create or replace function public.avocado_brands_set_slug() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.name), ''), 'brand');
  end if;
  return new;
end; $$;
drop trigger if exists brands_set_slug on avocado_kiss.brands;
create trigger brands_set_slug before insert or update on avocado_kiss.brands
  for each row execute function public.avocado_brands_set_slug();

grant select on avocado_kiss.brands to anon;
grant all on avocado_kiss.brands to authenticated, service_role;

alter table avocado_kiss.brands enable row level security;
drop policy if exists "Public read brands" on avocado_kiss.brands;
create policy "Public read brands" on avocado_kiss.brands for select
  to anon, authenticated using (true);
drop policy if exists "Admin write brands" on avocado_kiss.brands;
create policy "Admin write brands" on avocado_kiss.brands for all
  to authenticated using (public.is_admin()) with check (public.is_admin());

insert into avocado_kiss.brands (name)
  select distinct brand from avocado_kiss.products
  where brand is not null and brand <> ''
on conflict (name) do nothing;
```

Примечание: имя функции `public.avocado_brands_set_slug` (с префиксом avocado), чтобы не
конфликтовать с `public.brands_set_slug` cozy — обе схемы делят `public`.

- [ ] **Step 2: Проверить, что файл содержит секцию brands**

Run: `grep -c "avocado_kiss.brands" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: число ≥ 6 (таблица, индекс, grants, RLS, seed упоминают таблицу).

(Применение и проверка данных — в Task 7; здесь только сборка файла.)

---

## Task 2: Секция SLUG-ТРИГГЕРОВ (products, shop_categories)

**Files:**
- Modify: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` (append)

- [ ] **Step 1: Дописать slug-триггеры**

Добавить в конец файла:

```sql
-- ============ 2. SLUG-ТРИГГЕРЫ (как cozy products_set_slug) ============
-- Заполняют slug из title/name, если он пуст. Существующие строки уже со slug —
-- бэкофилл не нужен. Нужны, чтобы админ мог создавать товар/категорию без slug.
create or replace function public.avocado_products_set_slug() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.title), ''), 'product');
  end if;
  return new;
end; $$;
drop trigger if exists products_set_slug on avocado_kiss.products;
create trigger products_set_slug before insert or update on avocado_kiss.products
  for each row execute function public.avocado_products_set_slug();

create or replace function public.avocado_shop_categories_set_slug() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.name), ''), 'category');
  end if;
  return new;
end; $$;
drop trigger if exists shop_categories_set_slug on avocado_kiss.shop_categories;
create trigger shop_categories_set_slug before insert or update on avocado_kiss.shop_categories
  for each row execute function public.avocado_shop_categories_set_slug();
```

- [ ] **Step 2: Проверить наличие обоих триггеров в файле**

Run: `grep -c "set_slug" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: ≥ 6 (2 функции + 2 drop + 2 create trigger; brands-триггер из Task 1 тоже матчится — число будет больше, это ок).

---

## Task 3: Секция PRODUCT_CATEGORIES (M2M) + backfill

**Files:**
- Modify: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` (append)

- [ ] **Step 1: Дописать M2M-таблицу, grants/RLS и backfill (по SLUG!)**

```sql
-- ============ 3. PRODUCT_CATEGORIES (M2M товар↔shop_category) ============
-- ВАЖНО: у avocado products.category = shop_categories.SLUG (не name, как у cozy).
create table if not exists avocado_kiss.product_categories (
  product_id  uuid not null references avocado_kiss.products(id)        on delete cascade,
  category_id uuid not null references avocado_kiss.shop_categories(id) on delete cascade,
  primary key (product_id, category_id)
);
create index if not exists product_categories_category_id_idx
  on avocado_kiss.product_categories (category_id);

grant select on avocado_kiss.product_categories to anon;
grant all on avocado_kiss.product_categories to authenticated, service_role;

alter table avocado_kiss.product_categories enable row level security;
drop policy if exists "Public read product_categories" on avocado_kiss.product_categories;
create policy "Public read product_categories" on avocado_kiss.product_categories for select
  to anon, authenticated using (true);
drop policy if exists "Admin write product_categories" on avocado_kiss.product_categories;
create policy "Admin write product_categories" on avocado_kiss.product_categories for all
  to authenticated using (public.is_admin()) with check (public.is_admin());

-- Backfill из текстового products.category (matched by SLUG, не name).
insert into avocado_kiss.product_categories (product_id, category_id)
select p.id, c.id
from avocado_kiss.products p
join avocado_kiss.shop_categories c on c.slug = p.category
where p.category is not null and p.category <> ''
on conflict do nothing;
```

- [ ] **Step 2: Проверить секцию в файле**

Run: `grep -c "product_categories" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: ≥ 7.

---

## Task 4: Секция ITEM_COUNT (функция + триггер + первичный пересчёт)

**Files:**
- Modify: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` (append)

- [ ] **Step 1: Дописать авто-item_count**

Идёт ПОСЛЕ backfill product_categories, чтобы пересчёт увидел засеянные строки:

```sql
-- ============ 4. ITEM_COUNT (авто-счётчик shop_categories.item_count) ============
-- Заменяет ручной счётчик (skill avocado-content-ops). Держит item_count = числу
-- membership в product_categories. Срабатывает на любой insert/update/delete membership.
create or replace function avocado_kiss.sync_shop_category_item_count() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if tg_op = 'INSERT' then
    update avocado_kiss.shop_categories set item_count =
      (select count(*) from avocado_kiss.product_categories where category_id = new.category_id)
      where id = new.category_id;
  elsif tg_op = 'DELETE' then
    update avocado_kiss.shop_categories set item_count =
      (select count(*) from avocado_kiss.product_categories where category_id = old.category_id)
      where id = old.category_id;
  elsif tg_op = 'UPDATE' and new.category_id is distinct from old.category_id then
    update avocado_kiss.shop_categories set item_count =
      (select count(*) from avocado_kiss.product_categories where category_id = old.category_id)
      where id = old.category_id;
    update avocado_kiss.shop_categories set item_count =
      (select count(*) from avocado_kiss.product_categories where category_id = new.category_id)
      where id = new.category_id;
  end if;
  return null;
end; $$;
drop trigger if exists sync_item_count on avocado_kiss.product_categories;
create trigger sync_item_count after insert or update or delete on avocado_kiss.product_categories
  for each row execute function avocado_kiss.sync_shop_category_item_count();

-- Первичный пересчёт для всех категорий (backfill уже вставил membership выше).
update avocado_kiss.shop_categories c set item_count =
  (select count(*) from avocado_kiss.product_categories pc where pc.category_id = c.id);
```

- [ ] **Step 2: Проверить секцию**

Run: `grep -c "sync_shop_category_item_count\|sync_item_count" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: ≥ 3.

---

## Task 5: Секция FOLDER_ID + расширение CHECK admin_folders

**Files:**
- Modify: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` (append)

- [ ] **Step 1: Дописать folder_id и правку CHECK**

Имя constraint сверено: `admin_folders_section_check`, текущий def
`CHECK (section = ANY (ARRAY['media','recipes']))`.

```sql
-- ============ 5. FOLDER_ID у товаров + секция 'products' в admin_folders ============
alter table avocado_kiss.products
  add column if not exists folder_id uuid references avocado_kiss.admin_folders(id) on delete set null;
create index if not exists products_folder_id_idx on avocado_kiss.products (folder_id);

alter table avocado_kiss.admin_folders drop constraint if exists admin_folders_section_check;
alter table avocado_kiss.admin_folders add constraint admin_folders_section_check
  check (section in ('media','recipes','products'));
```

- [ ] **Step 2: Проверить секцию**

Run: `grep -c "folder_id\|admin_folders_section_check" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: ≥ 3.

---

## Task 6: Секция ПРОДУКТОВЫЙ SEO

**Files:**
- Modify: `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` (append)

- [ ] **Step 1: Дописать SEO-колонки товара**

```sql
-- ============ 6. Продуктовый SEO (живой фолбэк: пусто → title/description) ============
alter table avocado_kiss.products add column if not exists seo_title text;
alter table avocado_kiss.products add column if not exists seo_description text;
```

- [ ] **Step 2: Проверить, что файл собран целиком**

Run: `grep -cE "^-- ====" avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql`
Expected: 6 (шесть секций).

---

## Task 7: Применить миграцию и сверить данные (это «тест»)

**Files:**
- Apply: содержимое `0013_shop_m2m_brands.sql` через MCP.

- [ ] **Step 1: Применить миграцию через MCP**

Прочитать весь файл `avocado.kiss/supabase/migrations/0013_shop_m2m_brands.sql` и вызвать:
`mcp__supabase__apply_migration` с `project_id = "zwrkphynupdubevzwdzy"`, `name = "shop_m2m_brands"`, `query = <всё содержимое файла>`.
Expected: успех без ошибок.

- [ ] **Step 2: Сверить структуры и посев (ожидаемые числа)**

Run `mcp__supabase__execute_sql` (project_id `zwrkphynupdubevzwdzy`):

```sql
select
  (select count(*) from avocado_kiss.brands)                                as brands,        -- ожид. 20
  (select count(*) from avocado_kiss.brands where slug is null or slug='')  as brands_no_slug,-- ожид. 0
  (select count(*) from avocado_kiss.product_categories)                    as memberships,   -- ожид. 48
  (select count(*) from avocado_kiss.products p
     where not exists (select 1 from avocado_kiss.product_categories pc where pc.product_id=p.id)
       and p.category is not null and p.category<>'')                       as products_missing_membership, -- ожид. 0
  (select sum(item_count) from avocado_kiss.shop_categories)                as item_count_sum,-- ожид. 48
  (select count(*) from avocado_kiss.shop_categories c
     where c.item_count <> (select count(*) from avocado_kiss.product_categories pc where pc.category_id=c.id)) as item_count_drift, -- ожид. 0
  (select count(*) from information_schema.columns
     where table_schema='avocado_kiss' and table_name='products'
       and column_name in ('folder_id','seo_title','seo_description'))      as new_product_cols; -- ожид. 3
```
Expected: `brands=20, brands_no_slug=0, memberships=48, products_missing_membership=0, item_count_sum=48, item_count_drift=0, new_product_cols=3`.

- [ ] **Step 3: Проверить, что CHECK допускает 'products' и триггеры на месте**

```sql
select
  pg_get_constraintdef(con.oid) as check_def,
  (select count(*) from pg_trigger t join pg_class r on r.oid=t.tgrelid join pg_namespace n on n.oid=r.relnamespace
     where n.nspname='avocado_kiss' and r.relname='products' and t.tgname='products_set_slug') as products_slug_trg,
  (select count(*) from pg_trigger t join pg_class r on r.oid=t.tgrelid join pg_namespace n on n.oid=r.relnamespace
     where n.nspname='avocado_kiss' and r.relname='shop_categories' and t.tgname='shop_categories_set_slug') as shopcat_slug_trg,
  (select count(*) from pg_trigger t join pg_class r on r.oid=t.tgrelid join pg_namespace n on n.oid=r.relnamespace
     where n.nspname='avocado_kiss' and r.relname='product_categories' and t.tgname='sync_item_count') as item_count_trg
from pg_constraint con join pg_class rel on rel.oid=con.conrelid join pg_namespace n on n.oid=rel.relnamespace
where n.nspname='avocado_kiss' and rel.relname='admin_folders' and con.contype='c';
```
Expected: `check_def` содержит `'products'`; три счётчика триггеров = 1.

- [ ] **Step 4: Smoke-тест авто-item_count (транзакция с rollback — следов не остаётся)**

Проверяет, что триггер реально пересчитывает `item_count` при вставке membership.
Всё в одной транзакции с `rollback` — на проде НИЧЕГО не меняется:

```sql
begin;
-- добавить одну ещё не существующую связь товар↔категория (триггер должен сработать)
insert into avocado_kiss.product_categories (product_id, category_id)
select p.id, c.id
from avocado_kiss.products p cross join avocado_kiss.shop_categories c
where not exists (
  select 1 from avocado_kiss.product_categories pc
  where pc.product_id = p.id and pc.category_id = c.id
)
limit 1;
-- после вставки item_count всех категорий должен по-прежнему совпадать с реальным членством
select count(*) as drift_after_insert
from avocado_kiss.shop_categories c
where c.item_count <> (
  select count(*) from avocado_kiss.product_categories pc where pc.category_id = c.id
);
rollback;
```
Expected: `drift_after_insert = 0` (триггер держит счётчик синхронным); `rollback` откатывает тестовую связь — прод-данные не изменены. Сверить после этого, что `memberships` снова = 48 (Step 2), чтобы убедиться, что откат прошёл.

- [ ] **Step 5: Commit (только с разрешения пользователя)**

```bash
cd avocado.kiss && git add supabase/migrations/0013_shop_m2m_brands.sql \
  && git commit -m "feat(shop): additive DB migration — brands, product_categories M2M, folders, product SEO, slug + item_count triggers"
```

---

## Task 8: Обновить документацию схемы

**Files:**
- Modify: `platform-docs/database/schema.md` §9 (таблицы avocado_kiss) + §10 (история миграций)

- [ ] **Step 1: Дополнить §9**

Добавить описания новых таблиц/колонок в раздел avocado_kiss:
- `brands` — справочник брендов (id, created_at, name unique, slug unique, position); RLS public read + admin write; `products.brand` остаётся text (источник истины), FK нет.
- `product_categories` — M2M `products`↔`shop_categories` (PK обе колонки, FK on delete cascade); заменяет чтение `products.category` (колонка ещё существует до Плана 2).
- `products.folder_id` — nullable FK → admin_folders (секция `products`); `seo_title`/`seo_description` добавлены.
- `admin_folders.section` — CHECK теперь `('media','recipes','products')`.
- `shop_categories.item_count` — теперь поддерживается триггером `sync_item_count` на `product_categories` (не вручную).
- slug-триггеры на `products`/`shop_categories`.

- [ ] **Step 2: Дополнить §10 (история миграций avocado.kiss)**

Добавить строку: `0013 shop m2m + brands + folders + product SEO + slug/item_count triggers`.

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
cd platform-docs && git add database/schema.md \
  && git commit -m "docs(schema): avocado_kiss shop m2m/brands/folders/triggers (migration 0013)"
```

---

## Готово, когда

- Миграция `0013` применена; проверки Task 7 Step 2–4 дали ожидаемые числа.
- Файл миграции лежит в репозитории avocado.kiss (source of truth).
- schema.md §9/§10 обновлены.
- Прод-сайт avocado.kiss продолжает работать (он всё ещё читает `products.category` — не тронут).

Следующий план (после выполнения этого): **План 2 — переключение сайта avocado.kiss на M2M** + drop `products.category`, написанный уже по фактически применённой схеме.
