# Taxonomy UX Refinement — Implementation Plan

> Iteration on `2026-07-22-category-brand-unification-plan.md` after user review.
> Project: web.admin (frontend only — БД уже готова). Верификация: `npm run build` + `npm run lint` + ручной прогон.

**Решения пользователя:**
- **Category** — строка списка: миниатюра (только показ, НЕ открывает пикер), имя, счётчик
  «N products», стрелки ↑/↓ (reorder), кнопка **Delete**. Три точки убраны. Вся строка —
  `cursor-pointer`, клик открывает **модалку-редактор**; клики по стрелкам/delete не всплывают.
- **Category modal** (create+edit, один компонент как ProductEditPage) — «средний» набор
  полей: картинка-карточка (`image_path`), `name`, `hero_title`, `hero_description`, SEO
  (`seo_title`/`seo_description`, живой фолбэк). Hero_badge/hero_image_path НЕ трогаем.
- **New category** — сверху над списком, открывает модалку в create-режиме.
- **Brand** — зеркалит структуру: New сверху; строка: имя + счётчик + стрелки + кнопка
  rename (инлайн-инпут) + кнопка delete. Три точки убраны, модалки нет.
- **Reorder** — оптимистичный (мгновенно в UI) + конкурентная запись (Promise.all), без лага.

## Task A: Data layer — categories.ts

- Расширить `Category` (для модалки нужны hero/seo). List оставить лёгким.
- Добавить `getCategory(site, id): Promise<CategoryFull | null>` — select всех полей.
- Добавить `createCategoryFull(site, input): Promise<{ id: string }>` — insert всех
  редактируемых полей (name + image_path + hero_title + hero_description + seo_title +
  seo_description). `createCategory(site, name)` оставить для combobox/inline.
- Добавить `updateCategory(site, id, oldName, input): Promise<void>` — update строки +
  каскад в products при смене name (как renameCategory).
- `reorderCategories` — заменить последовательный for-await на `Promise.all` (конкурентно).
- `brands.ts`: `reorderBrands` — то же (Promise.all).

`CategoryInput = { name; image_path: string|null; hero_title: string|null;
hero_description: string|null; seo_title: string|null; seo_description: string|null }`.

## Task B: CategoryEditDialog.tsx (features/taxonomy/)

- Пропсы: `{ site; id: string | null; open; onOpenChange }`. id=null → create.
- RHF + Zod (name required; остальное optional-строки, пустые → null в toInput).
- Загрузка при edit: `useQuery(['categories', site.slug, id], getCategory)`, монтировать
  форму после загрузки (как ProductForm).
- Лаяут (в `DialogBody`): `ImagePreviewPicker` (image_path, square, contain) + Gallery-кнопка
  → `ImagePickerDialog` (onSelect кладёт плоский путь), «Remove image» когда картинка есть;
  Name; Hero title; Hero description (Textarea); SEO-секция (сворачиваемая, живой фолбэк:
  seo_title→name, seo_description→hero_description — как у товара).
- Save: create → `createCategoryFull`; edit → `updateCategory(...oldName...)`. Инвалидация
  `['categories', site.slug]`, `['products', site.slug]`, `['categories', site.slug, id]`;
  toast; onOpenChange(false).
- image_path контракт (api.md §5): в БД плоский ключ; picker отдаёт ключ — писать как есть.

## Task C: config.ts

- В `TaxonomyConfig` добавить `editor?: React.ComponentType<{ site; id: string | null;
  open: boolean; onOpenChange: (o: boolean) => void }>`.
- `categoryConfig.editor = CategoryEditDialog`. `brandConfig` — без editor.

## Task D: TaxonomyManager.tsx rework

- **New-кнопка сверху** (над списком) для обоих: если `config.editor` → открывает editor(id=null);
  иначе → инлайн create (как сейчас, но сверху).
- **Строки**:
  - Если `config.editor` (category): весь ряд — `role=button`, `cursor-pointer`, hover-подсветка,
    onClick → editor(item.id). Внутри: миниатюра (display-only, БЕЗ onClick-пикера), имя, счётчик,
    стрелки ↑/↓ и Delete — обёрнуты в `onClick={(e)=>e.stopPropagation()}` (не открывают модалку).
    Инлайн-rename и dots-меню убраны.
  - Если нет editor (brand): ряд не кликается целиком; имя + счётчик + стрелки + кнопка rename
    (инлайн-инпут, как в FoldersPanel) + кнопка Delete. Dots убран.
- **Reorder оптимистичный**: `move()` — `queryClient.setQueryData(key, reordered)` мгновенно,
  затем `reorder.mutate(ids)`; onError — invalidate (откат). Persist конкурентный (Task A).
- Delete-AlertDialog — общий (как сейчас). Editor рендерится, если `config.editor`.

## Task E: Верификация + доки

- `npm run build` + `npm run lint` (≤2 известных warning).
- Ручной прогон: категории (New→modal create, клик по ряду→edit, стрелки без лага и без
  открытия модалки, delete), бренды (New сверху, inline rename, delete, стрелки).
- Обновить components.md §3.11 (новая механика строк + модалка категории).
