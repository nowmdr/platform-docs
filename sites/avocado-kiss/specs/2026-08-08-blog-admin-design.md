# Avocado Kiss — Blog (Article) editor в web.admin — дизайн

> Date: 2026-08-08 | Scope: web.admin (админка) — новый раздел Blog для сайта avocado-kiss
> Site: **avocado.kiss** (схема `avocado_kiss`, бакет `avocado-kiss-photos`)
> Related:
> [../../../admin-panel/blog-avocado-kiss.md](../../../admin-panel/blog-avocado-kiss.md) (контракт — источник модели формы/секций),
> [../../../admin-panel/blog.md](../../../admin-panel/blog.md) (блог **cozycorner** — образец UI-механики),
> [../../../admin-panel/products.md](../../../admin-panel/products.md) (база любого CRUD),
> [../../../admin-panel/api.md](../../../admin-panel/api.md) (слой данных / картинки / гарды),
> [../../avocado-kiss.md](../../avocado-kiss.md) §3.1 (как это рендерит сайт)

## 1. Цель и контекст

Построить в общей админке `web.admin` раздел **Blog** для сайта Avocado Kiss —
Phase B, раздел ещё не построен. Контракт формы уже написан
(`admin-panel/blog-avocado-kiss.md`); эта спека фиксирует **как** он ложится в
существующий код `web.admin` и какие решения приняты при постройке.

Модель Avocado заметно отличается от блога CozyCorner, поэтому раздел строится в
**отдельной feature-папке**, а не ветвлением общего кода:

| | CozyCorner blog | Avocado Kiss blog |
|---|---|---|
| Таблицы секций | **две** (`post_text_sections` + `post_product_sections`), позиции сквозные между таблицами | **одна** `post_sections`, дискриминант `type` |
| Типы блоков | 2 (text / products) | 6 (`text`/`quote`/`image`/`recipe_card`/`qa`/`list_item`) |
| Тип поста | нет; есть `post_type` blog/seo | **`posts.template`** essay/interview/roundup (меняет hero-раскладку) |
| Автор | нет | `posts.author_id` → `authors` |
| FK-пикер в секциях | products | recipes |

**UI-механику** (конструктор блоков, diff-сохранение, dirty-guard, image-пикер,
FK-пикер, SEO customize/reset, sticky-панель) портируем из `src/features/posts/`.
**Модель данных** берём из контракта. Блог CozyCorner при этом не трогаем.

### Пост = 3 независимые оси (ментальная модель)

| Ось | Таблица | Задаёт | Влияет на сайте |
|---|---|---|---|
| **Тип поста** | `posts.template` (1 из 3) | только hero-раскладку + видимость 2 полей формы | `ArticleHero.tsx`; тело НЕ трогает |
| **Теги** | `post_tags` (m2m) | сквозные метки | эйброу hero (= только теги, через ` · `), фильтр архива, авто «Read also» |
| **Секции** | `post_sections` (упорядоченный список) | всё тело поста | любой из 6 блоков, в любом порядке/количестве |

Разграничения (частый источник путаницы): тип блока **не** привязан к типу поста
(любой блок в любом посте); эйброу hero — **не** поле формы (управляется тегами,
имя шаблона не подставляется); тип поста влияет только на hero и видимость
`subtitle`/`hero_caption`.

## 2. Решения (подтверждены пользователем)

1. **Отдельная feature-папка** `src/features/articles/` + `src/lib/articles.ts`.
   Общий код `features/posts/` не ветвим, блог CozyCorner не трогаем.
2. **Добавление блока** — одна кнопка «Add block» + выбор типа (меню на 6 типов);
   тип фиксируется при добавлении (сменить тип у существующего блока нельзя — как
   в CozyCorner).
3. **Авторы** — только `Select` существующих `authors` (без CRUD авторов в v1;
   заводить/править авторов пока через Supabase-коннектор).
4. **Объём v1 включает все 3 опциональные части:** ручные пины «Read also»
   (`post_related`), баннер архива `/blog` (`pages.blog`), папки для списка постов.

## 3. Реестр и роутинг

- **`src/config/sites.ts`** — добавить `"blog"` в `avocado-kiss.sections`. Для
  баннера `/blog` (§7.2) добавить также `"pages"`. Итог:
  `["media","products","categories","brands","blog","pages"]`.
- **`src/App.tsx`** — маршруты `blog` / `blog/new` / `blog/:postId` сейчас жёстко
  рендерят `PostsPage`/`PostEditPage` (модель cozycorner). URL для Avocado
  оставляем тот же `/blog`, поэтому вводим **диспетчер по `site.schema`**:
  - `BlogListRoute` → `avocado_kiss` ? `<ArticlesPage/>` : `<PostsPage section={BLOG_SECTION}/>`
  - `BlogEditRoute` → `avocado_kiss` ? `<ArticleEditPage/>` : `<PostEditPage section={BLOG_SECTION}/>`
  Диспетчер читает `site` из `useOutletContext<SiteConfig>()` (как остальные
  страницы). Nav-пункт «Blog» один, метка общая; доступ гейтит `sections`.
  SEO Posts (cozy) не затрагиваются.
- **Pages** — маршруты `pages`/`pages/:pageId` уже существуют и site-agnostic
  (читают строки `pages` текущей схемы). Для Avocado достаточно добавить `"pages"`
  в `sections`; строка `pages.blog` уже есть (миграция 0010). Проверить, что
  `PagesPage`/`PageEditPage` корректно показывают строку `blog` для avocado_kiss
  (список строк из БД, без хардкода cozy-строк).

## 4. Слой данных — `src/lib/articles.ts`

Все запросы через `getDb(site).schema('avocado_kiss')`. Явные списки колонок
(не `select('*')`).

- **Типы** (хендвритен, как в `posts.ts`):
  - `ArticleTemplate = 'essay' | 'interview' | 'roundup'`.
  - `Article` (строка `posts` + `author`), `ArticleListItem` (лёгкий, + `folder_id`).
  - `ArticleSection` — дискриминированный union по `kind` (6 вариантов). **В форме
    поле id секции называется `sectionId`** (не `id` — `useFieldArray` резервирует
    `id`). Маппинг колонок БД → поля формы такой же, как на сайте (`lib/blog.ts`):
    `text_variant`→`variant`, `quote_attribution`→`attribution`, `image_path`→
    `imagePath`, `card_eyebrow`→`eyebrow`. (Дискриминант в БД — `type`, в форме —
    `kind`; конверсия при load/save.)
  - `RelatedPin` — union `recipe` | `post` (полиморфный `post_related`).
  - `ArticleInput` / `ArticleWithRelations` (пост + секции + теги + пины).
- **Функции:**
  - `listArticles(site, { search?, status?, folderId? })` — лёгкий select
    (`id,title,template,is_published,published_at,folder_id`), сортировка
    `published_at desc` + tiebreaker `title asc` (как products/posts).
  - `getArticle(site, id)` — пост + `author` + `post_sections (order by position)`
    + `post_tags (order by position)` + `post_related (order by position)`, плюс
    батч-hydration рецептов для `recipe_card`/`list_item` пикеров (заголовки/превью
    в форме) — одним `.in('id', recipeIds)` (без N+1).
  - `createArticle` — insert `posts` (slug не шлём — триггер `posts_set_slug`;
    `is_published` задаём **явно**, т.к. в БД default `true`), затем секции/теги/
    пины с `position` = индекс; при падении дочерних — откат поста `delete`
    (паттерн products/posts).
  - `updateArticle` — update поста + **diff-синк** секций/тегов/пинов (§4.1).
    Возвращает свежий `ArticleWithRelations` (для пересинхронизации формы).
  - `deleteArticle` — delete только строки `posts`; секции/теги/пины уберёт FK
    `ON DELETE CASCADE`.
- Конвенции: пустые optional → `null` (`orNull`); SEO живой фолбэк
  (`seo_title`→title, `seo_description`→excerpt; значение = фолбэку или пустое →
  `null`); `updated_at`/`slug` — триггеры, не пишем; картинки —
  `resolveImageUrl`/`toStoragePath`.

### 4.1 Diff-сохранение (проще CozyCorner — одна таблица секций)

Для `post_sections` (и тем же паттерном для `post_tags`, `post_related`):
1. **insert** новых строк (без `sectionId`) с `position` = индекс.
2. **update** существующих — пишем контент **И** `position` всегда (оба могли
   измениться).
3. **delete** пропавших `id` строго **в конце** (после insert/update).
4. **Пересинхронизация формы после Save — критично:** `onSuccess` делает
   `form.reset(toFormValues(fresh))` и обновляет снапшот `loadedRef`, иначе у
   только что вставленных секций нет `sectionId` и повторный Save их задублирует.

Diff считается против снапшота загрузки. Несколько запросов без транзакции —
осознанно (один админ-пользователь), как в blog.md §1; при падении — тост, форма
остаётся dirty. Возможный hardening на потом — RPC-атомарность.

## 5. Форма поста — `ArticleEditPage.tsx` (порт `PostEditPage`)

Один компонент `ArticleForm` на create (`blog/new`) и edit (`blog/:postId`);
монтируется после загрузки данных. RHF + Zod v4, Select/Switch через Controller.
Раскладка v2 как у cozy: sticky-панель сверху (Back, Published `Switch`, Save,
Delete), Save вынесен из `<form>` (`id="article-form"` + `form=` на кнопке).
Dirty-guard — паттерн products (`useBlocker` + `beforeunload`, модалка
Stay/Discard/Save). Невалидный Save — тост «Fix validation errors before saving».

### 5.1 Общие поля (все типы)

| Поле формы | Колонка `posts` | Контрол | Обяз. | Примечание |
|---|---|---|---|---|
| Title | `title` | text | да | — |
| Slug | `slug` | read-only (edit) / скрыт (create) | — | триггер БД |
| Type | `template` | Select essay/interview/roundup | да | default `essay`; управляет hero + §5.2 |
| Excerpt | `excerpt` | textarea | нет | анонс; фолбэк карточки/SEO |
| Hero image | `hero_image_path` | `ImagePreviewPicker` + инпут + Gallery (`ImagePickerDialog`) | нет | нет → плейсхолдер |
| Author | `author_id` | Select из `authors` | нет | «by {name}» |
| Tags | `post_tags` (m2m) | мультиселект из `tags` | нет | порядок = порядок эйброу |
| Read time | `read_minutes` | number | нет | «N min read»; null → скрыто |
| Published | `is_published` | Switch (в sticky) | — | **задавать явно** (БД default true) |
| Published at | `published_at` | datetime-local | — | сортировка ленты (обрезка до минут) |
| SEO title | `seo_title` | text (живой фолбэк → title) | нет | Customize / Use defaults |
| SEO description | `seo_description` | textarea (живой фолбэк → excerpt) | нет | — |

**Tags** — мультиселект с сохранением порядка (порядок = `post_tags.position` =
порядок в эйброу). Переиспользовать идиому chips/combobox из
`features/products/CategoryChipsField.tsx` + `features/taxonomy/TaxonomyCombobox.tsx`,
но с явным порядком элементов (↑/↓ или порядок добавления).

### 5.2 Поля, зависящие от Type (`watch('template')`)

| Поле | Колонка | Показывать для | Назначение |
|---|---|---|---|
| Subtitle (dek) | `subtitle` | interview, roundup | подзаголовок hero |
| Hero caption | `hero_caption` | roundup | подпись под широкой hero-фигурой |

Смена Type только меняет hero; контент секций не конвертируется. Скрытые значения
`subtitle`/`hero_caption` при переключении можно не очищать (сайт не покажет).

## 6. Конструктор секций — `SectionsEditor.tsx` (порт `PostModulesEditor`)

`useFieldArray({ name: 'sections' })`; ключ рендера карточки — `field.id`
(стабильный ключ useFieldArray, чтобы состояние `RichTextEditor` переезжало при
`move`, а не пересоздавалось). Индекс массива = `position`.

Карточка: бейдж типа (неизменяемый), глазик `is_published` (скрытый блок —
`border-dashed opacity-60`, но валидацию проходит как обычно), ↑/↓ (`move`,
disabled на краях), корзина (`remove`). **Add block** — одна кнопка → меню на 6
типов, каждый `append(...)` инициализирует объект своего kind.

| `kind` (`type`) | Поля формы → колонки | Валидация | Компонент/контрол |
|---|---|---|---|
| **text** | Body → `body` (RichTextEditor markdown); Variant → `text_variant` (lead/body, default body) | `body` непустой | `RichTextEditor` + сегмент-тоггл variant |
| **quote** | Quote → `quote`; Attribution → `quote_attribution` (опц.) | `quote` непустой | textarea + input |
| **image** | Image → `image_path` (пикер); Caption → `caption`; Credit → `credit` | image **или** caption | `ImagePreviewPicker`+`ImagePickerDialog` |
| **recipe_card** | Recipe → `recipe_id` (RecipePicker); Eyebrow → `card_eyebrow` (опц., оверрайд «Recipe») | `recipe_id` задан | `RecipePickerDialog` (single) |
| **qa** | Question → `question`; Answer → `answer` | оба непустые | input + textarea |
| **list_item** | Rank → `rank` (number); Heading → `heading`; Body → `body`; Recipe → `recipe_id` (опц.); Eyebrow → `card_eyebrow` (опц.) | `heading` непустой | number + input + RichText + RecipePicker |

Примечания: `recipe_card` и `list_item` делят колонки `recipe_id`/`card_eyebrow`
(одна карточка рецепта на секцию). Удалённый рецепт → «Unknown recipe (deleted?)»,
id не чистим молча. `is_published=false` не отключает валидацию контента.

## 7. Опциональные части

### 7.1 «Read also» ручные пины — `RelatedEditor.tsx` (`post_related`)

Секция формы: список пинов, пикер «рецепт **или** пост» (радио тип + пикер), ↑/↓/✕.
Строка полиморфна: ровно одна ссылка `recipe_id` **XOR** `related_post_id`
(check `num_nonnulls=1`); `position` = порядок. Сохранение — тот же diff-паттерн
(§4.1). Пусто → сайт авто-подбирает Read also (посты по общему тегу → рецепты).

### 7.2 Баннер архива `/blog` — раздел Pages (`pages.blog`)

**Не** свойство поста — настройка страницы. Редактируется в существующем разделе
**Pages** (строка `slug='blog'`, миграция 0010): `hero_eyebrow`, `hero_title`,
`hero_description`, `hero_image_path` (контракт картинок; null → плейсхолдер) + SEO
(`seo_title`/`seo_description`/`og_image_path`). `pages` — **update-only**. Реализация:
включить `"pages"` в `avocado-kiss.sections` (§3); переиспользовать
`PageEditPage`/`HeroSectionsEditor` как есть, убедившись, что они читают строки
`pages` текущей схемы (без хардкода cozy).

### 7.3 Папки списка постов

`ArticlesPage` = порт `PostsPage`: поиск по title (`useDeferredValue`) + Select
фильтра статуса (All/Published/Draft, AND) + `FoldersPanel` слева + bulk-бар
(multi-select, Move to…, Delete через `useBulkDelete`, Clear). `folderSection:
'posts'` (секция `posts` в `admin_folders`, общий код `src/features/folders/`).
`folder_id` только в списочном типе — форма его не трогает, новые посты → Unsorted.

> ⚠️ **ПРЕДУСЛОВИЕ — миграция в репо `avocado.kiss` (админка схему не меняет).**
> Проверено по `database/schema.md` (2026-08-08): у `avocado_kiss.posts` **нет**
> `folder_id`, а `admin_folders.section` CHECK = `('media','recipes','products')`
> — **без `'posts'`** (миграция 0022 «products/posts.folder_id» — cozy-only; у
> avocado folder_id получили только `media`/`recipes`/`products`). Поэтому папки
> постов **невозможно** построить только в web.admin. Нужна аддитивная миграция в
> `avocado.kiss/supabase/migrations/` (по образцу cozy 0022 + avocado 0013):
> добавить `posts.folder_id uuid null references admin_folders(id) on delete set
> null` + индекс, расширить `admin_folders.section` CHECK до
> `('media','recipes','products','posts')`. **Решение (подтверждено
> пользователем 2026-08-08): миграция делается ПЕРВЫМ шагом (§11.0), затем папки
> строятся в админке** — полный паритет с CozyCorner. `ArticlesPage` сразу с
> `FoldersPanel`.

## 8. FK-пикеры

- **Author** (`posts.author_id` → `authors`): `Select`. Таблица `authors` =
  `name, slug (триггер), avatar_path, bio`. Только выбор существующих (§2.3).
- **Recipe** (`post_sections.recipe_id`, `post_related.recipe_id` → `recipes`):
  `RecipePickerDialog` — порт `ProductPickerDialog`, но **single-select** по
  таблице `recipes` (поиск по title, превью `hero_image_path`). Один общий диалог
  на все секции формы (`pickerFor: index | null`).
- **Post** (`post_related.related_post_id` → `posts`): пикер постов (в
  RelatedEditor; поиск по title, single).
- **Tags** (`post_tags`): мультиселект из `tags` с порядком.

## 9. Гарды (что раздел НЕ делает)

- Не меняет схему/типы/RLS/триггеры — это миграции в репозитории avocado.kiss.
- Не пишет `slug`/`updated_at` (триггеры).
- Эйброу hero — не поле (управляется тегами).
- Тип блока не ограничен типом поста (осознанное решение — макс. гибкость).
- Без service-role; все записи под админ-сессией (RLS + `is_admin()`, общий
  whitelist проекта).
- Новые `*image_path*`-поля схемы уже покрыты рендером сайта; в админке новых
  колонок картинок не заводим — `USAGE_SOURCES` в `src/lib/media.ts` при
  необходимости дополнить (hero/section image_path avocado, если ещё не учтены).

## 10. Тестирование и приёмка

- **Unit/компонентные (Vitest + RTL) — обязательно:** конверсия load/save
  (`toFormValues`/`toInput`/diff-синк — insert/update/delete, пересинхронизация
  после Save, отсутствие дублей при повторном Save), Zod-валидация каждого из 6
  типов блоков, условная видимость `subtitle`/`hero_caption` по `template`,
  сохранение порядка тегов/пинов, «Unknown recipe» для удалённого FK.
- **e2e (Playwright) — по протоколу:** предложить сценарий «создать пост каждого
  из 3 типов + добавить/переупорядочить блоки + опубликовать» (порт 3100, live
  Supabase — реестр в корневом AGENTS.md).
- Прод-проверки: `npm run build` + `npm run lint` (oxlint; 2 известных
  shadcn-варнинга ок).
- Тесты пишутся **по спеке**; расхождение код фичи ≠ спека — не чинить молча,
  вынести багом.

## 11. Порядок реализации (для плана)

0. **(ПРЕДУСЛОВИЕ, вне web.admin — репо `avocado.kiss`)** аддитивная миграция:
   `posts.folder_id uuid null references admin_folders(id) on delete set null` +
   индекс, расширить `admin_folders.section` CHECK до
   `('media','recipes','products','posts')` (§7.3, образец cozy 0022 + avocado
   0013). Обязательный первый шаг — папки (§7.3, п.6) строятся после неё.
1. `src/lib/articles.ts` (типы + функции + diff-синк) — фундамент, покрыть unit.
2. `RecipePickerDialog` (порт products-пикера, single).
3. `SectionsEditor` (6 типов, Add block + выбор типа).
4. `ArticleEditPage`/`ArticleForm` (общие поля + Type-зависимые + SEO + секции).
5. Диспетчер роутинга по `site.schema` в `App.tsx` + `"blog"` в `sections`.
6. `ArticlesPage` (список + папки + bulk).
7. `RelatedEditor` («Read also» пины).
8. Pages для avocado (баннер `/blog`): `"pages"` в `sections` + проверка
   `PageEditPage` на avocado_kiss.
9. Тесты + build/lint; предложить e2e.

## 12. Открытые вопросы / на потом

- CRUD авторов (пока select-only) — отдельный заход, если понадобится.
- RPC-атомарное сохранение поста (пост + все дочерние одной транзакцией) —
  hardening взамен diff без транзакции.
- Preview-ссылка на черновик (как у cozy `preview_token`) — **не строим**:
  проверено по schema.md — у `avocado_kiss.posts` `preview_token` нет (0029 —
  cozy-only миграция). Появится только с отдельной миграцией в репо avocado.kiss.
- Обновить `admin-panel/status.md` и `admin-panel/blog-avocado-kiss.md` («раздел
  построен») по завершении.
