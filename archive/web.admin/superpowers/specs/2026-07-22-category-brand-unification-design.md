# Спека: унификация Category & Brand (web.admin + миграция БД)

> Date: 2026-07-22 | Project: web.admin (+ миграция в репо cozycorner) | Scope: БД + фронт
> Related: [products.md](../../../../admin-panel/products.md), [components.md](../../../../admin-panel/components.md),
> [schema.md](../../../../database/schema.md), [media.md](../../../../admin-panel/media.md) §3 (паттерн папок)

Category и Brand у товара сейчас устроены по-разному и обе неудобны при
добавлении товара. Приводим их к единой модели «справочник-сущность»: обе можно
создать / переименовать / удалить (у категории ещё фото), выбор в форме товара —
единый combobox с созданием на месте, плюс полноценные разделы в навигации.

## Исходное состояние (проверено по миграциям, НЕ по доке)

- **brand** — просто `text`-колонка на `products` (миграция `0005_products_brand_category.sql`).
  **Таблицы `brands` НЕТ.** В форме товара — free-text `Input`. Управлять брендами
  (rename/delete) нельзя.
- **category** — полноценная таблица `categories` (миграция `0014_categories.sql`:
  `id, created_at, name unique, slug (авто-триггер `categories_set_slug`), image_path,
  position`; hero/seo-поля добавлены в `0015`). Но **раздела управления в админке нет** —
  категории только выбираются `Select`'ом из засиженных миграцией; создать/переименовать/
  удалить/сменить фото из админки нельзя.
- **Каскада нет ни у одной**: FK между `products.category`/`products.brand` и
  справочниками отсутствует сознательно (связь по имени-строке). Переименование имени
  в справочнике сейчас осиротит `products`.

Вывод: нужна новая миграция (таблица `brands`), общий слой данных с каскадом на
уровне приложения и общий UI.

## Цель и критерии успеха

- Для Category и Brand доступны: создать, переименовать, удалить; у Category — ещё
  сменить фото; обе — с изменением порядка (position).
- Rename справочника обновляет и связанные `products` (не осиротляет).
- В форме товара оба поля — единый combobox: выбрать существующий ИЛИ создать новый
  на месте, плюс вход в полное управление.
- В навигации есть разделы Categories и Brands (полный CRUD, паттерн Products).
- Category и Brand используют один generic-компонент/хук; различие — конфиг
  (`hasImage`).

Вне scope: hero/SEO-поля категорий (управляем позже), FK/атомарный каскад в БД
(осознанно app-level), бренд-страницы на сайте (сайт уже читает `products.brand`).

## Дефолты поведения (подтверждены пользователем)

- **Delete** справочника обнуляет ссылку у затронутых товаров (`products.category`/
  `brand := null`), а не оставляет висячую строку. Диалог показывает «N products use
  this — they will lose their <category|brand>».
- **Category-менеджер** в этот заход: `name` + `photo` + `position`. Hero/SEO — позже.

## 1. БД — миграция (проект `zwrkphynupdubevzwdzy`, схема `cozycorner`)

> Проверено через MCP: проект в `sites.ts` — `zwrkphynupdubevzwdzy` (имя «base-one»,
> НЕ отдельный проект «CozyCorner» `nkaobsivfzsjqypuaamw`). Контент-таблицы живут в схеме
> `cozycorner` (перенесены из `public` миграцией 0017); функции остаются в `public`.
> У `categories` фактически ДВЕ политики: `Public read categories` (SELECT anon+auth) и
> `Admin write categories` (ALL authenticated, `public.is_admin()`). `brands` зеркалит это.

Новый файл `cozycorner/supabase/migrations/0023_brands.sql` (следующий свободный номер;
через MCP `apply_migration` name=`0023_brands`), зеркально категориям, но **без `image_path`**.
Таблица создаётся **сразу в схеме `cozycorner`**, slug-функция — в `public`:

```sql
create table if not exists cozycorner.brands (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  name text not null unique,
  slug text not null,
  position integer not null default 0
);
create unique index if not exists brands_slug_key on cozycorner.brands (slug);

-- slug из name (slugify — в public, из 0007), паттерн public.categories_set_slug
create or replace function public.brands_set_slug() returns trigger
  language plpgsql set search_path = '' as $$
begin
  if new.slug is null or new.slug = '' then
    new.slug := coalesce(nullif(public.slugify(new.name), ''), 'brand');
  end if;
  return new;
end; $$;
drop trigger if exists brands_set_slug on cozycorner.brands;
create trigger brands_set_slug before insert or update on cozycorner.brands
  for each row execute function public.brands_set_slug();

-- Grants (как для схемы cozycorner в 0017: anon читает, authenticated/service_role пишут)
grant select on cozycorner.brands to anon;
grant all on cozycorner.brands to authenticated, service_role;

alter table cozycorner.brands enable row level security;
drop policy if exists "Public read brands" on cozycorner.brands;
create policy "Public read brands" on cozycorner.brands for select
  to anon, authenticated using (true);
drop policy if exists "Admin write brands" on cozycorner.brands;
create policy "Admin write brands" on cozycorner.brands for all
  to authenticated using (public.is_admin()) with check (public.is_admin());

-- сид: существующие бренды из products.brand
insert into cozycorner.brands (name)
  select distinct brand from cozycorner.products
  where brand is not null and brand <> ''
on conflict (name) do nothing;
```

Процесс изменения схемы — [schema.md §7](../../../../database/schema.md): применяем через MCP
`apply_migration` **и** дублируем файлом `0023_brands.sql` в репо cozycorner; синхронизация
типов `cozycorner/lib/types.ts` (добавить `Brand`), обновление
[schema.md §5/§6](../../../../database/schema.md) и `products.md`.

## 2. Слой данных (web.admin/src/lib)

Общая форма (categories и brands почти идентичны):

- `lib/categories.ts`:
  - `type Category = { id; name; slug; image_path: string|null; position }`
  - `listCategories(site)` — `id,name,slug,image_path,position` order by position.
  - `createCategory(site, { name })` — insert (slug/position — БД/дефолт).
  - `renameCategory(site, id, oldName, newName)` — update строки **и**
    `update products set category=newName where category=oldName`.
  - `deleteCategory(site, id, name)` — delete строки **и**
    `update products set category=null where category=name`.
  - `updateCategoryImage(site, id, image_path|null)` — только категории.
  - `reorderCategories(site, orderedIds)` — обновить position.
  - Оставить `listCategoryNames` как тонкую обёртку над `listCategories` (call sites
    в ProductsPage/фильтрах не переписывать разом) ИЛИ мигрировать их — решить в плане.
- `lib/brands.ts` — то же без image (`listBrands/createBrand/renameBrand/deleteBrand/
  reorderBrands`).

Каскад — app-level (два запроса), не атомарно; на текущих объёмах ок. При желании
позже — RPC-функция. Обработка конфликта уникальности name при create/rename — тост
об ошибке.

Query-ключи: `['categories', site.slug]` (уже есть), новый `['brands', site.slug]`;
после любых мутаций справочника инвалидировать и его ключ, и `['products', site.slug]`
(строки товаров показывают `brand · category`, а rename/delete их меняет).

## 3. Generic-абстракция таксономии

По образцу папок (`src/features/folders/` — один хук + один UI, различие в секции):

- `features/taxonomy/useTaxonomy.ts` — хук, принимает конфиг
  `{ kind: 'category'|'brand', hasImage: boolean }` и отдаёт данные + мутации
  (create/rename/delete/reorder/[setImage]) поверх соответствующего `lib/*`.
- `features/taxonomy/TaxonomyManager.tsx` — список строк с inline-rename (паттерн
  `FoldersPanel`: Input + ✓/✕, Enter/Esc, спиннер), создание (инлайн «New …»), delete
  (`AlertDialog` с «N products use this»), reorder (стрелки ↑/↓ по position), у
  `hasImage` — превью `ImagePreviewPicker` + `ImagePickerDialog` (сменить/убрать фото).
- `features/taxonomy/TaxonomyCombobox.tsx` — combobox для формы: выбор существующего,
  фильтр по вводу, «Create "<текст>"» при отсутствии совпадения (создаёт через
  `createCategory/createBrand`, инвалидирует, выбирает), пункт **Manage…** (открывает
  `TaxonomyManager` в модалке `Dialog`, унифицированной в спеке
  `2026-07-22-admin-dialog-layout-polish-design.md`). Пункт None — для необязательной
  категории/бренда (сентинел, как `__none__` сейчас).

Combobox требует shadcn-примитивов `command` + `popover` — добавить через CLI
(`npx shadcn@latest add command popover`), это разрешённый путь (не ручная правка ui/).

## 4. Интеграция в форму товара (ProductEditPage)

`src/features/products/ProductEditPage.tsx`:
- Заменить free-text `Input` Brand (строки ~379–382) и `Select` Category (строки
  ~420–441) на два `TaxonomyCombobox` (kind='brand' / 'category'). Значение — по-прежнему
  строка-имя (`products.brand` / `category`), контракт `toInput`/`toFormValues` не
  меняется (None → null для обоих; сейчас у brand null через `orNull`, у category через
  `__none__`).
- Убрать проп `categories: string[]` и загрузку `listCategoryNames` из страницы, если
  combobox грузит справочники сам (общий кэш `['categories'|'brands', site.slug]`).
- Гард несохранённых изменений товара не затрагиваем; создание справочника через combobox
  — **немедленная** мутация (как папки), не буферизуется формой (создаётся сущность, а
  значение поля товара выставляется `setValue(..., {shouldDirty:true})`).

## 5. Разделы в навигации

- `src/components/SiteLayout.tsx` — добавить в `NAV_ITEMS` пункты **Categories** и
  **Brands** (иконки lucide; подсветка активного пункта работает автоматически).
- `src/App.tsx` — роуты `/:siteSlug/categories` и `/:siteSlug/brands` внутри SiteLayout.
- Страницы `features/taxonomy/CategoriesPage.tsx` / `BrandsPage.tsx` — тонкие обёртки над
  `TaxonomyManager` с нужным конфигом (skeleton при загрузке, empty state, ошибка — красная
  плашка; паттерн списков из §3.7/§4 [components.md](../../../../admin-panel/components.md)).

## 6. Фильтры списка товаров (ProductsPage)

Сейчас фильтр Brand строится из уникальных `brand` загруженных товаров, Category — из
`listCategoryNames`. После появления таблицы `brands` источник фильтра Brand — справочник
`['brands', site.slug]` (полный список, а не только использованные). Обновить
[products.md §2](../../../../admin-panel/products.md) и [components.md §3.7](../../../../admin-panel/components.md)
соответственно.

## Затрагиваемые файлы

- **cozycorner**: `supabase/migrations/00NN_brands.sql` (+ схема сайта при необходимости),
  `lib/types.ts` (тип Brand).
- **web.admin**:
  - `src/lib/categories.ts` (новый, расширяет текущий `listCategoryNames` из products.ts),
    `src/lib/brands.ts` (новый); возможно перенос `listCategoryNames` из `lib/products.ts`.
  - `src/features/taxonomy/` — `useTaxonomy.ts`, `TaxonomyManager.tsx`, `TaxonomyCombobox.tsx`,
    `CategoriesPage.tsx`, `BrandsPage.tsx`.
  - `src/components/ui/command.tsx`, `src/components/ui/popover.tsx` (через CLI).
  - `src/features/products/ProductEditPage.tsx` (combobox вместо Input/Select).
  - `src/features/products/ProductsPage.tsx` (источник фильтра Brand).
  - `src/components/SiteLayout.tsx`, `src/App.tsx` (навигация + роуты).
- **Доки**: `schema.md` (§5/§6/§9/§10 — brands + миграция), `products.md`
  (§1 data-слой, §2 фильтры, §3 форма), `components.md` (§2 навигация, §3.7 форма/фильтры,
  §6 — Categories/Brands перестают быть «будущими»), `status.md`.

## Проверка (перед завершением)

- Миграция применяется, `brands` засижена из существующих `products.brand`.
- `npm run build` + `npm run lint` в web.admin.
- Ручной прогон: создать/переименовать/удалить бренд и категорию из раздела и из
  combobox в форме товара; после rename — связанные товары показывают новое имя;
  после delete — теряют ссылку; фото категории меняется; фильтры списка товаров видят
  полный справочник брендов.

## Риски / открытые вопросы к реализации

- ~~Write-политики RLS для brands~~ — **закрыто**: у `categories` есть `Admin write` (ALL,
  authenticated, `public.is_admin()`); миграция `brands` включает такую же (см. §1).
- ~~Схема cozycorner~~ — **закрыто**: таблицы контента живут в схеме `cozycorner`, функции —
  в `public`; `brands` создаётся сразу в `cozycorner`, дефолт-грант из 0017 дублируем явно (§1).
- **Каскад не атомарен** (rename/delete = 2 запроса) — на текущих объёмах приемлемо;
  зафиксировать как известное упрощение (аналогично «нет проверки использования товара
  в постах» из products.md §5).
- **Порядок реализации**: сначала миграция + слой данных, затем generic-абстракция,
  затем интеграция в форму и разделы навигации.
