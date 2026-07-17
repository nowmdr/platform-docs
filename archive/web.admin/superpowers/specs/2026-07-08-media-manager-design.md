# Медиа-менеджер (фаза 2) + WP-подобный layout сайта — дизайн

Дата: 2026-07-08. Основание: `docs/admin-app-spec.md` §7 (контракт Storage/БД) + требования
пользователя (WP-подобная навигация, грид 3 колонки, копирование имени и публичной ссылки).

## Решения, принятые с пользователем
- Загрузка — **только кнопка** (скрытый `<input type="file" multiple>`), без drag-and-drop.
- Объём фазы — **полный паритет со спекой §7**: upload + list + delete + бейдж «used by».
- Pages и Products в меню — **кликабельные заглушки** («Coming soon»), маршруты существуют.

## Layout (вариант A)
AppShell не меняется (шапка: бренд → /dashboard, имя сайта, Sign out). Для `/:siteSlug/*`
добавляется `SiteLayout` (`src/components/SiteLayout.tsx`): слева вертикальное меню
(Media / Pages / Products, `NavLink` с подсветкой активного), справа `<Outlet/>`.
Site прокидывается дальше через outlet-context.

## Маршруты (App.tsx)
- `/:siteSlug` → redirect на `/:siteSlug/media`
- `/:siteSlug/media` → `MediaPage`
- `/:siteSlug/pages`, `/:siteSlug/products` → `ComingSoonPage`

## Слой данных
- `src/lib/images.ts` — `resolveImageUrl(site, path)`: внешний URL (`http(s)://`) как есть,
  иначе `${projectUrl}/storage/v1/object/public/${bucket}/${path}`.
- `src/lib/media.ts` (все запросы через `getDb(site)` / `getClient(site).storage`):
  - `listImages(site)`: `media` сорт `created_at desc`; «used by» — выборка path-полей из
    `products.image_path`, `categories.image_path|hero_image_path`,
    `posts.cover_image_path`, `hero_sections.bg_image_path`, `pages.og_image_path`,
    сравнение по прямому равенству path → `usedBy: string[]` (названия сущностей).
  - `uploadImage(site, file)`: валидация mime (jpeg/png/webp/gif) и size ≤ 6 МБ;
    sanitize имени (lowercase, `[^a-z0-9._-]→-`, схлопнуть/обрезать `-`); коллизии —
    суффикс `-2`, `-3`… по занятости `media.path`; `storage.upload(key, file,
    {contentType, upsert: false})`; insert в `media {original_name, path, mime, size}`;
    при ошибке insert — откат объекта из Storage.
  - `deleteImage(site, item)`: `storage.remove([path])` → delete строки `media` по id.
    Если фото используется — предупреждение в confirm-диалоге, но не блокировать.

## UI (`src/features/media/`)
- `MediaPage.tsx` — заголовок, кнопка Upload, query `['media', site.slug]` (TanStack),
  мутации инвалидируют кэш, тосты sonner. Мульти-выбор файлов грузится последовательно,
  ошибка одного файла не прерывает остальные.
- `MediaGrid.tsx` — `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`, skeleton, empty state.
- `MediaCard.tsx` — квадратное превью `object-cover`; имя файла + иконка copy;
  публичный URL (ellipsis) + иконка copy (`navigator.clipboard` + тост «Copied»);
  бейдж «Used by: …»; удаление — иконка trash → `AlertDialog` confirm.
- Копируется полный публичный URL; в БД — только относительный путь (контракт §7).

## Ошибки
RLS/сеть — тост с текстом ошибки; невалидные файлы отклоняются до загрузки с тостом.

## Проверка
`npm run build`, `npm run lint`; вручную на dev: логин → upload → грид/копирование →
delete; файл виден в бакете и строке `media`.
