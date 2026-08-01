# Avocado Kiss — архитектура сайта

> Last updated: 2026-08-01 | Source project: avocado.kiss (AGENTS.md,
> sites/avocado-kiss/specs/2026-07-17-avocado-kiss-v1-design.md) — пути файлов
> относятся к репозиторию `avocado.kiss/`

Кулинарный журнал (рецепты, английский UI): курируемая через `home_slots`
главная, страницы рецептов, страницы категорий. **Next.js 16** (App Router,
RSC) + Supabase + Vercel. Шрифт Fira Sans — self-hosted woff2 (лицензия OFL).
Модель данных — [../database/schema.md](../database/schema.md) §9 (раздел
avocado_kiss).

> ⚠️ Это **не привычный Next.js** — версия с breaking changes. Перед написанием
> кода сверяйтесь с гайдами в `node_modules/next/dist/docs/` (см. AGENTS.md
> репозитория).

Фаза A (сайт + БД) реализована и жива; интеграция с общей админкой
`web.admin` — **фаза B**, отдельная задача, ещё не начата. Сейчас курирование
главной и правка контента возможны только напрямую через БД.

## 1. Структура репозитория

```
app/
  layout.tsx              # root layout: Header + Footer, метаданные (SEO/OG), lang="en"
  globals.css              # дизайн-токены (:root) + @font-face Fira Sans + сброс
  page.tsx                 # home: hero-карусель + курируемая сетка + Editor's Picks + newsletter
  recipes/[slug]/page.tsx  # страница рецепта (SSG+ISR): meta, image, Ingredients/Method
  category/[slug]/page.tsx # архив категории (SSG+ISR): заголовок + сетка RecipeCard
  not-found.tsx            # глобальная 404
  sitemap.ts               # sitemap.xml из БД (ISR 60s): главная + категории + опубл. рецепты
  robots.ts                # robots.txt: allow all + sitemap
components/            # один компонент = файл + CSS Module; SVG-иконки — в icons/
  Header.tsx / CategoryNav (внутри Header) / MobileMenu.tsx  # шапка + навигация категорий
  HeroCarousel.tsx      # клиентская hero-карусель, GSAP-кроссфейд, автоплей, точки+стрелки
  RecipeCard.tsx        # унифицированная карточка: image-box с фикс-пропорцией (medium 4:3)
                        #   + MediaPlaceholder-фолбэк без фото + hover→accent заголовок;
                        #   варианты large/medium/list/wide. medium (сетка категории) с пропом
                        #   tags показывает теги вместо (дублирующей) категории
  TagLabels.tsx         # общий рендер меток-тегов «TAG | TAG» (Editor's Picks и карточки категории)
  EditorsPicks.tsx      # секция «Editor's Picks»: нумерованный список + sticky-карточка (метки — TagLabels)
  NewsletterBlock.tsx / NewsletterForm.tsx  # декоративный блок рассылки (submit никуда не пишет)
  Footer.tsx            # подвал (текст из footer_settings)
  Reveal.tsx            # GSAP reveal-обёртка (prefers-reduced-motion учтён)
  icons/                # ChevronLeft/Right, Clock, Menu, Search, Users + соц-иконки XIcon/PinterestIcon/InstagramIcon
lib/
  supabase/client.ts / server.ts  # браузерный/серверный клиенты (db.schema='avocado_kiss'), тип DbClient
  content.ts            # fetchCategories, fetchCategoryBySlug, fetchHomeSlots (сгруппировано по слоту),
                        # fetchRecipeBySlug, fetchRecipesByCategory (embed'ит recipe_tags → recipe.tags),
                        # fetchEditorsPicks, fetchRecipeSlugs, fetchPageSeo, fetchFooterSettings
  images.ts             # resolveRecipeImage() (контракт путей картинок)
  types.ts              # Recipe (+optional tags)/Category/Tag/Post/EditorPick/HomeSlot/PageSeo/FooterSettings, HOME_SLOTS
supabase/migrations/    # единственное место изменения схемы БД (workflow — schema.md §7, история — §10)
mockups/                # исходные SingleFile-макеты Lovable (home, recipe) — referencia для вёрстки
```

Папки `docs/` в репозитории нет — вся документация в `platform-docs/` (этот файл).

## 2. Рендеринг и SEO

- Все контентные страницы пререндерятся статически с ISR
  (`export const revalidate = 60`) — правка данных в БД появится на сайте в
  пределах ~минуты без редеплоя (после запуска фазы B — из админки; сейчас —
  через прямые изменения БД).
- ⚠️ Supabase-клиент не передаёт в fetch своих опций кэша — запросы наследуют
  сегментные 60s. **Не добавлять `force-cache`** в клиент — заморозит данные.
- Метадата: дефолт — `app/layout.tsx` (`title.template`, OpenGraph); страница
  home читает `pages` (slug `home`) через `fetchPageSeo`, рецепт — свои
  `seo_title`/`seo_description` (фолбэк `title`/`excerpt`), категория — свои
  `seo_title`/`seo_description` (фолбэк — автоформула из `name`).
- `sitemap.ts` включает главную, все категории и все опубликованные рецепты
  (по `fetchRecipeSlugs`, без фильтра — таблица `recipes` отдаёт всё в анон,
  RLS ограничивает чтение полностью; см. schema.md §3 про `is_published`).
  `robots.ts` — allow all + ссылка на sitemap.

## 3. Курирование главной — модель `home_slots`

Главная не собирается кодом по фиксированной выборке — она **курируется**:
каждая строка `home_slots` указывает, какой рецепт показать в каком слоте
макета и в каком порядке внутри слота. Загрузчик `fetchHomeSlots()`
(`lib/content.ts`) делает один запрос с `recipes!inner` embed и двойным
фильтром `is_published = true` (и на слот, и на `recipe.is_published`) —
слот с неопубликованным рецептом просто не попадает в выдачу, а не рендерится
пустым. Результат группируется по имени слота в `Record<HomeSlotName, HomeSlot[]>`.

Слоты и роль по макету v2 (валидирует админка фазы B; сайт рендерит что есть,
не проверяет лимиты):

| Слот | Ёмкость | Где на странице (v2) |
|---|---|---|
| `hero` | 3 | Hero-карусель (слайды) |
| `grid_large` | 1 | Мозаика: feature-карточка c7 (3/2) + подзаголовок |
| `grid_medium` | 2 | Мозаика: карточки c5 (1/1) и c4 (4/5) |
| `grid_list` | 5 | Мозаика: карточки c4/c8/c3 (были текстовым списком — теперь плитки с фото) |
| `wide` | 2 | Мозаика: карточки c3 (1/1) и c6 (16/10) |
| `mosaic_banner` | 1 | Мозаика: широкий баннер c12 (21/6) с текстом поверх градиента |
| `pick` | 3 | Editor's Picks — нумерованный список 1–3 |
| `pick_feature` | 1 | Editor's Picks — карточка справа (sticky, с фото) |

Мозаика (`app/page.tsx`, компоненты `MosaicCard`/`MosaicBanner`) — фиксированный
шаблон на 12-колоночной сетке, питается **только рецептными** слотами
(`grid_*`, `wide`, `mosaic_banner`). `pick`/`pick_feature` отданы секции
Editor's Picks и могут ссылаться на рецепт **или** пост (см. ниже).

Оверрайды и фолбэки (поля `home_slots`, не `recipes`):
- `eyebrow` — оверрайд лейбла над карточкой/hero-заголовком; пусто →
  категория рецепта (или «Recipe of the week» для hero — фолбэк в `app/page.tsx`).
- `eyebrow_secondary` — **устар.** (до v2 — второй лейбл picks); метки Editor's
  Picks теперь берутся из тегов элемента, не из этого поля.
- `description` — оверрайд описания (feature-карточка мозаики, Editor's Picks);
  пусто → `excerpt` элемента.

### 3.1 Типы контента и теги (модель v2)

Три типа контента (post types):
1. **recipe** (рецепт) — таблица `recipes`, свой single-page `/recipes/[slug]`.
2. **post** (пост блога) — таблица `posts` (скелет; финальный дизайн/поля позже),
   single-page `/archive/[slug]` (в работе); в шапке ведёт пункт **Articles**.
3. **product** (товар) — по образцу cozycorner; **ещё не реализован**, заведём
   позже; пункт шапки **Curated Shop** (`/shop`).

**Категории vs теги** — разные сущности:
- **Категории** (`categories`, поле `recipes.category`) — раздел рецепта,
  показываются в карточках (SEASONAL). Рецепт может относиться к нескольким.
- **Теги** (`tags` + `recipe_tags`/`post_tags`) — сквозные метки контента,
  общие для всех типов. Два пласта: журнальные (LIFE, COMMUNITY) и дескрипторы
  блюда (Vegetarian, Quick, Comfort Food, …). Выводятся через «\|»
  (`VEGETARIAN | QUICK`) общим компонентом `TagLabels` — в Editor's Picks и в
  теле карточки на странице категории (там теги **заменяют** дублирующую
  категорию; данные — `fetchRecipesByCategory` embed'ит `recipe_tags`).

**Editor's Picks** (`fetchEditorsPicks`, компонент `EditorsPicks`): слоты
`pick`/`pick_feature` разрешаются в рецепт или пост; выводятся заголовок,
описание, теги, а у правой карточки — ещё и фото. Админка фазы B выбирает для
каждого слота элемент (рецепт/пост) и держит набор тегов элемента коротким.

## 4. Картинки

- Формат хранения — [schema.md §4](../database/schema.md): относительный
  плоский ключ бакета `avocado-kiss-photos` ИЛИ внешний URL. Разбор —
  `resolveRecipeImage()` (`lib/images.ts`) → `{ url, unoptimized }` (или
  `null`, если путь пуст).
- Свои картинки оптимизируются через `next/image` (хост `*.supabase.co`
  разрешён в `next.config.ts`, `pathname: '/storage/v1/object/public/**'`);
  внешние рендерятся с `unoptimized: true` — их хосты в `remotePatterns`
  добавлять не нужно (тот же принцип, что в cozycorner — не расширять
  allowlist, не тратить квоту оптимизации Vercel на чужие CDN).
- Поля картинок рецептов в v1 — `recipes.hero_image_path` (используется и как
  обложка карточки, и как hero-фото страницы рецепта); отдельного поля под
  картинку рецептной категории нет. У магазина (§8) свои поля картинок:
  `products.image_path` и `shop_categories.hero_image_path` (тот же контракт
  бакета `avocado-kiss-photos`), разбор — `resolveProductImage()` (`lib/shop.ts`,
  зеркало `resolveRecipeImage`, тот же бакет). Тестовый сид товаров —
  `image_path=null` → рендерится `MediaPlaceholder`.
- Битый/пустой путь → `resolveRecipeImage` возвращает `null`; карточки мозаики,
  Editor's Picks и `RecipeCard` (сетка категории) рендерят брендовый плейсхолдер
  `MediaPlaceholder` (надпись «Avocado Kiss» на бумажном градиенте) вместо
  пустого прямоугольника — поэтому ячейки сетки категории не «схлопываются».
- `RecipeCard` держит фикс-пропорцию image-box на `.imageWrap` (medium — 4:3,
  large — 5:4, wide/portrait — 4:5), чтобы фото и плейсхолдер занимали одну
  рамку и лента категории была ровной.

## 5. GSAP-анимации

- **`HeroCarousel`** (`components/HeroCarousel.tsx`, клиентский): слайды
  наложены друг на друга (`position: absolute`), активный проявляется
  GSAP-кроссфейдом (`autoAlpha`, `power2.inOut`, 900ms). Автоплей каждые 6с
  с паузой при hover (`hovered` ref, не state — не триггерит лишний рендер);
  точки + стрелки-кнопки переключают вручную. `prefers-reduced-motion` →
  мгновенное переключение (`duration: 0`). Ширина: слайдер **во всю ширину**
  вьюпорта с малыми отступами (`padding-inline: clamp(0.75rem,1.5vw,1.5rem)`),
  без `.shell`-обёртки — в отличие от контентных секций (1320px). Высота (≥768):
  не aspect-ratio, а `height: calc(100svh - 11.5rem)` (вычтены sticky-шапка
  bar+nav 8rem, верхний отступ 2rem, зазор снизу), `min/max-height` — чтобы
  слайдер целиком помещался во вьюпорт при первой загрузке.
- **`Reveal`** (`components/Reveal.tsx`, клиентский): обёртка вокруг секций/
  карточек — GSAP `from()` (`autoAlpha: 0, y: 24`) через `ScrollTrigger`
  (`start: "top 88%", once: true`), `@gsap/react`'s `useGSAP` со `scope`.
  Контент рендерится на сервере как обычно — прячет и анимирует его только
  клиентский GSAP после гидрации, поэтому SEO/no-JS не страдают.
  `prefers-reduced-motion` → анимация не запускается, контент остаётся
  видимым. Пропс `delay` — для стаггера соседних карточек (напр. лента
  категории). Мозаика главной в `Reveal` не оборачивается (плитки видны сразу).
- Hover-зум картинок карточек и смена цвета заголовков — чистый CSS
  (transition), без GSAP — как в cozycorner.

## 6. Навигация и структура страниц

- **Header**: sticky, blur-фон; строка поиска (декоративная — не работает,
  `aria-label` для a11y), логотип по центру; справа — соц-иконки
  (X/Pinterest/Instagram, line-стиль под набор, ссылки на **домашние страницы
  сервисов**, видны ≥768px), на мобильном справа — кнопка поиска; под шапкой —
  `CategoryNav` из `fetchCategories()` (сортировка `position, name`).
- **MobileMenu**: простое раскрытие (не отдельный диалог) — см. «Вне объёма v1»
  в спеке дизайна.
- **Footer**: колонки Magazine/Follow + copyright из `footer_settings`; ссылки
  Magazine (About, Contributors, Contact, Issue Archive) — без страниц-
  адресатов в v1 (декоративные).
- Поиск в шапке — нерабочий рендер в v1; функциональность — вне объёма.

## 7. Окружение и деплой

```
NEXT_PUBLIC_SUPABASE_URL=https://zwrkphynupdubevzwdzy.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=sb_publishable_…   # публичный ключ, не secret!
```

- Обе переменные нужны всегда (главная и рецепты тянут данные при сборке);
  шаблон — `.env.example`. Service-role сайту не нужен. Схема `avocado_kiss`
  уже добавлена в **Exposed schemas** (готово, не pending).
- **Деплой на Vercel ещё не настроен** — прод-URL pending. Когда появится
  проект на Vercel: связать с GitHub-репозиторием, задать те же переменные
  окружения (Production + Preview), `NEXT_PUBLIC_SITE_URL` — реальным доменом.
- Интеграция с общей админкой `web.admin` — **фаза B**, отдельная задача:
  запись в `SITES` (`web.admin/src/config/sites.ts`, slug `avocado-kiss`),
  секции Recipes/Categories/Home (пикер по слотам)/Pages/Footer/Media.
  Контракт БД под неё уже спроектирован (см. schema.md).

## 8. Curated Shop (раздел магазина)

Куратируемый магазин: сетка категорий + товары + карточки товара со связанной
курацией. Собственная таксономия (`shop_categories`), товары в `products`,
курация тремя join-таблицами (schema.md §9). Данные — `lib/shop.ts` (загрузчики
поверх Supabase; **не** `server-only` — `ProductGrid` вызывает
`fetchProductsPage` из браузера для Load more). Типы — `lib/types.ts`
(`Product`, `ShopCategory`, `ReadingItem`, `SHOP_PAGE_SIZE = 9`).

**Маршруты (все SSG + ISR `revalidate = 60`):**

- `/shop` — хаб: `ShopHero` (редакционная константа) + сетка категорий
  (`fetchShopCategories`, сортировка `position, name`, «N items» из
  `item_count`) + «Editors' picks this month» (`fetchEditorsPicks`, 4 товара).
  SEO — `fetchPageSeo('shop')` (строка `pages.shop`).
- `/shop/[category]` — страница категории: `ShopHero` из полей `shop_categories`
  (`generateStaticParams` ← `fetchShopCategorySlugs`) + статичная панель
  `ShopFilters` (декоративная, фильтрация в v1 не работает) + `ProductGrid`.
  `ProductGrid` рендерит первую страницу (9, сетка 3×3) на сервере и догружает
  следующие кнопкой **Load more** через браузерный `fetchProductsPage(supabase,
  {categorySlug, page})`; дедуп по `id`; кнопка прячется, когда пришло
  < `SHOP_PAGE_SIZE`. Порядок «Newest first» детерминирован (`created_at desc,
  id desc` — как cozycorner).
- `/product/[slug]` — страница товара (`generateStaticParams` ←
  `fetchProductSlugs`): `ProductDetail` (`fetchProductBySlug`, `category_name`
  для эйброу/бэклинка достраивается lookup'ом к `shop_categories`) +
  «Pairs well with» (`fetchProductPairings` → 3 товара, без само-ссылки) +
  «Related reading» (`fetchRelatedReading` → 3 **опубликованных** рецепта).

**Курация:** `fetchRelatedReading` embed'ит `recipes!inner` +
`eq('recipe.is_published', true)` — неопубликованные рецепты в блок не
протекают (паттерн `fetchHomeSlots`); проекция рецепта → `ReadingItem`
(`href=/recipes/{slug}`). Связь товар→категория — **текстовая**
(`products.category = shop_categories.slug`, без FK), поэтому `category_name`
достраивается отдельным запросом, а не PostgREST-embed'ом.

**Картинки товаров:** бакет `avocado-kiss-photos` (§4), `products.image_path` /
`shop_categories.hero_image_path`; разбор — `resolveProductImage()`. Тестовый
сид — `image_path=null` → `MediaPlaceholder`.

**Контент/посев:** через скилл `avocado-content-ops` (6 категорий, 36 товаров,
4 picks, по 3 пары/чтения на товар). Правка — тем же скиллом; фаза B добавит
секции магазина в `web.admin`.
