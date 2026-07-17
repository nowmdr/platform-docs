# Раздел Products — правила и концепции

> Last updated: 2026-07-17 | Source project: web.admin (docs/products.md) — пути `src/…` относятся к репозиторию `web.admin/`

Документ фиксирует, как устроен раздел Products админки: контракты с данными,
поведение операций и решения по UI, принятые в ходе разработки (2026-07-10,
итерации v1–v4). Модель данных — [../database/schema.md](../database/schema.md) §5;
история решений — спеки `../archive/web.admin/superpowers/specs/2026-07-10-products-*.md`;
UX-детали — [components.md](components.md) §3.7. Читать этот файл ПЕРЕД любыми
правками раздела и перед началом CRUD других сущностей (categories и т.п.) —
многие решения переиспользуются.

## 1. Главные концепции и контракты

- **Слой данных — `src/lib/products.ts`**: `listProducts` (облегчённый select
  `id,title,created_at,brand,category,image_path,folder_id` для списка), `getProduct`, `createProduct`,
  `updateProduct`, `deleteProduct`, `listCategoryNames`. Всё через `getDb(site)` —
  схема активного сайта, RLS-запись только под админом.
- **slug не редактируется в админке.** БД-триггер `products_set_slug`
  (BEFORE INSERT OR UPDATE) генерирует slug из title, если он пуст; при create
  поле slug не отправляется вовсе, при edit показывается read-only.
- **`image_path` в БД — плоский ключ бакета ИЛИ внешний URL, никогда — абсолютный
  URL нашего Storage** (общий контракт — [api.md](api.md) §5). При этом в
  UI инпут Image path всегда показывает **публичную ссылку**: при чтении ключ
  разворачивается `resolveImageUrl`, при сохранении наш storage-URL сворачивается
  обратно `toStoragePath(site, value)` (`src/lib/images.ts`), внешние URL проходят
  как есть. Нарушать это нельзя — иначе переезд на другой Supabase-проект побьёт
  ссылки на сайте.
- **`description` — Markdown** (ограниченное подмножество: абзацы пустой строкой,
  списки `-`/`1.`, `**bold**`/`*italic*`; без ссылок/заголовков/кода). Редактор —
  generic `src/components/RichTextEditor.tsx` (TipTap v3 + официальный
  `@tiptap/markdown`; `contentType: 'markdown'`, чтение `editor.getMarkdown()`).
  Компонент переиспользуется в блоге и pages. Старые плоские описания —
  валидный markdown, фолбэк не нужен. Сайт cozycorner рендерит markdown
  компонентом `MarkdownText` (выполнено — см. [../sites/cozycorner.md](../sites/cozycorner.md)).
- **SEO-поля «живые»**: пустые `seo_title`/`seo_description` в БД (`null`) значат
  «сайт использует title/description товара». В `toInput` значение, равное (trim)
  соответствующему фолбэку или пустое, пишется как `null` — намеренно, чтобы SEO
  менялось вместе с товаром, а не замораживалось копией. UI: при пустых полях
  секция свёрнута (текст-подсказка + Customize), Customize предзаполняет поля
  текущими title/description БЕЗ пометки формы dirty, Use defaults очищает
  (`shouldDirty: true` — сравнение с defaults, лишнего dirty не будет).
- **Пустые optional-поля всегда `null`**, не пустые строки (`orNull` в `toInput`).
- **`category` товара = `categories.name`** (текст, не FK) — редактируется только
  Select'ом из справочника + пункт None (сентинел `__none__`: radix SelectItem
  не принимает пустое value; тот же приём — `__all__` в фильтрах списка).

## 2. Список (`src/features/products/ProductsPage.tsx`)

- Одна колонка строк-ссылок: title слева, `brand · category` приглушённо справа.
- Сортировка: `created_at desc` + **tiebreaker `title asc`** — у импортированных
  товаров timestamps совпадают пачками, без tiebreaker'а порядок нестабилен.
  Не убирать.
- Поиск (по title, `useDeferredValue`-паттерн из MediaPage) + фильтры **Brand,
  затем Category** (порядок как в строке товара): бренды — уникальные из
  загруженных товаров, категории — полный справочник `listCategoryNames`
  (query `['categories', site.slug]`, общий с формой). Фильтрация клиентская
  (товаров десятки), условия по AND; счётчик `видимые / все` при активном
  фильтре; ghost-кнопка Clear сбрасывает всё.

### Папки

- Слева от списка — панель папок (`FoldersPanel`), в строках — чекбоксы
  multi-select и иконка Move (`MoveToFolderMenu`), над списком — bulk-бар
  «N selected · Move to… · Clear». Канонические правила и контракты папок —
  [media.md](media.md) §3 (общий код — `src/features/folders/`, хук
  `useFolders`, секция `'products'` в `admin_folders`).
- `folder_id` есть **ТОЛЬКО** в списочном типе `ProductListItem` — в
  `Product`/`ProductInput` его нет: редактор и payload'ы create/update поле не
  трогают, новые товары попадают в Unsorted.
- Семантика фильтров: поиск — сквозь все папки; селекты Brand/Category — AND с
  активной папкой; Clear чистит поиск/селекты, папку не сбрасывает. Пустая
  папка без прочих фильтров — «This folder is empty.»

## 3. Форма (`src/features/products/ProductEditPage.tsx`)

- Один компонент на create (`products/new`) и edit (`products/:productId`);
  `ProductForm` монтируется только после загрузки данных — defaultValues
  задаются один раз, «мигания» формы нет.
- RHF + Zod v4 (`z.url()`, price — строка с refine, конвертация в число в
  `toInput`) + Field; Select'ы — через Controller.
- Query-ключи: список `['products', site.slug]` (инвалидация после всех
  мутаций), товар `['products', site.slug, id]`, категории
  `['categories', site.slug]`.
- Пикер картинок `ImagePickerDialog`: грид из `listImages` (query
  `['media', site.slug]` — ОБЩИЙ кэш с разделом Media, не заводить свой ключ),
  поиск, Upload одного файла через `uploadImage` из `lib/media` (вся валидация
  и откат там), выбранный/загруженный путь попадает в форму через
  `setValue(..., { shouldDirty: true })`; в БД пишется только по Save.
- Удаление — AlertDialog с предупреждением про продуктовые секции постов
  (точной проверки `post_product_sections.product_ids` нет — осознанное
  упрощение v1).
- **Раскладка v2** (спека/план `../archive/web.admin/superpowers/{specs,plans}/2026-07-10-blog-v2-layout*`,
  тот же паттерн, что у поста в [blog.md](blog.md) §3, без тоггла статуса —
  у товара его нет): sticky-панель действий сверху (`sticky top-0 z-10
  bg-background`) — слева Back, справа Save и Delete. Save вынесен из
  `<form>`: сабмит через `id="product-form"` + атрибут `form="product-form"`
  на кнопке. Грид-шапка `md:grid-cols-[16rem_1fr]` — превью image + slug
  слева, Title/Price+Brand/Image path+Gallery справа; ниже на всю ширину
  `max-w-3xl` — Referral URL, Category+Image style, Description, SEO.
  Невалидный Save — тост `'Fix validation errors before saving'` (паттерн
  поста, ранее был немым).

## 4. Гард несохранённых изменений (важно для будущих CRUD-форм)

- Приложение на **data router** (`createBrowserRouter` в `App.tsx`, экспорт
  `router`; `RouterProvider` в `main.tsx`) — это требование `useBlocker`;
  декларативный `<Routes>` не вернуть без потери гарда.
- Условие блокировки: `isDirty && !save.isPending && !remove.isPending`.
  `!isPending` обязателен: во время мутации гард снят, поэтому `navigate` из
  `onSuccess` (redirect на созданный товар, возврат после delete) не ловится
  собственным блокером.
- Модалка гарда: Stay (`blocker.reset`), Discard (`blocker.proceed`), Save —
  `form.handleSubmit`: валидна → `mutateAsync` → `proceed` (продолжаем
  прерванный переход; `leavingRef` подавляет внутренний redirect на созданный
  товар), невалидна/ошибка → `reset` (остаёмся, ошибки подсвечены).
- После успешного Save — `form.reset(values)` (значения становятся новыми
  defaults, форма чистая); после Delete — `form.reset()`.
- Закрытие/перезагрузка вкладки — `beforeunload` (навешивается только пока
  isDirty).
- При создании новых форм (categories и т.п.) переиспользовать этот паттерн
  целиком.

## 5. Известные хвосты / на потом

- Проверки «товар используется в постах» при удалении нет.
- Серверная пагинация/фильтрация не нужна на текущих объёмах (десятки строк).
- Гард несохранённых изменений есть в формах товара/поста/страницы; для новых
  форм — переносить паттерн §4.
