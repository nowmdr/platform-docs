# CozyCorner — архитектура сайта

> Last updated: 2026-07-23 | Source project: cozycorner (CLAUDE.md, docs/page-content.md, docs/product-images.md) — пути файлов относятся к репозиторию `cozycorner/`

Каталог уютных товаров для дома: главная с hero и сеткой товаров (бесконечный скролл),
каталог по категориям (`/shop` → `/shop/[category]`), страницы товаров, блог,
управляемый из БД контент (hero + SEO). **Next.js 16** (App Router, RSC) + Supabase +
Vercel. Сайт на английском, шрифт SF Pro (системный стек — лицензия Apple не позволяет
самохостинг). Модель данных — [../database/schema.md](../database/schema.md) §5.

> ⚠️ Это **не привычный Next.js** — версия с breaking changes. Перед написанием кода
> сверяйтесь с гайдами в `node_modules/next/dist/docs/` (см. AGENTS.md репозитория).

## 1. Структура репозитория

```
app/
  layout.tsx          # root layout: Header + Footer, метаданные (SEO/OG), lang="en"
  icon.svg            # favicon (знак-лого) — App Router подхватывает автоматически
  page.tsx            # home: серверная загрузка 1-й страницы товаров + Hero + ProductGrid
  globals.css         # дизайн-токены (:root) + сброс
  product/[slug]/page.tsx   # страница товара (SSG+ISR) + RelatedProducts + RelatedArticles
  shop/page.tsx             # хаб каталога: Hero (align=left) + CategoryGrid
  shop/[category]/page.tsx  # архив категории (SSG+ISR): hero/SEO из categories.* + ShopCatalog
  blog/page.tsx             # архив блога: Hero + сетка PostCard
  blog/[slug]/page.tsx      # пост (SSG+ISR): hero из полей поста + секции из БД + Related posts
  terms|privacy/page.tsx    # юридические страницы (LegalArticle): SEO из pages; тело — pages.body
  search/page.tsx           # результаты поиска (?q=, динамическая, noindex)
  featured/page.tsx         # заглушка (PagePlaceholder); скрыта из меню, жива по URL
  about/page.tsx            # заглушка (PagePlaceholder) — навигация из Footer
  not-found.tsx             # глобальная 404
  sitemap.ts                # sitemap.xml из БД (ISR 60s): статика + категории + товары
                            # + посты ОБОИХ типов (blog и seo); /search и /featured не включаются
  robots.ts                 # robots.txt: allow all + sitemap
components/           # один компонент = файл + CSS Module; каталог — docs/design-system.md
  Header.tsx          # шапка: навигация + лого + SearchBox + Instagram + бургер; sticky auto-hide
                      # instagramUrl приходит пропом из root layout (header_settings)
  SearchBox.tsx       # live-поиск в шапке (debounce 300ms, ≤5 строк, «See all N results»)
  Footer.tsx          # подвал (текст из footer_settings) + NewsletterForm
  Hero.tsx / HeroBackground.tsx  # hero из БД или собранный страницей; фон с GSAP-параллаксом
  Reveal.tsx          # GSAP reveal-обёртка (prefers-reduced-motion учтён)
  FiltersBar.tsx / ShopCatalog.tsx  # фильтры Brand/Price + связка с сеткой
  CategoryCard.tsx / CategoryGrid.tsx  # карточки категорий на /shop
  ProductCard.tsx / PostCard.tsx       # карточки товара/поста (stretched-link)
  PostTextSection.tsx / PostProductsSection.tsx  # секции поста из БД
  RecommendedPosts.tsx / RelatedProducts.tsx / RelatedArticles.tsx  # «related»-блоки
  ProductGrid.tsx     # сетка 3 колонки + бесконечный скролл + скелетоны
  ProductCardSkeleton.tsx / ImageWithFallback.tsx / CustomScrollbar.tsx
  CookieConsent.tsx / LegalArticle.tsx / ProductDetail.tsx / MarkdownText.tsx
  BackLink.tsx / PagePlaceholder.tsx / Button.tsx / icons/
lib/
  supabase/client.ts / server.ts  # браузерный/серверный клиенты (схема cozycorner), тип DbClient
  products.ts         # fetchProducts(+фильтры) + fetchFilterOptions + fetchProductBySlug
                      # + fetchRelatedProducts + resolveProductImage + formatPrice
  categories.ts / posts.ts / search.ts / content.ts  # загрузчики сущностей
  markdown.ts         # stripMarkdown() — плоский текст для мета-тегов
  consent.ts          # cookie-согласие (гейт аналитики)
  types.ts            # Product/Category/Post, PAGE_SIZE, POSTS_PAGE_SIZE
supabase/migrations/  # единственное место изменения схемы БД (см. schema.md §6–§7)
```

Папки `docs/` в репозитории больше нет — вся документация в `platform-docs/` (см. §7).

## 2. Рендеринг и SEO

- Первая страница товаров грузится на сервере (`app/page.tsx`) и попадает в HTML
  (индексируется). Дефолтные метаданные — в `app/layout.tsx`; SEO конкретной
  страницы отдаёт `generateMetadata()`.
- **Все контентные страницы пререндерятся статически с ISR**
  (`export const revalidate = 60`) — правка контента в БД появляется на сайте
  в пределах ~минуты **без редеплоя** (проверено end-to-end на production).
- ⚠️ Supabase-клиент не передаёт в fetch своих опций кэша — запросы наследуют
  сегментные 60s. **Не добавляйте `force-cache` в клиент** — заморозит данные.
- ⚠️ **`app/layout.tsx` тоже держит `export const revalidate = 60`.** Root layout
  читает `header_settings` (ссылка на Instagram) при рендере; без своего `revalidate`
  это значение «запекается» в статический шелл и правки из админки не появляются до
  редеплоя. С сегментным `revalidate` шапка обновляется на том же 60s-такте, что и
  контент. Так же читается singleton `footer_settings` (текст футера). Оба —
  «сайт-глобальные настройки», не привязаны к странице (см.
  [../database/schema.md](../database/schema.md) §5, `fetchHeaderSettings`/
  `fetchFooterSettings` в `lib/content.ts`).
- `/search` — динамическая (`await searchParams` в Next 16 сам делает роут
  динамическим), `noindex, follow`.

## 3. Откуда страницы берут hero и SEO

Правило: **статические страницы** читают из `pages` (+ `hero_sections`),
**страницы сущностей** — из полей самой сущности. Пустое поле = фолбэк в коде,
поэтому в админке любое поле можно оставить незаполненным.

| Страница | SEO | Hero |
|---|---|---|
| `/` (home) | `pages` (`home`) | `hero_sections` |
| `/shop` | `pages` (`shop`) | `hero_sections` |
| `/blog` | `pages` (`blog`) | `hero_sections` |
| `/terms`, `/privacy`, `/about`, `/featured` | `pages` (одноимённый slug) | нет hero |
| `/shop/[category]` | `categories.seo_*` (фолбэк — автоформула из `name`) | `categories.hero_*` (фолбэки: `Category` / `name` / автоформула / фон `/shop`) |
| `/product/[slug]` | `products.seo_*` (фолбэк — `title`/`description`) | нет hero (детальная раскладка) |
| `/blog/[slug]` | `posts.seo_*` (фолбэк — `title`/`excerpt`) | собирается из полей поста |

- Загрузка: `fetchPageContent(client, slug)` → `{ page, hero }`; метадату собирает
  `pageMetadata(page, fallback)` (`lib/content.ts`) — title как `absolute`, плюс
  description и OpenGraph. Тела terms/privacy — `pages.body` (Markdown) через проп
  `body` у `LegalArticle`; NULL → встроенный текст в коде.
- Футер: `fetchFooterSettings()` — singleton `footer_settings`, фолбэки в коде.
  ⚠️ Amazon Associates-дисклеймер обязателен (см. schema.md, footer_settings).
- Компонент `Hero` один на все страницы: новая страница с hero = новая строка в БД.
  Тип пропса — `HeroContent` (Pick), `/blog/[slug]` собирает его из полей поста.

## 4. Картинки

- Формат хранения — [schema.md §4](../database/schema.md): относительный плоский
  ключ бакета `cozycorner-photos` ИЛИ внешний URL. Разбор — `resolveProductImage()`
  (`lib/products.ts`) → `{ url, unoptimized }`.
- Свои картинки оптимизируются через `next/image` (хост `*.supabase.co` разрешён в
  `next.config.ts`); **внешние рендерятся с `unoptimized`** — отдаются напрямую,
  минуя `/_next/image`, поэтому их хосты в `remotePatterns` добавлять не нужно.
  Почему не allowlist: широкий `remotePatterns` тратил бы квоту оптимизации Vercel
  на чужие CDN и делал бы `/_next/image` открытым прокси. Не менять этот подход.
- **Стиль отображения** задаёт `products.image_style`: `photo` — «живое» фото,
  `object-fit: cover` вплотную к краям; `cutout` — товар без фона / на белом
  (Amazon JPEG), `contain` с паддингами на **белой** плитке. Автоопределения нет
  (JPEG на белом не отличим по файлу; canvas упирается в CORS) — флаг ставится
  вручную. Картинки категорий (`categories.image_path`) — **всегда cutout**
  (PNG с прозрачным фоном); `categories.hero_image_path` — обычное фото.
- Битый URL → `ImageWithFallback` показывает плейсхолдер со знаком-лого.
- URL в БД не валидируется — ответственность на том, кто заполняет каталог.

## 5. Каталог, блог, поиск, навигация

- **Бесконечный скролл**: `ProductGrid` (клиентский) получает первую страницу
  пропсом и догружает через `IntersectionObserver` + `.range(from, to)`.
  ⚠️ Пагинация должна быть **детерминированной**: `fetchProducts` сортирует по
  `created_at desc, id desc`, `ProductGrid` отбрасывает уже загруженные `id`.
  Не убирайте ни то, ни другое — иначе строки «плавают» и дублируются key.
- **Фильтры** `/shop/[category]`: нативные select Brand/Price (`FiltersBar`,
  контролируемый) + `ShopCatalog` (выбор → `ProductFilters` → перезагрузка сетки);
  `baseFilters` страницы (категория) должны соответствовать `initialProducts`.
- **Блог**: `/blog/[slug]` рендерит секции из БД по `position` (текст — в узкой
  колонке `--content-narrow`, продуктовые сетки — на полной ширине), затем
  `RecommendedPosts`. SEO-посты (`post_type='seo'`) скрыты из лент и поиска,
  открываются по прямому URL, включены в sitemap — **без cloaking**.
- **Поиск**: `SearchBox` в шапке (live-панель) + `/search?q=`. Запросы — `ilike`
  через `.or()` с обязательной санитизацией (`sanitizeSearchQuery` — `,()` ломают
  синтаксис PostgREST). Детали — [cozycorner/search.md](cozycorner/search.md).
- **Навигация**: Header (`/`, `/shop`, `/blog`), Footer (`/about`, `/terms`,
  `/privacy`); `/featured` скрыт из меню, но роут жив. Заглушки — `/featured`,
  `/about`. Глобальный cookie-баннер `CookieConsent`: без согласия ничего
  необязательного не собирается (гейт `lib/consent.ts` → `hasAnalyticsConsent()`).
- **Motion**: шапка sticky auto-hide; появление — `Reveal` (GSAP + ScrollTrigger);
  скелетоны при догрузке; нативный скроллбар скрыт — оверлейный `CustomScrollbar`
  (контент не «прыгает»). ⚠️ `overflow-x` у `html/body` — только `clip`
  (не `hidden` — ломает sticky-шапку).

## 6. Окружение и деплой

```
NEXT_PUBLIC_SUPABASE_URL=https://zwrkphynupdubevzwdzy.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=sb_publishable_…   # публичный ключ, не secret!
```

- Обе переменные нужны всегда (главная тянет данные при сборке); шаблон —
  `.env.example`. Такие же — в Vercel (Production + Preview). Service-role сайту
  **не нужен**. Схема `cozycorner` должна быть в Exposed schemas.
- Vercel связан с GitHub-репо `nowmdr/cozycorner`: **пуш в `main` → авто-деплой**.
- Прод: **https://cozycorner-omega.vercel.app** (личный Hobby-scope — через
  Vercel MCP/CLI не виден, проверять прод напрямую по URL). Старые URL мертвы:
  `cozycorner-one.vercel.app` — 404, `cozycorner.vercel.app` — чужой проект.
- Если правки контента «не доходят» до сайта — сперва проверить в дашборде Vercel,
  что авто-деплой прошёл и домен указывает на актуальный production-деплой.
- Быстрая проверка, из какой БД собран прод: `curl -sL <прод-URL> | grep -o
  '<project-ref>'` — в HTML зашиты Storage-URL картинок.
- Админка (`/admin`, `/api/admin`, service-role) из репозитория **удалена** —
  управление контентом только через `web.admin`.

## 7. Глубокие доки сайта (папка [cozycorner/](cozycorner/design-system.md) рядом с этим файлом)

- [cozycorner/design-system.md](cozycorner/design-system.md) — токены, шрифт/веса,
  каталог компонентов, правила GSAP.
- [cozycorner/project-structure.md](cozycorner/project-structure.md) — конвенции
  файлов/папок/стилей + чек-лист коммита.
- [cozycorner/search.md](cozycorner/search.md) — устройство поиска (санитизация,
  лимиты, известные ограничения).
- `../archive/cozycorner/admin-photos.md` — историческая справка о старой встроенной
  админке (код удалён).
