# Avocado Kiss — Blog (Article) editor — контракт для web.admin

> Last updated: 2026-08-04 | Site: **avocado.kiss** (схема `avocado_kiss`, бакет
> `avocado-kiss-photos`). Раздел в web.admin **ещё не построен** (Phase B) — этот
> файл его контракт: по нему агент собирает форму «Blog».
>
> ⚠️ Это НЕ [blog.md](blog.md) — тот про блог **cozycorner** (две таблицы секций
> `post_text_sections`/`post_product_sections` + `post_type` blog/seo). У Avocado
> модель другая: **одна** таблица секций `post_sections` + **3 шаблона поста** +
> блоки `qa`/`list_item`. Не смешивать. Общие паттерны слоя данных, картинок и
> гардов — [api.md](api.md); модель БД — [../database/schema.md](../database/schema.md);
> как это рендерит сайт — [../sites/avocado-kiss.md](../sites/avocado-kiss.md) §3.1.

## 0. Ментальная модель: пост = Тип + Теги + Секции

Три **независимые** оси. Меняешь одну — остальные не трогаются.

| Ось | Таблица | Что задаёт | На что влияет на сайте |
|---|---|---|---|
| **Тип поста** (`template`) | `posts.template` — 1 из 3 (`essay`/`interview`/`roundup`) | **только hero + где стоит byline** | НЕ трогает тело поста |
| **Теги** | `tags` + `post_tags` (m2m) | сквозные метки | фильтр архива `/blog?tag=`, эйброу hero (**= только теги**), авто-подбор «Read also» |
| **Секции** (блоки) | `post_sections` (упорядоченный список) | всё тело поста | свободный конструктор: любой из 6 типов, в любом порядке/количестве |

**Ключевые разграничения (частый источник путаницы):**
- **Тип блока ≠ тип поста.** Блоки `qa`/`list_item` названы «интервью/роллап-овыми»,
  но к шаблону НЕ привязаны — редактор может добавить любой блок в любой пост
  (решение зафиксировано: без ограничений).
- **Эйброу hero = теги, а не имя типа.** Имя шаблона в эйброу НЕ подставляется
  (в макетах essay→«ESSAY», но roundup→«BAKING»). Хочешь показать «Interview» —
  повесь такой тег. Отдельного поля «eyebrow» нет.
- Тип поста влияет ТОЛЬКО на hero-раскладку и на видимость двух полей формы
  (`subtitle`, `hero_caption`) — см. §2.

## 1. Слой данных

- Читает сайт — `avocado.kiss/lib/blog.ts` (`fetchPostBySlug`/`fetchPostSections`/…).
  Админка пишет напрямую в таблицы схемы `avocado_kiss` через `getDb('avocado-kiss')`.
- **Slug** генерирует БД-триггер `posts_set_slug` — при create НЕ отправлять;
  при edit read-only. `authors.slug` — тоже триггер.
- `updated_at` (на `posts` и `post_sections`) — триггеры, вручную не задавать.
- Пустые optional-поля → `null` (`orNull`), не пустые строки (паттерн products).
- SEO «живые»: `seo_title`→`title`, `seo_description`→`excerpt`; значение, равное
  фолбэку или пустое, пишется как `null` (api.md §6).
- **Картинки** (`hero_image_path`, `avatar_path`, section `image_path`): контракт
  как у products — в БД плоский ключ бакета или внешний URL, в UI всегда публичная
  ссылка (`resolveImageUrl` ↔ `toStoragePath('avocado-kiss', value)`), пикер —
  общий `ImagePickerDialog` (api.md §5).
- **Diff-сохранение секций** — модель ПРОЩЕ cozycorner: секции лежат в ОДНОЙ
  таблице `post_sections`, поэтому нет сквозного слияния двух таблиц. Порядок —
  `position` = индекс секции в массиве формы. На Save: insert новых (без `id`) →
  update существующих (пишем контент И `position` всегда) → delete пропавших
  в конце. После Save — пересинхронизация формы свежими данными (иначе повторный
  Save продублирует новые секции). Риски и почему без транзакции — как в blog.md §1.
- **id секции в форме называть `sectionId`**, не `id` (useFieldArray резервирует `id`).

## 2. Форма поста

### 2.1 Общие поля (для всех типов)

| Поле формы | Колонка `posts` | Тип/контрол | Обяз. | Примечание |
|---|---|---|---|---|
| Title | `title` | text | да | — |
| Slug | `slug` | read-only (edit) / скрыт (create) | — | триггер БД |
| Type | `template` | Select: essay / interview / roundup | да | default `essay`; управляет hero и видимостью §2.2 |
| Excerpt | `excerpt` | textarea | нет | анонс; фолбэк карточки/SEO |
| Hero image | `hero_image_path` | ImagePicker (контракт картинок) | нет | нет → брендовый плейсхолдер |
| Author | `author_id` | Select из `authors` (§4) | нет | «by {name}» |
| Tags | `post_tags` (m2m, §4) | мультиселект из `tags` | нет | эйброу + фильтр + Read also |
| Read time | `read_minutes` | number (мин) | нет | «N min read»; null → скрыто |
| Published | `is_published` | Switch | — | фильтр ленты |
| Published at | `published_at` | datetime-local | — | сортировка ленты (обрезка до минут) |
| SEO title | `seo_title` | text (живой фолбэк → title) | нет | — |
| SEO description | `seo_description` | textarea (живой фолбэк → excerpt) | нет | — |

### 2.2 Поля, зависящие от Type (показывать по `template`)

| Поле | Колонка | Показывать для | Назначение |
|---|---|---|---|
| Subtitle (dek) | `posts.subtitle` | interview, roundup | подзаголовок в hero (у essay hero его нет) |
| Hero caption | `posts.hero_caption` | roundup | подпись под широкой hero-фигурой |

Смена Type не переносит контент секций — только меняет hero. Значения `subtitle`/
`hero_caption` можно не очищать при переключении (сайт их просто не покажет).

## 3. Конструктор секций (`post_sections`)

`useFieldArray({ name: 'sections' })`. Карточка секции: селектор типа (или тип
задаётся при добавлении), контролы `is_published` (глазик), ↑/↓ (`move`), корзина.
**Все 6 типов доступны в любом посте** (без привязки к шаблону). Поля по типу
(`post_sections.type` — дискриминант, остальные колонки nullable, заполняются по типу):

| Тип (`type`) | Поля формы → колонки | Валидация | Где на сайте |
|---|---|---|---|
| **text** | Body → `body` (rich/markdown, абзацы по пустой строке); Variant → `text_variant` (`lead`/`body`, default body) | `body` непустой | абзац; `lead` — крупнее/легче |
| **quote** | Quote → `quote`; Attribution → `quote_attribution` (опц.) | `quote` непустой | центр-цитата курсивом |
| **image** | Image → `image_path` (контракт картинок); Caption → `caption`; Credit → `credit` | image ИЛИ caption есть | `<figure>` + подпись |
| **recipe_card** | Recipe → `recipe_id` (пикер рецептов, §4); Eyebrow → `card_eyebrow` (опц., оверрайд «Recipe») | `recipe_id` задан | карточка-ссылка на рецепт |
| **qa** | Question → `question`; Answer → `answer` | оба непустые | вопрос (жирнее) + ответ |
| **list_item** | Rank → `rank` (number); Heading → `heading`; Body → `body`; Recipe → `recipe_id` (опц., пикер); Eyebrow → `card_eyebrow` (опц.) | `heading` непустой | нумер. пункт + вложенная карточка рецепта |

Примечания:
- `recipe_card` и `list_item` делят колонки `recipe_id`/`card_eyebrow` (одна карточка
  рецепта на секцию). Удалённый рецепт → на сайте карточка просто не рендерится
  (FK `on delete set null`); в форме показывать «Unknown recipe (deleted?)», id не
  чистить молча (паттерн products-модуля в blog.md §4).
- `is_published=false` у секции скрывает её на сайте, но валидацию контента НЕ
  отключает (нельзя «спрятать» пустой блок вместо заполнения).

## 4. FK-пикеры

- **Author** (`posts.author_id` → `authors`): Select. Таблица `authors` = `name`,
  `slug` (триггер), `avatar_path` (картинка), `bio`. Отдельного CRUD авторов пока
  нет — либо простой select существующих, либо мини-редактор (решить при постройке).
- **Recipe** (`post_sections.recipe_id` → `recipes`): пикер рецептов (поиск по title,
  превью `hero_image_path`) — аналог `ProductPickerDialog`, но по таблице `recipes`.
- **Tags** (`post_tags`): мультиселект из `tags` (m2m; строки `post_tags(post_id,
  tag_id, position)`; `position` = порядок в эйброу).

## 5. «Read also» — ручные пины (`post_related`), опционально

На сайте Read also **авто-подбирается** (посты по общему тегу/свежести, затем
рецепты) — редактор может НИЧЕГО не делать. Опционально можно **закрепить** до 3
карточек сверху авто-выдачи:

- Таблица `post_related(post_id, position, recipe_id, related_post_id)` —
  полиморфно: ровно одна ссылка (`recipe_id` XOR `related_post_id`, check
  `num_nonnulls=1`). `position` = порядок пинов.
- UI: список пинов с пикером «рецепт или пост» + ↑/↓/✕. Сохранение — тот же
  diff-паттern, что и секции. Не обязательный раздел формы (можно отложить).

## 5b. Баннер страницы архива `/blog` (таблица `pages`, НЕ пост)

Баннер вверху архива (белый бокс с текстом на фоне-картинке, компонент
`ShopHero`) — это **не** свойство поста, а настройка **страницы**. Живёт в строке
`pages` со `slug='blog'` (миграция 0010; аналогично `pages.shop` для `/shop`):
- Поля: `hero_eyebrow`, `hero_title`, `hero_description`, `hero_image_path`
  (контракт картинок; `null` → брендовый плейсхолдер). Пусто → фолбэк на
  код-константу на сайте.
- `pages` — **update-only** (строки `home`/`shop`/`blog` фиксированы, create/delete
  нет). Тот же ряд несёт SEO (`seo_title`/`seo_description`/`og_image_path`).
- В админке это отдельная секция **Pages/Settings**, не форма поста. Редактор
  `/blog`-баннера = `update` полей `hero_*` строки `pages.blog`.

## 6. Что НЕ делает админка (guardrails)

- Не меняет схему/типы/RLS (это миграции в репозитории avocado.kiss).
- Не пишет `slug`/`updated_at` (триггеры).
- Эйброу hero — не поле; управляется тегами.
- Тип блока не ограничивается типом поста (осознанное решение — макс. гибкость).
- Перенос поста между типами — просто смена `template` (контент секций не
  конвертируется; редактор сам приводит блоки к новому hero, если нужно).
