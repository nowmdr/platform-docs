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
  category/[slug]/page.tsx # архив категории (SSG+ISR): только сетка RecipeCard (без заголовка/эйброу)
  shop/page.tsx            # хаб Curated Shop (SSG+ISR): ShopHero + сетка категорий + Editors' picks
  shop/[category]/page.tsx # категория магазина (SSG+ISR): ShopHero + ShopCatalog (ShopFilters + ProductGrid)
  product/[slug]/page.tsx  # товар (SSG+ISR): ProductDetail + «Pairs well with» + «Related reading»
  not-found.tsx            # глобальная 404
  sitemap.ts               # sitemap.xml из БД (ISR 60s): главная + категории + опубл. рецепты
  robots.ts                # robots.txt: allow all + sitemap
components/            # один компонент = файл + CSS Module; SVG-иконки — в icons/
  Header.tsx / CategoryNav (внутри Header) / MobileMenu.tsx  # шапка + навигация категорий
  HeroCarousel.tsx      # клиентский hero-слайдер: GSAP-сдвиг трека (translateX), автоплей 10с, точки+стрелки
  RecipeCard.tsx        # унифицированная карточка: image-box с фикс-пропорцией (medium 4:3)
                        #   + MediaPlaceholder-фолбэк без фото + hover→accent заголовок;
                        #   варианты large/medium/list/wide. medium (сетка категории) с пропом
                        #   tags показывает теги вместо (дублирующей) категории
  TagLabels.tsx         # общий рендер меток-тегов «TAG | TAG» (Editor's Picks и карточки категории)
  EditorsPicks.tsx      # секция «Editor's Picks»: нумерованный список + sticky-карточка (метки — TagLabels)
  # --- Curated Shop (магазин) ---
  ProductCard.tsx       # карточка товара (brand/name/price/Shop) — общая: хаб, категория, «Pairs well with»
  ShopCategoryCard.tsx / ShopCategoryGrid.tsx  # «Shop by category»: image + name + «N items»
  ShopHero.tsx          # hero /shop и /shop/[category] (eyebrow + h1 + p + фон)
  EditorsPicksProducts.tsx  # «Editors' picks this month» (4 ProductCard) — НЕ путать с EditorsPicks (рецепты)
  ProductGrid.tsx       # клиентская сетка 3×3 + Load more (браузерный fetchProductsPage, дедуп по id); перезагружается при смене filters
  ShopFilters.tsx       # контролируемая панель фильтров (Brand/Price/Sort + Clear) — значения/onChange от ShopCatalog
  ShopCatalog.tsx       # композиция: держит выбор селектов, транслирует в ProductQuery для ProductGrid (паттерн cozycorner)
  ProductDetail.tsx     # товар: image + brand + title + description + price + «Buy from …» + бэклинк
  RelatedProducts.tsx / RelatedReading.tsx  # «Pairs well with» (товары) / «Related reading» (рецепты)
  NewsletterBlock.tsx / NewsletterForm.tsx  # блок рассылки; форма валидирует e-mail на фронте (success/error), но в базу пока не пишет (задел на server action)
  Footer.tsx            # подвал (текст из footer_settings)
  Reveal.tsx            # GSAP reveal-обёртка (prefers-reduced-motion учтён)
  icons/                # ChevronLeft/Right, ArrowLeft (бэклинк товара), Clock, Menu, Search, Users + соц-иконки XIcon/PinterestIcon/InstagramIcon
lib/
  supabase/client.ts / server.ts  # браузерный/серверный клиенты (db.schema='avocado_kiss'), тип DbClient
  content.ts            # fetchCategories (все), fetchNavCategories (только show_in_nav — для шапки),
                        # fetchCategoryBySlug, fetchHomeSlots (сгруппировано по слоту),
                        # fetchRecipeBySlug, fetchRecipesByCategory (embed'ит recipe_tags → recipe.tags),
                        # fetchEditorsPicks, fetchRecipeSlugs, fetchPageSeo, fetchFooterSettings
  images.ts             # resolveRecipeImage() (контракт путей картинок рецептов)
  shop.ts               # загрузчики магазина: fetchShopCategories/…/fetchProductsPage/fetchShopFilterOptions/fetchEditorsPicks/
                        #   fetchProductPairings/fetchRelatedReading + resolveProductImage/formatPrice/productPath;
                        #   типы ProductQuery (categorySlug/brand/priceMin/priceMax/sort) + ProductSort
                        #   (НЕ server-only — ProductGrid зовёт fetchProductsPage из браузера)
  types.ts              # Recipe/Category/Tag/Post/EditorPick/HomeSlot/PageSeo/FooterSettings + HOME_SLOTS;
                        #   Product/ShopCategory/ReadingItem + SHOP_PAGE_SIZE (магазин)
supabase/migrations/    # единственное место изменения схемы БД (workflow — schema.md §7, история — §10)
mockups/                # исходные SingleFile-макеты Lovable: v1 (home, recipe) + version-2/ (home v2, shop) — referencia для вёрстки
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
- `sitemap.ts` включает главную, `/shop`, все категории рецептов и магазина
  (`/category/[slug]`, `/shop/[slug]`), все опубликованные рецепты
  (`/recipes/[slug]`) и все товары (`/product/[slug]`). Рецепты грузятся
  `fetchRecipeSlugs` (анон видит только опубликованные — RLS `is_published`);
  категории и товары — публичные целиком. `robots.ts` — allow all + ссылка на
  sitemap.
- ⚠️ **Правило: любой новый или изменённый публичный роут обновляет
  `app/sitemap.ts` в том же изменении** (и `generateStaticParams` страницы, если
  роут динамический). Sitemap строится из БД-загрузчиков (`lib/content.ts`,
  `lib/shop.ts`) — добавил тип контента с публичной страницей → добавь его слаги
  в sitemap. Не полагайся на память: сверь список папок-роутов в `app/` с
  записями `sitemap.ts`.

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
2. **post** (пост блога) — таблица `posts` + динамические секции тела
   `post_sections` + авторы `authors`; архив `/blog` (баннер+бокс `ShopHero` из
   строки `pages.blog` — hero_eyebrow/title/description/image, мигр. 0010, фолбэк
   на код-константу; фильтр по тегам + Load more) и single-page `/blog/[slug]`;
   в шапке ведёт пункт
   **Article**. Данные — `lib/blog.ts` (`fetchPosts`/`fetchPostBySlug`/
   `fetchPostSlugs`/`fetchPostSections`/`fetchRelatedReading`/`fetchBlogTags`).
   Три шаблона (`posts.template`) реализованы: **essay** (полноэкранный hero +
   отдельный byline), **interview** (сплит-hero, блок `qa`), **roundup**
   (центрированный hero + широкая фигура с подписью `posts.hero_caption`, блок
   `list_item` с номером + вложенной карточкой рецепта). byline/share у essay —
   отдельной полосой (`ArticleByline`), у interview/roundup — внутри hero
   (`ArticleMeta`). Hero-вариант выбирается `ArticleHero` по `template`.
   **Блоки тела** (`post_sections`, единая таблица, дискриминант `type`,
   порядок `position`): `text` (вариант lead/body), `quote`, `image`,
   `recipe_card` (FK → recipes), `qa`, `list_item` — переиспользуемые, редактор
   добавляет в любом порядке (компонент-диспетчер `PostSections`). Hero-эйброу —
   теги поста через « · ». **Read also** (`fetchRelatedReading`) — гибрид: ручные
   пины `post_related` (полиморфно recipe/post) сверху, затем авто-добор постами
   (общий тег → свежесть), затем рецептами до 3 карточек.
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

- **`HeroCarousel`** (`components/HeroCarousel.tsx`, клиентский): слайды выложены
  в горизонтальный flex-**трек** (`.track`) внутри `overflow:hidden`-вьюпорта;
  переключение — GSAP-сдвигом трека (`xPercent: -100*index`, `power2.inOut`,
  600ms) — виден «проезд», не кроссфейд. **Бесшовный вперёд-луп:** за последним
  слайдом рендерится клон первого; долистав до клона, `onComplete` мгновенно
  снапит трек на слайд 0 (`gsap.set`). Автоплей каждые **10с** (`AUTOPLAY_MS`,
  задел под настройку в админке) с паузой при hover (`hovered` ref) и `animating`
  ref-гардом против наложения твинов; точки/стрелки — вручную. `prefers-reduced-
  motion` → мгновенный сдвиг (`duration: 0`). Не текущие слайды (и клон) помечены
  `aria-hidden` + `tabIndex=-1` — вне tab-order/скринридера. Ширина: слайдер
  центрируется на `--hero-slider-max` (**1400px** — чуть шире контентного
  `--shell-max` 1320px, но не во всю ширину), с малыми боковыми отступами
  (`padding-inline: clamp(0.75rem,1.5vw,1.5rem)`). Высота (≥768): не aspect-ratio,
  а `height: calc(100svh - 11.5rem)` (вычтены sticky-шапка bar+nav 8rem, верхний
  отступ 2rem, зазор снизу), `min/max-height` — чтобы слайдер целиком помещался
  во вьюпорт при первой загрузке.
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

- **Header**: sticky, **сплошной непрозрачный фон** (`--background`, без blur —
  чтобы при скролле контент не просвечивал); слева — **рабочая** строка поиска
  (десктоп, ≥768px), логотип по центру; справа — соц-иконки (X/Pinterest/Instagram,
  line-стиль под набор, ссылки на **домашние страницы сервисов**, видны ≥768px) и
  **иконка-триггер поиска** (мобайл, <768px, открывает полноэкранный оверлей — как
  в cozycorner); под шапкой — `CategoryNav` из **`fetchNavCategories()`** (только
  `show_in_nav = true`, сортировка `position, name`), а следом — статические пункты
  **Articles** (архив блога, в работе) и **Curated Shop** (`/shop`). Оба варианта
  шапки (`Header` и `MobileMenu`) содержат эти ссылки. Скрытая из нав категория
  (напр. `Seafood`, `show_in_nav=false`) остаётся доступна по `/category/<slug>`,
  в поиске и на главной — страницы/sitemap берут полный `fetchCategories()`.
  Глобальный поиск — §9.
- **MobileMenu**: простое раскрытие (не отдельный диалог) — см. «Вне объёма v1»
  в спеке дизайна.
- **Footer**: бренд + tagline, колонка Magazine (**About, Contact** — заглушки
  `href="#"`, страниц-адресатов пока нет), колонка Follow (**Twitter, Pinterest,
  Instagram** из `SOCIALS`), нижняя строка copyright/made_with из
  `footer_settings`.
- Глобальный поиск в шапке — реализован (§9).

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

- `/shop` — хаб: `ShopHero` из hero-полей строки `pages.shop`
  (`hero_eyebrow`/`hero_title`/`hero_description`/`hero_image_path`, мигр. 0010;
  фолбэк на код-константу `HUB_HERO`, если поле пусто) + сетка категорий
  (`fetchShopCategories`, сортировка `position, name`, «N items» из
  `item_count`) + «Editors' picks this month» (`fetchEditorsPicks`, 4 товара).
  SEO и hero-баннер — из одной строки `pages.shop` (`fetchPageSeo('shop')`).
- `/shop/[category]` — страница категории: `ShopHero` из полей `shop_categories`
  (`generateStaticParams` ← `fetchShopCategorySlugs`) + `ShopCatalog` = рабочая
  панель `ShopFilters` + `ProductGrid` (паттерн cozycorner `ShopCatalog`).
  Фильтры: **Brand** (реальные бренды категории через `fetchShopFilterOptions(client,
  categorySlug)`), **Price** (диапазоны Under $30 / $30–$60 / $60 and up), **Sort**
  (Newest / Price ↑ / Price ↓) + кнопка **Clear** (видна, когда что-то выбрано).
  `ShopCatalog` держит выбор в клиентском состоянии (**без URL-параметров** — как
  cozycorner) и транслирует его в `ProductQuery`; смена фильтров сбрасывает сетку и
  грузит её заново с первой страницы. Категория страницы — постоянный фильтр поверх
  выбора. `ProductGrid` рендерит первую страницу (9, сетка 3×3) на сервере и
  догружает следующие кнопкой **Load more** через браузерный
  `fetchProductsPage(supabase, {categorySlug, page, ...filters})`; дедуп по `id`;
  кнопка прячется, когда пришло < `SHOP_PAGE_SIZE`. Порядок «Newest first»
  детерминирован (`created_at desc, id desc`); сортировки по цене — `price` +
  вторичный ключ `id desc`.
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

## 9. Глобальный поиск

Поиск по всему сайту (порт из cozycorner). **Обычный ilike-подстрочный поиск
через PostgREST `.or()`** — RPC/FTS/`pg_trgm` для текущего размера каталога не
нужны, отдельных миграций/индексов нет. Данные — `lib/search.ts` (**не**
`server-only`: те же функции работают и в браузере — live-панель в шапке, — и на
сервере — страница `/search`). Функции принимают `client: DbClient` аргументом.

**Что ищется (5 типов, порядок выдачи — рецепты первыми):**
рецепты → категории рецептов → статьи блога → товары → категории магазина.

| Функция | Таблица | Поля (`.or()` ilike) | Фильтр | Сортировка |
|---|---|---|---|---|
| `searchRecipes` | `recipes` | title, excerpt, category, author_name | `is_published=true` | `published_at desc, id desc` |
| `searchCategories` | `categories` | name | — | `position, name` |
| `searchPosts` | `posts` | title, excerpt | `is_published=true` (у posts **нет** `post_type`) | `published_at desc, id desc` |
| `searchProducts` | `products` | title, description, brand | — (нет `is_published`; наличие = живой) | `created_at desc, id desc` |
| `searchShopCategories` | `shop_categories` | name | — | `position, name` |

- `sanitizeSearchQuery` — обрезает до 100 символов, заменяет ломающие `.or()`
  символы `,()` пробелами, экранирует спецсимволы LIKE `\ % _`; пустая строка на
  выходе = «поиска нет». `searchPath(q)` → `/search?q=…`. `SearchResult<T> =
  { items, total }` (`count:"exact"` — для счётчиков и «See all N»). Лимиты —
  константы `SEARCH_*_LIMIT` + `SEARCH_DROPDOWN_LIMIT=5` / `SEARCH_OVERLAY_LIMIT=10`.
- `searchProducts` **не** достраивает `category_name` (ни строка выпадашки, ни
  `ProductCard` его не используют) — лишний lookup к `shop_categories` не делается.

**UI — `components/SearchBox.tsx`** (`"use client"`): общий хук `useSiteSearch`
(query + debounce 300ms + стоп-контроль устаревших ответов `seqRef` + сброс при
навигации) и две поверхности, переключаемые по брейкпоинту **768px** (совпадает с
шапкой):

- **Десктоп (≥768px)** — `SearchBox` (default export): инлайн-инпут слева в шапке
  + live-панель `#header-search-results`. `Enter` → `/search`, `Escape` / клик вне
  закрывают панель.
- **Мобайл (<768px)** — `SearchMobile` (named export): иконка-триггер **справа** в
  шапке → полноэкранный оверлей (`role="dialog"`, блокировка прокрутки фона,
  автофокус). Крестик/`Escape` закрывают.
- Активна всегда только видимая поверхность (у скрытой query пустой → без
  запросов), поэтому дублирования запросов нет.
- **Cmd/Ctrl+K** (расширение сверх cozycorner): фокус инпута на десктопе / открытие
  оверлея на мобайле.
- Миниатюры строк: `resolveRecipeImage` (рецепты/статьи/категории магазина),
  `resolveProductImage` (товары); у категорий рецептов картинки нет → плитка-
  заглушка.

**Страница `/search`** (`app/search/page.tsx`, RSC): динамическая (`await
searchParams`), `robots: { index: false }` — **намеренно не в `sitemap.ts`**. Hero
(`ShopHero`) + сводка непустых групп + секции карточек в том же порядке
(рецепты первыми), переиспользуют `RecipeCard`/`PostCard`/`ProductCard`/
`ShopCategoryCard`; категории рецептов — лёгкие текст-карточки-ссылки.

## 10. Рейтинги рецептов (звёзды)

Оценка рецептов на странице `/recipes/[slug]`. Две поверхности:

- **Read-only бейдж** (`RatingBadge`) под заголовком рецепта — звёзды +
  отображаемое среднее и число оценок.
- **Интерактивный 5-звёздочный пикер** (`RatingSection`) в самом низу рецепта —
  посетитель ставит оценку.

Отображаемое среднее **`display_avg` берётся с полом `min_display_rating`
(default 4.1)** и может быть «затравлено» админом (`seed_count`/`seed_sum`) —
модель данных и GENERATED-колонки в [../database/schema.md](../database/schema.md)
§9 (`recipes` рейтинги, `recipe_ratings`, `rate_recipe`).

**Отображаемое число оценок — не `display_count` напрямую.** Пока реальных
оценок ≤100, показывается стабильное псевдослучайное число **1–500**,
детерминированно выведенное из `recipe.id` (`socialProofCount` в
`avocado.kiss/lib/rating.ts`) — одинаковое между рендерами/перезагрузками
(не мигает) и никогда не «0 Ratings». Как только реальных оценок станет **>100**
(`REAL_COUNT_THRESHOLD`) — показывается реальное `display_count`. Это чисто
витринное число: в БД оно не хранится, применяется и к бейджу, и к секции.

JSON-LD `aggregateRating` (schema.org Recipe) отдаётся только при
`display_count > 0` и содержит **честный** реальный `display_count`/`display_avg`
(не витринный 1–500) — иначе выдуманные счётчики отзывов в структурированных
данных нарушают правила Google.

Голоса пишутся через `POST /api/recipes/[slug]/rate` — **Route Handler, первая
мутация в репозитории**: он проверяет токен Cloudflare Turnstile и пишет
service-role ключом через RPC `rate_recipe` (server-only секреты
`SUPABASE_SECRET_KEY` + `TURNSTILE_SECRET_KEY`, никогда не `NEXT_PUBLIC_`).
Дедуп «один голос на браузер» — **только localStorage** (UX-подсказка, на сервере
не форсится). Это API-роут (не страница), нового публичного PAGE-роута нет —
`sitemap.ts` не меняется.

Спека: [avocado-kiss/specs/2026-08-06-recipe-ratings-design.md](avocado-kiss/specs/2026-08-06-recipe-ratings-design.md).

**Тесты:** `lib/search.test.ts` (unit: `sanitizeSearchQuery`, `searchPath`,
`searchRecipes`), `e2e/search.spec.ts` (Playwright, :3100, live Supabase: десктоп
выпадашка → `Enter` → `/search`, прямой переход, empty-state; мобайл триггер →
оверлей → ввод → закрытие).
