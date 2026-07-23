# Админ-панель — UX/UI структура и компоненты

> Last updated: 2026-07-23 | Source project: web.admin (docs/ux-ui.md, актуально на 2026-07-16) — пути `src/…` относятся к репозиторию `web.admin/`

Полное описание интерфейса: карта навигации, экраны, иерархия компонентов, паттерны
взаимодействия и состояния. Дополняет [media.md](media.md) (правила раздела Media и
папок), [products.md](products.md), [blog.md](blog.md), [pages.md](pages.md) и
[api.md](api.md) (контракт с данными).

## 1. Дизайн-система

- **База** — Tailwind v4 + shadcn/ui (библиотека radix, пресет **nova**, base color
  neutral). Готовые компоненты — в `src/components/ui/*`, добавляются CLI
  (`npx shadcn@latest add <name>`), вручную не редактируются.
- **Шрифт** — системный стек SF (`-apple-system, …` в `--font-sans`); отдельных
  webfonts нет.
- **Тема** — светлая (токены в `src/index.css`; dark-набор токенов существует,
  переключателя нет — на потом).
- **Иконки** — lucide-react, размер по умолчанию в кнопках shadcn; в компактных
  местах — явные `size-3.5` / `size-4`.
- **Язык интерфейса — английский** (лейблы, плейсхолдеры, тосты, диалоги).
  Доки и комментарии в коде — русский.
- **Глобальные правила** (`src/index.css`): `cursor: pointer` для всех
  `button:not(:disabled)` и `[role="button"]` (Tailwind v4 preflight его не ставит);
  фокус — кольцо `outline-ring/50`; скроллбар — `scrollbar-gutter: stable` на html
  (ширина контента не прыгает при появлении скролла) + тонкий минималистичный вид
  (`scrollbar-width/color`, для Safari — `::-webkit-scrollbar` фолбэк).
- **Обратная связь** — тосты sonner (`<Toaster richColors position="top-center"/>`
  в `main.tsx`): success — зелёный, error — красный. Ошибка формы — под полем
  (Field + FieldError), ошибка запроса — тост или красная плашка на странице.

## 2. Карта навигации

```
/login                        — вход (публичный)
/dashboard                    — выбор сайта (guard)
/:siteSlug                    — redirect → /:siteSlug/media (guard)
/:siteSlug/media              — медиа-менеджер
/:siteSlug/pages              — список страниц (+ строка «Header» сверху)
/:siteSlug/pages/header       — сайт-глобальные настройки шапки (header_settings)
/:siteSlug/pages/:pageId      — детальная страница (SEO + hero-блоки + body)
/:siteSlug/products           — список товаров
/:siteSlug/products/new       — создание товара
/:siteSlug/products/:productId — детальная страница товара (edit/delete)
/:siteSlug/categories         — категории каталога (CRUD, фото)
/:siteSlug/brands             — бренды каталога (CRUD, без фото)
/:siteSlug/blog               — список постов блога
/:siteSlug/blog/new           — создание поста
/:siteSlug/blog/:postId       — детальная страница поста (edit/delete)
/:siteSlug/seo-posts{,/new,/:postId} — SEO-посты (те же компоненты, конфиг sections.ts)
*                             — redirect → /dashboard
```

Роутер — data-режим react-router (`createBrowserRouter` в `App.tsx`, экспорт
`router`; `RouterProvider` в `main.tsx`) — нужен для `useBlocker` (гард
несохранённых форм). Иерархия layout'ов (вложенные Outlet):

```
RequireAuth (guard: сессия + is_admin)
└─ AppShell (шапка)
   ├─ DashboardPage                  — для /dashboard
   └─ SiteLayout (левое меню сайта)  — для /:siteSlug/*
      └─ MediaPage | ProductsPage | ProductEditPage | PostsPage | PostEditPage
         | PagesPage | PageEditPage
```

Контекст: `RequireAuth` кладёт активный `SiteConfig` в outlet-context, ниже он
прокидывается через каждый Outlet (`AppShell` → `SiteLayout` → страница).

## 3. Экраны

### 3.1 Login (`/login`, `features/auth/LoginPage.tsx`)
- Центрированная карточка `max-w-sm` на весь экран: заголовок «Site Admin»,
  подзаголовок «Administrator sign-in», поля Email / Password, кнопка Sign in
  на всю ширину.
- Валидация RHF+Zod, ошибки под полями; при сабмите кнопка блокируется
  («Signing in…»).
- Ошибки входа — тостами: неверные креды («Invalid email or password»), не-админ
  («Access denied: user is not on the admin whitelist», сессия сбрасывается).
- UX-деталь: гард передаёт исходный путь в `state.from` — после логина возврат туда,
  откуда выкинуло; сессия кладётся в кэш query до navigate (иначе гард вернёт на
  /login — известный баг первого логина).

### 3.2 Guard-состояния (`components/RequireAuth.tsx`)
- **Загрузка** — skeleton-каркас (полоса шапки + блок контента).
- **Нет сессии** — redirect `/login` c `state.from`.
- **Не админ** — полноэкранное «Access denied» с email пользователя и кнопкой
  Sign out (единственный выход).

### 3.3 Шапка (`components/AppShell.tsx`)
- Высота 14, нижняя граница; контент по центру `max-w-7xl`.
- Слева бренд **Site Admin** — ссылка на `/dashboard` (это и есть «выход к выбору
  сайта», отдельного свитчера нет — решение вместо SiteSwitcher).
- Рядом — «/ {site.label}», только внутри раздела сайта.
- Справа — ghost-кнопка **Sign out** → signOut + redirect `/login`.

### 3.4 Dashboard (`/dashboard`, `features/dashboard/DashboardPage.tsx`)
- Грид карточек сайтов (1/2/3 колонки — mobile/sm/lg) из реестра `SITES`.
- Карточка: название + `schema · bucket`, hover — подсветка `accent/50`,
  клик — переход в `/:siteSlug` (→ media). Это единственная точка выбора
  активного сайта.

### 3.5 Раздел сайта (`components/SiteLayout.tsx`)
- Двухколоночный flex: слева фиксированное меню `w-44`, справа контент
  (`min-w-0 flex-1` — контент не распирает layout).
- Меню (WP-паттерн): **Media / Pages / Products / Categories / Brands / Blog /
  SEO Posts**, у пункта иконка
  + подпись. Активный пункт — заливка `accent`; неактивный — приглушённый текст,
  hover — полупрозрачная заливка.
- Все пункты — рабочие разделы (заглушек «Coming soon» не осталось, компонент
  ComingSoonPage удалён).

### 3.6 Media (`/:siteSlug/media`, `features/media/*`)

**Шапка страницы** (одна строка): заголовок «Media» → (справа) поле поиска
`max-w-56` с иконкой-лупой → primary-кнопка **Upload** (иконка + текст;
при загрузке — disabled + «Uploading…»). Файловый input скрыт, множественный
выбор, accept — jpeg/png/webp/gif.

**Поиск**: клиентский фильтр по имени и пути по мере ввода; на время применения
(deferred) лупа меняется на спиннер, грид гаснет до 60% opacity. Пустой
результат — «No images match your search.» Поиск идёт **сквозь все папки**
(активная папка игнорируется, пока строка непустая).

**Папки** (общий код разделов — `src/features/folders/`, правила —
[media.md](media.md) §3): слева от грида `aside` `w-44 lg:w-52` с панелью
`FoldersPanel` — пункты All / Unsorted / папки по алфавиту со счётчиками,
инлайн-создание («New folder») и переименование (Enter/Esc, ✓/✕), меню «…» по
ховеру (Rename/Delete), delete — AlertDialog «N images will be moved to
Unsorted». Множественный выбор — через **режим выбора** (§4): кнопка **Select** в
шапке (по умолчанию чекбоксов нет), в режиме — чекбокс-оверлей в левом верхнем
углу каждой карточки (`selectable`) и панель `SelectionBar` на месте поиска/Upload.
Одиночное перемещение — кнопка Move в футере карточки и строка Folder в модалке
деталей (текущая папка исключена из целей). Upload кладёт файлы в открытую сейчас
папку (All/Unsorted → без папки).

**Грид** (`MediaGrid`): 2/3/4 колонки (mobile/sm/lg), gap-3; карточки одной
высоты. Первичная загрузка — 12 квадратных skeleton'ов; пустой список —
пунктирная плашка «No images yet. Upload the first one.»; пустая папка —
«This folder is empty.»

**Карточка** (`MediaCard`), сверху вниз:
1. Квадратное превью `object-cover`, клик открывает модалку деталей
   (`cursor-zoom-in`, вся зона — кнопка с aria-label).
2. Маркеры-иконки на превью (правый верхний угол, тултипы):
   - тёмный кружок с Unlink — «Not used anywhere — safe to delete»;
   - янтарный кружок с TriangleAlert — «Larger than 1 MB».
3. Имя файла (truncate, полный текст в title) и размер (`formatBytes`) —
   компактные xs/11px.
4. Футер (граница сверху, фон muted): ghost-кнопки **URL** и **Name**
   (копирование публичного URL / имени, тост «… copied to clipboard»),
   справа — **Move to folder** (`MoveToFolderMenu`) и красная корзина
   (icon-only).
Текстового бейджа «used by» на карточке нет — только маркер и модалка.

**Модалка деталей** (`MediaDetailsDialog`, до `max-w-4xl`, сетка 3:2):
- Слева квадратный врапер на всю ширину колонки (muted-фон, рамка): большая
  картинка вписывается `object-contain`, маленькая — натуральный размер по
  центру (без апскейла) — форма превью стабильна для любого разрешения.
- Справа список метаданных: File name (с карандашом → инлайн-переименование:
  Input + ✓/✕, Enter/Esc, спиннер на время запроса), Size, Type, Dimensions
  (px, читаются из загруженного изображения), Uploaded (en-GB локаль),
  Public URL (полностью, break-all), Folder (имя папки + `MoveToFolderMenu`),
  Used by (бейджи-«где используется» или «Not used anywhere — safe to delete»).
- Низ правой колонки: outline-кнопки **Copy URL**, **Copy name**, справа —
  красная **Delete**.

**Подтверждение удаления** (AlertDialog, из карточки и из модалки):
- заголовок `Delete "<имя>"?`;
- если используется — перечисление мест и предупреждение, что картинки на сайте
  сломаются; удаление при этом **не блокируется**;
- если нет — «The file will be permanently removed from storage.»;
- Cancel / Delete; после удаления из модалки она закрывается.

### 3.7 Products (`/:siteSlug/products`, `features/products/*`)

**Список** (`ProductsPage`): шапка — заголовок «Products» + счётчик, справа
primary-кнопка **New product** (→ `products/new`). Под шапкой — панель
поиска/фильтров: поле поиска по названию (`max-w-56`, deferred-паттерн как в
Media — лупа ↔ спиннер, список гаснет), Select **Brand** («All brands» +
уникальные бренды из загруженных товаров, по алфавиту), затем Select
**Category** («All categories» + полный справочник из таблицы `categories`) —
порядок как в строке товара (`brand · category`). Условия совмещаются
по AND, фильтрация клиентская; при активном фильтре счётчик — `видимые / все`,
в конце панели появляется ghost-кнопка **Clear** (X) — сброс поиска и фильтров,
и (при непустом списке) outline-кнопка **Select** справа — вход в режим выбора (§4).
Ниже — flex: слева `aside` `w-44 lg:w-52` с панелью папок (`FoldersPanel`,
секция `products`; та же механика, что в Media — см. 3.6 и [media.md](media.md) §3),
справа колонка списка. Строка списка — flex-ряд:
чекбокс выборки (только в режиме выбора; сосед ссылки, не вложен в неё) +
строка-ссылка (сортировка
`created_at desc` + tiebreaker `title asc` — у импортированных товаров
timestamps совпадают; новые сверху): название слева, приглушённое
`brand · category` справа (только заполненные); hover — заливка `accent/50`,
клик → детальная страница; справа иконка Move (`MoveToFolderMenu` с
`currentFolderId` строки). В режиме выбора строка поиска/фильтров **заменяется**
панелью `SelectionBar` «N selected · Move to… · Delete · Cancel» (`BulkDeleteButton`
+ `useBulkDelete`, confirm-диалог «Delete N products?») — без вертикального сдвига
списка. Семантика фильтров: поиск — сквозь все папки,
селекты Brand/Category — AND с активной папкой, Clear папку не сбрасывает.
Состояния: 8 skeleton-строк при загрузке; «No products yet.»; «No products
match your filters.» при пустом результате фильтра; «This folder is empty.»
для пустой папки без прочих фильтров; ошибка — красная плашка.

**Детальная страница** (`ProductEditPage`, общая для edit и new), раскладка v2
(спека `../archive/web.admin/superpowers/specs/2026-07-10-blog-v2-layout-design.md`):
- **Sticky-панель действий** (`sticky top-0 z-10 bg-background`, работает, т.к.
  шапка AppShell не sticky): слева ссылка «← Back to products», справа **Save**
  и красная **Delete**. Save стоит вне `<form>` — сабмит через `id="product-form"`
  на форме + атрибут `form="product-form"` на кнопке. В состояниях
  loading/error/not-found панели нет — родитель рендерит только ссылку назад.
- **Грид-шапка формы** `md:grid-cols-[16rem_1fr]` (на mobile — одна колонка):
  слева квадратное превью картинки — общий компонент `ImagePreviewPicker`
  (`src/components/ImagePreviewPicker.tsx`, object-contain; плейсхолдер ImageOff
  при пустом/битом пути). Превью **кликабельно**: по ховеру/фокусу — оверлей с
  кнопкой **Add image** (пусто) / **Change image** (есть картинка), открывает тот
  же пикер (2026-07-22). Под превью read-only slug; справа — Title*, ряд Price*
  (`grid-cols-[8rem_1fr]` — Price узкий, `type=text inputMode=decimal`, без
  спиннеров) + Image path (инпут + outline-кнопка **Gallery** → диалог-пикер, см.
  ниже), затем Referral URL* (валидный URL). `ImagePreviewPicker` сам ведёт
  `broken`-состояние и сбрасывает его при смене `url` — call site стейт не отслеживает.
- Ниже грида на всю ширину формы (`max-w-4xl`): ряд Category + Brand (оба
  `TaxonomyCombobox` — выбор/создание + Manage), Description (markdown-редактор, см.
  ниже), секция SEO. **Image style в вёрстке нет** (2026-07-22): поле/схема/`toInput`
  целы, для новых товаров дефолт `cutout`, у существующих — сохранённое значение.
  Пустые optional-поля пишутся в БД как `null`. Невалидный Save —
  тост «Fix validation errors before saving» (кнопка в панели, инлайн-ошибка
  может быть за экраном).
- Description — `RichTextEditor` (`src/components/RichTextEditor.tsx`, generic —
  переиспользуется в блоге): TipTap v3 + официальный `@tiptap/markdown`, тулбар
  Bold/Italic/списки, без ссылок и заголовков. В БД — Markdown (абзацы пустой
  строкой, списки `-`/`1.`); старые плоские описания валидны как markdown.
- Секция SEO: если оба seo-поля пусты — свёрнутый блок (пунктирная рамка,
  «Search engines use the product title and description.», кнопка **Customize**);
  Customize разворачивает поля, предзаполненные текущими title/description
  (не делает форму dirty), **Use defaults** очищает и сворачивает. При
  сохранении пустое или совпадающее с title/description значение → `null`
  (фолбэк остаётся «живым»).
- Image path в инпуте — всегда **публичная ссылка** (контракт — [api.md](api.md) §5).
- Гард несохранённых изменений: пока форма dirty, любая внутренняя навигация
  (ссылка Back, браузерная «назад») блокируется `useBlocker` → AlertDialog
  «Discard unsaved changes?» (Stay / Discard / **Save** — сохранить и продолжить
  прерванный переход; невалидная форма или ошибка — остаёмся с подсветкой
  ошибок); закрытие или обновление вкладки —
  нативный prompt (`beforeunload`). После успешного Save/Delete форма
  сбрасывается в чистое состояние и уход свободен.
- Пикер картинки (`ImagePickerDialog`, `sm:max-w-3xl`): **фильтр по папкам**
  (`Select` All/Unsorted/папки со счётчиками, слева от поиска — 2026-07-22),
  поиск по имени/пути (сквозь все папки), кнопка **Upload** (один файл,
  `uploadImage` из `lib/media` — грузит в выбранную папку, валидация, sanitize,
  откат; после загрузки путь сразу подставляется и диалог закрывается), грид
  миниатюр 3/4/5 колонок из `listImages` (общий query-кэш `['media', site.slug]`
  с разделом Media), текущая картинка подсвечена ring. Клик по миниатюре →
  `image_path` в форму (`setValue`, `shouldDirty`); запись в БД — только по Save.
- Кнопки (в sticky-панели): **Save** (блокируется на время запроса, «Saving…»);
  в режиме edit — красная **Delete** через AlertDialog (предупреждение, что
  товар исчезнет из продуктовых секций постов; после удаления — возврат к
  списку).
- Создание: slug не отправляется (БД автогенерирует из title), после Save —
  переход на страницу созданного товара. Кривой id — «Product not found.»
- Данные: `lib/products.ts`; ключи Query — [api.md](api.md) §6.

### 3.8 Blog (`/:siteSlug/blog`, `features/posts/*`)

Правила и контракты раздела — [blog.md](blog.md) (читать перед правками).

**Список** (`PostsPage`): шапка — заголовок «Blog» + счётчик, справа
primary-кнопка **New post** (→ `blog/new`). Панель фильтров: поле поиска по
заголовку (`max-w-56`, deferred-паттерн как в Media) + Select **Status**
(«All statuses» / Published / Draft, сентинел `__all__`); условия по AND,
фильтрация клиентская; при активном фильтре счётчик `видимые / все` и
ghost-кнопка **Clear** (сбрасывает и поиск, и статус, папку не трогает).
Ниже — flex: слева панель папок (`FoldersPanel`, секция `posts` — та же
механика, что в Products/Media, см. 3.7), справа колонка строк. Строка —
чекбокс (только в режиме выбора) + строка-ссылка (сортировка `published_at desc`
+ tiebreaker `title asc`): title слева, справа приглушённо бейдж **Draft** (при
`is_published=false`) + дата `formatDate(published_at)`; справа иконка Move.
Режим выбора (кнопка **Select**, `SelectionBar` на месте фильтров) и семантика
фильтров — как в Products (поиск сквозь папки, Status — AND с папкой). Состояния: 8 skeleton-строк; «No posts
yet.»; «No posts match
your filters.»; «This folder is empty.»; ошибка — красная плашка.

**Детальная страница** (`PostEditPage`, общая для edit и new) — та же
v2-раскладка, что у товара:
- **Sticky-панель**: слева «← Back to blog»; справа тоггл **Published/Draft**
  (Switch на `is_published` с текстовым лейблом), **Save**
  (`form="post-form"`), **Delete**. Поля Status в теле формы нет — тоггл
  только в панели.
- **Грид-шапка**: слева превью cover — `ImagePreviewPicker` (`aspect-video`, на
  md+ квадрат, object-cover) с hover-кнопкой **Add/Change image** (открывает
  пикер) + slug; справа Title*, Excerpt (textarea), Cover image (инпут публичной
  ссылки + **Gallery** → общий `ImagePickerDialog` из products).
- Ниже: **Published at** (`datetime-local`, `max-w-xs`; при каждом Save секунды
  обрезаются до минут), конструктор **Content** (см. ниже), секция SEO
  (живой фолбэк `seo_title`→title, `seo_description`→excerpt — механика
  products). Невалидный Save — тост + инлайн-ошибки. Гард несохранённых
  изменений — тот же паттерн (useBlocker + beforeunload).

**Конструктор модулей** (`PostModulesEditor`) — тело поста как единый
упорядоченный стек карточек-модулей (позиция в БД = индекс в списке; тип
модуля ↔ таблица `post_text_sections`/`post_product_sections`):
- Шапка карточки: иконка + тип («Text» / «Products»), пометка «(hidden)» у
  скрытых; справа контролы: глазик (per-module `is_published`; скрытый модуль
  приглушён `border-dashed opacity-60`, но валидацию проходит), ↑/↓
  (disabled на краях), корзина.
- **Text**: Heading (опционально) + `RichTextEditor` (тот же markdown-редактор,
  что в description товара).
- **Products**: Heading, переключатель сетки **2/3 columns** (aria-pressed),
  строки выбранных товаров (превью + название; ↑/↓/крестик — порядок строк =
  `product_ids` = порядок карточек на сайте; удалённый товар — «Unknown
  product (deleted?)», пока справочник грузится — «Loading…»), кнопка
  **Add products** → пикер.
- Внизу кнопки **+ Text** и **+ Products** (новый products-модуль — 3 колонки
  по умолчанию). Пустое тело — пунктирная плашка-подсказка.
- Валидация: body текстового модуля непустой; в products-модуле ≥ 1 товара.
- **Пикер товаров** (`ProductPickerDialog`, `sm:max-w-2xl`): поиск по названию,
  мультивыбор строк (чек-индикатор, aria-pressed), уже добавленные в модуль
  товары скрыты; футер Cancel / **Add N products** — выбор отдаётся пачкой.
  Кэш — общий `['products', site.slug]`.

**Сохранение** — одна кнопка Save на весь пост: поля поста + diff секций
(insert новых / update существующих / delete удалённых — в конце); после Save
форма пересинхронизируется свежими данными из БД (иначе повторный Save
продублировал бы новые секции — детали в [blog.md](blog.md) §1). Удаление поста —
AlertDialog; секции удаляет каскад БД.

### 3.9 SEO Posts (`/:siteSlug/seo-posts`)

Тот же UI, что Blog (§3.8): разделы рендерят одни компоненты, различия — только
конфиг-объект `PostsSectionConfig` (`src/features/posts/sections.ts`). Правила —
[blog.md](blog.md) §1a.

### 3.10 Pages (`/:siteSlug/pages`, `features/pages/*`)

Правила — [pages.md](pages.md). Список: одна колонка строк-ссылок (`slug` +
бейдж «N hero» + `seo_title`), без поиска/фильтров/New — 7 строк. Форма: только
edit; sticky-панель Back + Save; грид-шапка (превью OG image слева —
`ImagePreviewPicker`, hover **Add/Change image** — + slug; SEO title / SEO
description / OG image + Gallery справа); Body (`RichTextEditor`, только
terms/privacy); hero-секции (`HeroSectionsEditor` — только update существующих,
глазик/badge/title/description/bg image/align left|right). У каждой hero-секции —
превью bg image (`ImagePreviewPicker`, `max-w-64`) с той же hover-кнопкой над
инпутом + Gallery (2026-07-22).

### 3.11 Categories / Brands (`/:siteSlug/{categories,brands}`, `features/taxonomy/*`)

Два раздела-справочника на одном generic-компоненте `TaxonomyManager` (различие —
конфиг `categoryConfig`/`brandConfig`: `hasImage` true/false; у категории задан
`editor` — компонент модалки). Верхняя панель: кнопка **New** + поле **поиска** по
имени (клиентский фильтр; при активном поиске стрелки reorder отключены — перестановка
имеет смысл только на полном списке). Список — простой borderless-стиль строк, как в
Products/Blog (кликабельная зона с hover-подсветкой + иконки-действия соседями, без
рамки-карточки на каждой строке). Общее в строке:
счётчик «N products» (из общего кэша `['products', site.slug]`), стрелки ↑/↓
(reorder — **оптимистичный**: порядок меняется в кэше мгновенно, запись позиций
конкурентная `Promise.all`), кнопка **Delete** (`AlertDialog` «N products use this …
will lose it» — не блокирует; ссылка у товаров обнуляется). Мутации немедленные,
rename/delete каскадят в `products` (см. [products.md](products.md) §1).

- **Categories** (`editor` задан): миниатюра-превью (только показ, клик по ней НЕ
  открывает пикер), имя, счётчик, стрелки, Delete. **Весь ряд `cursor-pointer`** —
  клик открывает **`CategoryEditDialog`** (кнопки-действия — соседи кликабельной зоны,
  не всплывают). New открывает ту же модалку в create-режиме. Модалка (один компонент
  create+edit, как ProductEditPage): картинка-карточка — `ImagePreviewPicker` +
  **инпут Image path** (показывает публичную ссылку; внешний URL можно вписать вручную)
  + Gallery — тот же контракт, что у товара ([api.md](api.md) §5: в БД плоский ключ или
  внешний URL, в UI — публичная ссылка; `resolveImageUrl`/`toStoragePath`). Далее Name,
  Hero title, Hero description (Textarea), SEO-секция со свёрткой и живым фолбэком
  (seo_title→name, seo_description→hero_description). Поля `hero_badge` и
  `hero_image_path` в этот заход не редактируются.
- **Brands** (без `editor`): имя, счётчик, стрелки, кнопка **Rename** (инлайн-инпут
  Enter/Esc/✓✕, как у папок) и **Delete**. Модалки нет.

Тот же справочник доступен из формы товара через `TaxonomyCombobox` (выбор/создание +
Manage… открывает `TaxonomyManager` в модалке). Все пункты поповера — `cursor-pointer`
и внутри `CommandGroup` (единый инсет `p-1`), чтобы фон при наведении не «налазил»
ближе к краю, чем у сгруппированных элементов (`ui/command.tsx` руками не правим —
только через `className` на `CommandItem`; 2026-07-22).

## 4. Паттерны взаимодействия

- **Копирование в буфер** — всегда кнопка с иконкой + тост об успехе/ошибке
  (`lib/clipboard.ts`); никаких «скопируй текст руками».
- **Опасные действия** — только через AlertDialog с описанием последствий;
  предупреждение информирует, но не запрещает (решение за админом).
- **Мутации** — TanStack Query: инвалидация ключа после мутаций; кнопки
  блокируются на время запроса; результат — тост.
- **Сообщения об ошибках** (`src/lib/errors.ts`) — релевантные, а не общие:
  невалидный сабмит формы показывает КОНКРЕТНОЕ сообщение поля
  (`firstFieldErrorMessage(errors)` в invalid-колбэке `handleSubmit`, фолбэк
  «Please fix the highlighted fields»), а не «Fix validation errors». Ошибки мутаций
  прогоняются через `humanizeError(e, fallback)` — мапит коды Postgres (23505
  дубликат → «That name is already taken…», 23503/23502, RLS/JWT) в понятный текст,
  иначе исходное сообщение. Применять в новых формах/CRUD.
- **Длинные строки** (имена, URL) — truncate + полный текст в `title`;
  в модалке URL показывается целиком с `break-all`.
- **Загрузка данных** — skeleton на первом заходе, спиннеры на локальных
  операциях; ошибка загрузки списка — красная плашка с текстом ошибки.
- **Доступность** — icon-only кнопки всегда с `aria-label`; поля форм — label +
  `aria-invalid`; фокус видим (кольцо); модалки/диалоги — radix (фокус-трап, Esc).
- **Модалки** — единый примитив `ui/dialog.tsx`: `DialogContent` — flex-колонка,
  ограничена `max-h-[85svh]` (минимум по контенту), середина — `DialogBody` (скроллится,
  `-mx-4 px-4` держит контент на всю ширину, но сохраняет отступ, чтобы кольца/индикаторы
  у краёв не срезались), header/footer фиксированы. Ручные `max-h-[NNsvh]` на внутренних
  div не заводить — использовать `DialogBody`.
- **Выбор картинки** — везде, где у поля есть превью (шапки редакторов товара/
  поста/страницы, bg hero-секций), превью — общий `ImagePreviewPicker`
  (`src/components/ImagePreviewPicker.tsx`): кликабельный `<button>` с ImageOff-
  плейсхолдером и hover/focus-оверлеем **Add/Change image**, открывающим пикер.
  Сам ведёт `broken`-состояние (сброс при смене `url`) — call site стейт не
  держит. Рядом остаётся текстовый инпут пути + кнопка Gallery (внешние URL).
- **Bulk-удаление** — общий `BulkDeleteButton` (destructive, проп `disabled` при
  0 выбранных) + хук `useBulkDelete<T>` (`src/features/folders/useBulkDelete.ts`):
  доменная `deleteOne` из раздела, последовательное удаление, инвалидация + чистка выборки.
- **Режим выбора** (media/products/posts, 2026-07-22) — чекбоксов по умолчанию нет.
  Кнопка **Select** (иконка `ListChecks`) включает `selectionMode` (стейт в общем
  `useFolders`): появляются чекбоксы, а панель bulk-действий `SelectionBar`
  (`src/features/folders/SelectionBar.tsx`: «N selected · Move to… · <Delete-слот> ·
  Cancel») встаёт **на место** строки поиска/фильтров — та же высота (`min-h-9`),
  нулевой вертикальный сдвиг списка. Кнопку удаления страница передаёт слотом
  (`children`) — у media она несёт своё предупреждение о занятых картинках. **Cancel**
  (`exitSelection`) выходит и очищает выбор. Триггер выбора — `onClick` на чекбоксе
  (radix Checkbox — это `<button>`, событие приходит и с мыши, и со Space).
- **Shift-выбор диапазона** — `selectRange(id, shiftKey, orderedIds)` в `useFolders`
  держит якорь (последний кликнутый); Shift+клик добавляет в выборку весь диапазон
  **видимого** порядка от якоря до клика. Работает во всех разделах (в media —
  через `onToggleSelect(item, shiftKey)` в `MediaCard`).

## 5. Отступы и размеры (ориентиры)

- Контент страницы — `max-w-7xl`, паддинг 4; шапка h-14.
- Меню сайта — `w-44`, зазор с контентом gap-6.
- Грид медиа — gap-3; внутри карточки паддинги 2, футер p-1, кнопки h-7.
- Радиусы и тени — дефолты shadcn (rounded-xl у карточек).

## 6. Будущие разделы (footer_settings, …)

> Categories и Brands реализованы (2026-07-22, §3.11) — generic `TaxonomyManager` +
> `TaxonomyCombobox`; здесь остаются как образец паттерна для новых CRUD-разделов.


- Добавлять пункт в `NAV_ITEMS` (`SiteLayout`) и маршрут в `App.tsx` — меню и
  подсветка активного пункта работают автоматически.
- Держаться паттернов: список = грид/таблица с skeleton и empty state; мутации —
  через Query с инвалидацией и тостами; опасное — через AlertDialog; формы —
  RHF+Zod+Field; гард несохранённых изменений — паттерн [products.md](products.md) §4.
- Выбор картинок в CRUD — пикер поверх `listImages` (см. [media.md](media.md) §5);
  превью поля — через `ImagePreviewPicker`, bulk-удаление — через `useBulkDelete`
  + `BulkDeleteButton` (§4), не переизобретать.
