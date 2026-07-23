# Дизайн-система CozyCorner

Модульная дизайн-система: единый источник правды для оформления. Все цвета,
типографика, скругления, тени и отступы объявлены как **CSS-переменные** в
[app/globals.css](../../../cozycorner/app/globals.css) в блоке `:root`. Компоненты (CSS Modules)
**не хардкодят** цвета и размеры — они ссылаются на токены через `var(--token)`.

> Меняете внешний вид всего сайта? Правьте токен в `:root`, а не значения в компонентах.

## Рабочий принцип (важно)

По ходу разработки **всё оформление превращается в переиспользуемые единицы**:

1. **Токен** — любое повторяющееся значение (цвет, размер, скругление, тень, отступ)
   заводится переменной в `:root` и используется через `var()`. Магических чисел и
   хардкод-цветов в компонентах быть не должно.
2. **Компонент** — любой самостоятельный кусок UI (карточка, кнопка, бейдж, секция)
   оформляется отдельным компонентом со своим CSS Module и документируется ниже в
   разделе «Компоненты».

Это не разовая задача, а постоянная дисциплина: увидели новый цвет/элемент —
сначала заведите токен/компонент, потом используйте. Структура файлов — в
[docs/project-structure.md](project-structure.md).

## Шрифт

Шрифт проекта — **SF Pro** (системный шрифт Apple). Его файлы по лицензии Apple
нельзя самохостить на вебе, поэтому `next/font` не используется — шрифт подключён
через системный стек в `--font-sans` ([app/globals.css](../../../cozycorner/app/globals.css)):
`-apple-system` / `BlinkMacSystemFont` резолвятся в SF Pro на macOS/iOS, остальные
платформы получают свой системный UI-шрифт (Segoe UI на Windows, Roboto на Android).
Стек применяется на `body`.

**Скруглённый шрифт заголовков hero — `--font-rounded`.** Заголовок `Hero .title`
использует **Nunito** (Google Fonts, OFL) весом Heavy (`--fw-heavy` 800). SF Pro
Rounded самохостить нельзя (та же лицензия Apple), а `ui-rounded` работает только
в Safari — поэтому Nunito **самохостится**: `@font-face` в globals.css +
`public/fonts/nunito-800-latin.woff2` (latin, `font-display: swap`). Стек:
`"Nunito", ui-rounded, "SF Pro Rounded", …` — на всех браузерах даёт скруглённый
Heavy. Больше нигде не применяется (только hero-заголовки).

| Токен | Значение | Назначение |
|---|---|---|
| `--fw-light` | 300 | лёгкий текст, лиды/описания |
| `--fw-regular` | 400 | основной текст |
| `--fw-medium` | 500 | **основной акцентный вес**: меню, кнопки, цены, заголовки карточек, бейджи |
| `--fw-bold` | 700 | крупные заголовки (напр. заголовки страниц) |
| `--fw-heavy` | 800 | заголовок hero (`--font-rounded` / Nunito) |

**Политика веса:** для «жирного» текста по умолчанию используем `--fw-medium` (500),
а не `--fw-bold` (700). 700 оставляем для крупных display-заголовков (+ одно
сознательное исключение: текст primary-кнопки, см. «Button»).

> Контент сайта — на английском (`<html lang="en">`). SF Pro кириллицу поддерживает,
> но фолбэки на других платформах различаются — не добавляйте русский текст в UI,
> чтобы не терять единообразие.

## Токены

### Цвета

| Токен | HEX | Назначение |
|---|---|---|
| `--color-bg` | `#ffffff` | фон страницы |
| `--color-surface` | `#f5f6f8` | приглушённый фон секций |
| `--color-card` | `#ffffff` | фон карточек |
| `--color-border` | `#ececec` | тонкие границы, разделители |
| `--color-text` | `#16181d` | заголовки и основной текст |
| `--color-text-muted` | `#6b7280` | вторичный текст, описания |
| `--color-primary` | `#4b4fe5` | индиго — кнопки CTA |
| `--color-primary-hover` | `#3a3ecb` | наведение на CTA |
| `--color-primary-contrast` | `#ffffff` | текст на primary |
| `--color-price` | `#2a2e6e` | тёмно-синий — цена |
| `--color-accent` | `#f5a623` | оранжевый — бейджи, акценты |

### Типографика (размеры)

`--fs-xs` 13px · `--fs-sm` 15px · `--fs-base` 16px · `--fs-lg` 20px ·
`--fs-xl` 28px · `--fs-2xl` адаптивный `clamp(2.25rem, 5vw, 3.5rem)` (h1).

### Скругления

`--radius-sm` 6px · `--radius-md` 12px · `--radius-lg` 20px ·
`--radius-pill` 999px.

### Тени

| Токен | Назначение |
|---|---|
| `--shadow-card` | базовая тень карточки товара |
| `--shadow-card-hover` | подъём карточки при наведении |
| `--shadow-float` | плавающая карточка hero |

### Отступы (шкала 4px)

`--space-1` 4px · `--space-2` 8px · `--space-3` 12px · `--space-4` 16px ·
`--space-5` 24px · `--space-6` 32px · `--space-7` 48px · `--space-8` 64px.

### Раскладка

`--content-max` 1200px — максимальная ширина контентного контейнера.
`--content-narrow` 860px — узкая колонка: динамические секции поста и любой длинный
текст, который не должен растягиваться на всю ширину.

## Фотостиль (imagery)

Эталон — фон hero страницы `/shop`
([Unsplash photo-1522708323590](https://images.unsplash.com/photo-1522708323590-d24dbb6b0267)).
Все фоновые фото hero, обложки блога и прочие иллюстрации подбираем в этой стилистике:

- **Светлые, воздушные интерьеры** с естественным дневным светом; окна в кадре —
  плюс. Никаких тёмных/мрачных сцен и вечерней подсветки.
- **Тёплая нейтральная база**: белый, кремовый, беж, светлое дерево.
- **1–2 тёплых акцента** на кадр (терракота, красный, оранжевый — перекликаются с
  `--color-accent`): красное кресло, плед, корешки книг. Не больше — иначе пестрит.
- **Обжитые пространства, а не рендеры**: растения, посуда, текстиль, лёгкая
  «жизнь» в кадре. Без людей крупным планом.
- **Композиция под hero**: визуальный центр тяжести — на стороне, противоположной
  текстовой карточке (`align: left` → интерес справа, и наоборот). Фон должен
  выдерживать кадрирование `cover` на разных ширинах.
- Источник — стоки (Unsplash и т.п.) или собственные загрузки в bucket `photos`;
  обложки блога кадрируются карточкой в 3:2 (`object-fit: cover`).

## Компоненты

Каталог UI-компонентов. Каждый — отдельный файл + CSS Module.

### Card — карточка товара

[components/ProductCard.tsx](../../../cozycorner/components/ProductCard.tsx) ·
[ProductCard.module.css](../../../cozycorner/components/ProductCard.module.css)

Базовый компонент-карточка проекта. Структура: `imageWrap` (плитка **4:3**) → `body`
(растягивается на свободную высоту) → `footer` (цена + кнопка, прибит к низу через
`margin-top: auto`). Цена в сетке и на странице товара — `--fs-lg` / `--fs-xl`.

Стиль фото зависит от `products.image_style` (миграция 0013, см.
[../../platform-docs/sites/cozycorner.md](../cozycorner.md) §4):

- `photo` — «живое» фото с фоном: `object-fit: cover` **вплотную к краям** плитки
  (как обложка `PostCard`), фон плитки `--color-surface` (виден только до загрузки);
- `cutout` — товар без фона или на белом фоне: `object-fit: contain` с паддингами
  (паддинг прямо на fill-картинке — `box-sizing` сжимает content box, отдельная
  внутренняя обёртка не нужна), фон плитки **белый** (`--color-card`) — у Amazon-картинок
  белый фон запечён в JPEG, на сером был бы виден шов.

Обоим типам — лёгкий зум фото при hover (`scale(1.05)`, `prefers-reduced-motion`
учтён; `overflow: hidden` на плитке — зум не вылезает за скругление).

- Карточка тянется на всю высоту ячейки сетки (`height: 100%`), поэтому при разной
  высоте контента низ карточек выровнен.
- Название (`title`) обрезается до **2 строк** с многоточием (`line-clamp: 2`).
- Тень `--shadow-card`, при наведении — подъём и `--shadow-card-hover`.
- **Вся карточка — ссылка на `/product/[slug]`** (паттерн stretched-link: `titleLink::after`
  накрывает карточку). Кнопка `Button` лежит выше оверлея (`z-index`) — «View on Amazon»
  работает отдельно и не триггерит переход.

### CategoryCard + CategoryGrid — карточки категорий на /shop

[components/CategoryCard.tsx](../../../cozycorner/components/CategoryCard.tsx) ·
[CategoryCard.module.css](../../../cozycorner/components/CategoryCard.module.css) ·
[components/CategoryGrid.tsx](../../../cozycorner/components/CategoryGrid.tsx) ·
[CategoryGrid.module.css](../../../cozycorner/components/CategoryGrid.module.css)

Хаб каталога `/shop` показывает вместо товаров карточки категорий (таблица
`categories`, миграции 0014–0015). `CategoryCard` намеренно повторяет пластику
`ProductCard` — те же `--radius-lg`, `--shadow-card`(`-hover`), подъём и зум при
hover, — но контент проще: cutout-картинка категории (`image_path`, PNG с прозрачным
фоном; правила как у cutout-товара — `contain` с паддингами на белой плитке 4:3) и
название по центру (`--fs-lg`, `--fw-medium`, над `border-top`). **Вся карточка —
ссылка на `/shop/[slug]`** (stretched-link, как у `ProductCard`).

`CategoryGrid` — серверная сетка этих карточек: та же раскладка 3 → 2 → 1, что у
`ProductGrid`, с Reveal-стаггером, но без пагинации — категорий немного, список
рендерится целиком.

### ProductDetail — страница товара

[components/ProductDetail.tsx](../../../cozycorner/components/ProductDetail.tsx) ·
[ProductDetail.module.css](../../../cozycorner/components/ProductDetail.module.css)

Раскладка страницы `/product/[slug]`: над обеими колонками — `BackLink` (ряд
`grid-column: 1 / -1`); слева фото, справа заголовок, описание (Markdown через
`MarkdownText`, типографика — в `styles.description`), мета-строки
Brand/Category (`dl`, показываются только заполненные), цена (`--fs-xl`) и кнопка
`Button`. На ≤900px — одна колонка.

Фото ветвится по `products.image_style` (те же правила, что в `ProductCard`):
`cutout` — белая плитка с паддингами и `object-fit: contain`; `photo` — `cover`
**без отступов**, вплотную к краям плитки (`overflow: hidden` держит скругление).

После `ProductDetail` страница рендерит `RelatedProducts` и `RelatedArticles`.

### MarkdownText — рендер Markdown-контента

[components/MarkdownText.tsx](../../../cozycorner/components/MarkdownText.tsx) ·
[MarkdownText.module.css](../../../cozycorner/components/MarkdownText.module.css)

Рендер полей, хранящихся в Markdown (сейчас — `products.description`; тот же
компонент планово переиспользуется для блога). Обёртка над `react-markdown` c
жёстким whitelist элементов (`allowedElements`: `p`, `ul`, `ol`, `li`, `strong`,
`em`, `br`; остальное разворачивается в текст через `unwrapDisallowed`, сырой
HTML не рендерится). Свой CSS Module задаёт **только внутренние** отступы между
блоками (`--space-3`), маркеры/отступы списков (`--space-5`) и вес `strong`
(`--fw-medium` по политике весов); размер/цвет/вес текста передаёт родитель
через `className` — компонент вписывается в типографику любого контекста.

Для мест, где нужен плоский текст (мета-теги, сниппеты), — хелпер
`stripMarkdown()` в [lib/markdown.ts](../../../cozycorner/lib/markdown.ts): снимает разметку и
схлопывает переносы, разметка в `<meta>` не попадает.

### BackLink — кнопка «назад»

[components/BackLink.tsx](../../../cozycorner/components/BackLink.tsx) ·
[BackLink.module.css](../../../cozycorner/components/BackLink.module.css)

`'use client'`. Текстовая кнопка со CSS-стрелкой (как каретка в старом FiltersBar):
`router.back()`, а при прямом заходе без истории (`window.history.length <= 1`) —
`router.push('/shop')`, чтобы кнопка не была «мёртвой».

### RelatedProducts / RelatedArticles — блоки в конце страницы товара

[components/RelatedProducts.tsx](../../../cozycorner/components/RelatedProducts.tsx) ·
[components/RelatedArticles.tsx](../../../cozycorner/components/RelatedArticles.tsx)

Родственники `RecommendedPosts`, но со своим контейнером (`--content-max` +
боковые паддинги): `ProductDetail` держит ширину только для себя. RelatedProducts —
`h2` «Related products» + 3 карточки `ProductCard` из той же категории
(`fetchRelatedProducts`: при нехватке добирает свежими из остальных, чтобы блок
всегда был полным). RelatedArticles — `h2` «Related articles» + 3 свежих поста
(`PostCard`, `fetchPosts(0, 3)`). Оба скрываются при пустом списке.

### PostCard — карточка поста блога

[components/PostCard.tsx](../../../cozycorner/components/PostCard.tsx) ·
[PostCard.module.css](../../../cozycorner/components/PostCard.module.css)

Родственник `ProductCard` (та же геометрия, тень, stretched-link, `height: 100%`),
но минимальная: обложка **вплотную к краям** (`aspect-ratio: 3/2`, `object-fit:
cover` — иллюстрация, не продуктовое фото) + заголовок + excerpt (clamp 3 строки).
Ни кнопки, ни даты, ни разделителя. Вся карточка — ссылка на `/blog/[slug]`; на
hover — подсветка заголовка и лёгкий зум обложки (`scale(1.05)`).

### PostTextSection / PostProductsSection — секции поста

[components/PostTextSection.tsx](../../../cozycorner/components/PostTextSection.tsx) ·
[components/PostProductsSection.tsx](../../../cozycorner/components/PostProductsSection.tsx)

Рендерят динамические секции single-поста из БД (`post_*_sections`, см.
[../../platform-docs/database/schema.md](../../database/schema.md) §5). Text: опциональный `h2` + абзацы (разделитель в `body` — пустая
строка); живёт в узкой колонке `--content-narrow`. Products: `h2` + сетка
`ProductCard` в 2 или 3 колонки (`columns` из БД); живёт на полной ширине
`--content-max` — в узкой колонке футер карточки «цена + кнопка» не помещается.

### RecommendedPosts — «Related posts»

[components/RecommendedPosts.tsx](../../../cozycorner/components/RecommendedPosts.tsx)

Блок в конце поста, после динамических секций: `h2` «Related posts» + до 3 свежих
постов (`PostCard`, без текущего). Не управляется из БД. Полная ширина контейнера.

### FiltersBar + ShopCatalog — фильтры каталога

[components/FiltersBar.tsx](../../../cozycorner/components/FiltersBar.tsx) ·
[FiltersBar.module.css](../../../cozycorner/components/FiltersBar.module.css) ·
[components/ShopCatalog.tsx](../../../cozycorner/components/ShopCatalog.tsx)

Рабочие фильтры каталога, сейчас — на страницах категорий `/shop/[category]` (оба
`'use client'`). `FiltersBar` — панель над сеткой: **нативные** `<select>` (Category —
опционален: без пропса `categories` селект не рендерится, как на странице категории;
Brand; Price-диапазоны из `PRICE_RANGES`, экспортируется там же) в облике
чипов-пилюль (`appearance: none`, своя стрелка — фоновый inline-SVG) +
secondary-`Button` «Clear» (появляется, только когда что-то выбрано; сбрасывает все
селекты). Список опций — системный, кастомных дропдаунов нет. Сам компонент
контролируемый и без состояния: `selection`/`onChange` приходят от `ShopCatalog`.

`ShopCatalog` — связка «панель + сетка»: держит выбор селектов, транслирует его в
`ProductFilters` (`lib/products.ts`) и передаёт в `ProductGrid` — фильтрация
применяется **сразу при изменении** селекта: сетка сбрасывается, показывает
скелетоны и грузит первую страницу заново с фильтрами (дальше бесконечный скролл
работает уже внутри отфильтрованной выборки; пустой результат — сообщение «No
products match these filters»). Опц. `baseFilters` — постоянные фильтры страницы
поверх выбора селектов (на `/shop/[category]` — категория; должны соответствовать
`initialProducts`, иначе сетка перезагрузится сразу после монтирования). Значения
для селектов собирает сервер: `fetchFilterOptions(client, category?)` — с `category`
в Brand попадают только бренды этой категории. Чистая композиция — своего CSS Module
у `ShopCatalog` нет.

### ProductCardSkeleton — скелетон карточки

[components/ProductCardSkeleton.tsx](../../../cozycorner/components/ProductCardSkeleton.tsx) ·
[ProductCardSkeleton.module.css](../../../cozycorner/components/ProductCardSkeleton.module.css)

Повторяет геометрию `ProductCard` (сетка не прыгает при догрузке). Мерцание — бегущий
градиент из токенов `--color-surface`/`--color-border`; при `prefers-reduced-motion`
статичен. Показывается в `ProductGrid` на время загрузки следующей страницы.

### ImageWithFallback — картинка с фолбэком

[components/ImageWithFallback.tsx](../../../cozycorner/components/ImageWithFallback.tsx) ·
[ImageWithFallback.module.css](../../../cozycorner/components/ImageWithFallback.module.css)

`'use client'`, обёртка над `next/image`. Плейсхолдер (`--color-surface` +
полупрозрачный `LogoMark`) рендерится в двух случаях: (1) файл не загрузился
(протухший внешний URL с Amazon), (2) картинки **нет вовсе** (`src` пустой —
`resolveProductImage` вернул `null`). Это **единый фолбэк для всех фото проекта**:
`ProductCard`, `CategoryCard`, `PostCard`, `ProductDetail` и миниатюры поиска
рендерят компонент всегда (без `{img && …}`), передавая `src={img?.url}`. Знак
масштабируется под контейнер (`width: 40%`, `max 56px`) — одинаково смотрится в
крупной карточке и в 44px-миниатюре. Контейнеры fill имеют `position` + `aspect-ratio`,
поэтому плейсхолдер занимает область даже при пустой картинке.

### CustomScrollbar — кастомный скроллбар страницы

[components/CustomScrollbar.tsx](../../../cozycorner/components/CustomScrollbar.tsx) ·
[CustomScrollbar.module.css](../../../cozycorner/components/CustomScrollbar.module.css)

`'use client'`, смонтирован в root layout. Нативный скроллбар скрыт в `globals.css`
(`scrollbar-width: none` + `::-webkit-scrollbar`), вместо него — fixed-полоса поверх
контента: **не занимает ширину вьюпорта**, поэтому контент не сдвигается между
страницами с/без скролла. Сам скролл остаётся нативным; полоса появляется при
скролле/наведении, поддерживает drag, прячется на нескроллящихся страницах.
Цвета — токены `--scrollbar-thumb` / `--scrollbar-thumb-active`.

### CookieConsent — cookie-баннер

[components/CookieConsent.tsx](../../../cozycorner/components/CookieConsent.tsx) ·
[CookieConsent.module.css](../../../cozycorner/components/CookieConsent.module.css)

`'use client'`, в root layout. Плавающая карточка снизу; показывается, пока выбор не
сделан. «Accept»/«Decline» пишут выбор в localStorage через
[lib/consent.ts](../../../cozycorner/lib/consent.ts). **Правило**: любые необязательные скрипты
(аналитика и т.п.) подключаются только через `hasAnalyticsConsent()` — при отказе
ничего не собирается.

### LegalArticle — обёртка юридических страниц

[components/LegalArticle.tsx](../../../cozycorner/components/LegalArticle.tsx) ·
[LegalArticle.module.css](../../../cozycorner/components/LegalArticle.module.css)

Узкая читабельная колонка (72ch): `h1` + «Last updated» + стили для `h2`/`p`/`ul`/`a`
из контента страницы. Используется на `/terms` и `/privacy`.

Опциональный проп `body` (Markdown из `pages.body`): непустой — вместо `children`
рендерится `MarkdownText` (класс `mdBody`); типографику p/ul/li дают те же
селекторы `.article` (внешние margin схлопываются — ритм совпадает со встроенным
контентом), а `padding-left` списков из модуля MarkdownText гасится селектором
`.article .mdBody ul/ol` (иначе отступ удвоился бы). NULL/пусто — `children`
(встроенный текст страницы), как раньше.

### PagePlaceholder — заглушка страницы

[components/PagePlaceholder.tsx](../../../cozycorner/components/PagePlaceholder.tsx). Простой «coming soon»
для ненаполненных роутов (сейчас — Featured/About).

### Hero — hero-секция

[components/Hero.tsx](../../../cozycorner/components/Hero.tsx) ·
[Hero.module.css](../../../cozycorner/components/Hero.module.css)

Бейдж + `h1` + лид в плавающей белой карточке (`--shadow-float`). Положение карточки —
из БД (`hero_sections.align`): `right` (home) или `left` (shop, blog). Компонент один
на все страницы — новая страница с hero = новая строка в БД, без нового кода.
Контент приходит пропсом из БД (`hero_sections`, см.
[../../platform-docs/database/schema.md](../../database/schema.md) §5).
Фон: если задан `bg_image_path` — рендерится слоем `HeroBackground` (`'use client'`,
[components/HeroBackground.tsx](../../../cozycorner/components/HeroBackground.tsx)) с **аккуратным
параллаксом**: слой выше секции на 20% и смещается на ±5% GSAP-scrub'ом при прокрутке;
при `prefers-reduced-motion` статичен. Иначе — заглушка-градиент. Карточка обёрнута в
`Reveal` (мягкое появление). Контент может приходить не только из `hero_sections`:
тип пропса — `HeroContent` (Pick), и `/blog/[slug]` собирает его из полей поста.
**Заголовок** (`.title`) — скруглённый `--font-rounded` (Nunito) весом `--fw-heavy`
(800), см. «Шрифт». **Высота на мобилке (≤720px)**: `min-height: calc(100svh -
var(--header-h))` — hero занимает первый экран за вычетом сплошной шапки; лид
плотнее (`line-height: 1.3`). На десктопе — прежний `clamp`.

### Header — глобальная шапка

[components/Header.tsx](../../../cozycorner/components/Header.tsx) ·
[Header.module.css](../../../cozycorner/components/Header.module.css)

Смонтирована в root layout → есть на всех страницах. Раскладка в 3 зоны
(`grid-template-columns: 1fr auto 1fr`): слева навигация, по центру лого, справа —
Instagram + поиск (именно в таком порядке). Активный пункт меню подсвечивается (`usePathname`).
На мобилке (≤720px) навигация прячется в бургер-меню, а зоны переставляются через
`order`/`justify-self` (разметку не трогаем): Instagram + иконка-поиск уходят влево,
лого — по центру, бургер — вправо. На мобилке поиск — не инлайн-инпут, а иконка,
открывающая полноэкранный оверлей (см. «Поиск» ниже).

**Ссылка на Instagram** — иконка `InstagramIcon` в правой зоне рядом с поиском.
URL приходит пропом `instagramUrl` (root layout читает singleton `header_settings`,
см. [../../database/schema.md](../../database/schema.md) §5 и раздел «Данные шапки»
в [cozycorner.md](../cozycorner.md)). **Пусто/недоступно → иконка не рендерится**
(безопасный фолбэк, как у текста футера). Внешняя ссылка (`target="_blank"`,
`rel="noopener noreferrer"`, `aria-label`), цвет `--color-text`, hover — `--color-primary`.

**Sticky auto-hide**: при скролле вниз шапка уезжает, при скролле вверх выезжает;
у верха страницы видна всегда, при проскролленной странице — лёгкая тень. Шапка
всегда сплошная (белый фон + нижняя граница) — на десктопе и на мобилке.
⚠️ `position: sticky` ломается, если у `body` появится `overflow-x: hidden` —
поэтому в `globals.css` используется `overflow-x: clip`.

**Поиск (`SearchBox`)**: на десктопе — инлайн-инпут с live-панелью (debounce 300ms,
≤5 строк, «See all N results»). На мобилке инпут скрыт, вместо него иконка,
открывающая **полноэкранный оверлей**: строка ввода + крестик сверху, ниже —
результаты (лимит **10** — используем вертикальное место), при пустом запросе —
подсказка «Type what you're looking for…». Прокрутка фона блокируется на время
оверлея; логика запроса/результатов общая с десктопом (`renderResults(limit)`).

### Button — кнопка

[components/Button.tsx](../../../cozycorner/components/Button.tsx) ·
[Button.module.css](../../../cozycorner/components/Button.module.css)

Переиспользуемая кнопка. **Полиморфна**: с `href` рендерится как `<a>`, иначе — `<button>`
(по умолчанию `type="button"`). Пропсы `variant` и `size` (`sm`|`md`|`lg`):

- `primary` — индиго CTA («View on Amazon», «Subscribe», «Accept»); вес текста
  `--fw-bold` (700) — сознательное исключение из политики весов, CTA читается жирнее;
- `secondary` — нейтральная с тонкой границей («Decline» в cookie-баннере), вес 500;
  на hover — синяя обводка и синий текст (`--color-primary`), transition включает
  `border-color`;
- `lg` — крупная CTA (паддинги `--space-3/--space-6`, шрифт `--fs-base`) — «View on
  Amazon» на странице товара (`ProductDetail`).

Новые вариации добавляем сюда, не плодя локальные кнопки.

### Footer — глобальный подвал

[components/Footer.tsx](../../../cozycorner/components/Footer.tsx) ·
[Footer.module.css](../../../cozycorner/components/Footer.module.css)

Async серверный компонент, смонтирован в root layout → на всех страницах, прижат к низу
(`margin-top: auto`). Верх в 2 колонки: слева знак-лого (`LogoMark`) + тэглайн +
навигация, справа — форма подписки `NewsletterForm`. Ниже — серая линия и мелкий
приглушённый текст. **Весь текст настраиваемый** — тянется из таблицы `footer_settings`
(см. [../../platform-docs/database/schema.md](../../database/schema.md) §5),
с фолбэками в коде.

### NewsletterForm — форма подписки

[components/NewsletterForm.tsx](../../../cozycorner/components/NewsletterForm.tsx) ·
[NewsletterForm.module.css](../../../cozycorner/components/NewsletterForm.module.css)

`'use client'`. Инпут в стиле поиска (pill) + кнопка `Button` **в столбик, оба на
всю ширину** (кнопка не обрезается на узких экранах). Живёт в футере. **Реальный
сбор подписок**: на сабмит запускается невидимый Cloudflare Turnstile → серверное
действие `lib/newsletter.ts` (проверка токена + запись в `subscribers` под
service_role). Статусы: Subscribing… / Thanks / ошибка (`--color-danger`). Полный
поток и ключи — [cozycorner.md](../cozycorner.md) §7.

### Icons — иконки-компоненты

[components/icons/](../../../cozycorner/components/icons/): `Logo`, `LogoMark`, `SearchIcon`, `MenuIcon`,
`CloseIcon`, `InstagramIcon`. Каждая — SVG-компонент с `SVGProps`, без фикс-размеров
(размер через CSS), цвет — `currentColor` (кроме брендовых цветов `Logo`/`LogoMark`).
`InstagramIcon` — обводка (`viewBox 0 0 24 24`, `fill=none`, `stroke-width 2`, скруглённые
концы) в одном стиле с `SearchIcon`/`MenuIcon`: скруглённый квадрат + кружок-объектив +
точка-вспышка (заливка `currentColor`). См. правило 9 в
[project-structure.md](project-structure.md).

### Элементы

- **Badge** — `Hero` (`.badge`): `--color-accent`, uppercase, `letter-spacing`.
  Кандидат на вынос в отдельный компонент, когда понадобится во втором месте.
- **Input (pill)** — одинаковый стиль pill-инпута сейчас продублирован в `Header`
  (`.searchInput`) и `NewsletterForm` (`.input`): высота 42px, `--radius-pill`, тонкая
  граница, фокус `--color-primary`. **Кандидат №1 на вынос** в общий компонент `Input`
  (используется уже в 2 местах) — сделать при следующем касании форм.

> Когда элемент используется во втором месте — выносим его в самостоятельный
> компонент `components/<Name>.tsx` и обновляем этот раздел.

## Анимация (GSAP)

Все анимации в проекте делаем через **GSAP** (`gsap`, установлен). Никаких сторонних
animation-библиотек и разрозненных `@keyframes` для сложных эффектов — единый инструмент.

**Reveal-анимация**: при появлении элементов на странице можно давать лёгкий reveal
(мягкое проявление + небольшой сдвиг), особенно для секций и карточек. Держим его
**сдержанным** (короткая длительность, малое смещение) — в духе бренда.

Принципы:

- GSAP работает только на клиенте → анимируем в компонентах с `'use client'` внутри
  `useGSAP` (`@gsap/react`). Серверные компоненты (RSC) сами не анимируют.
- **Прогрессивное улучшение для SEO**: контент должен быть в HTML и виден без JS.
  Начальное «скрытое» состояние задаём из JS (`gsap.from(...)`), а не в CSS — тогда без
  JS элемент просто отрисуется как есть, а текст останется индексируемым.
- **`prefers-reduced-motion` уважается всегда**: при reduce анимация не запускается.
- Длительности/смещения — тоже кандидаты в токены, если начнут повторяться.

Готовые анимационные компоненты (используйте их, не пишите новые ad-hoc твины):

- **[Reveal](../../../cozycorner/components/Reveal.tsx)** — универсальный reveal: fade + подъём 24px при
  входе во вьюпорт (ScrollTrigger, `once`), проп `delay` для стаггера соседей (карточки
  в ряду: `delay={(index % columns) * 0.08}`). Оборачивает hero-карточку и карточки
  всех сеток (товары, посты).
- **[HeroBackground](../../../cozycorner/components/HeroBackground.tsx)** — параллакс фона hero:
  слой выше секции на 20%, смещение ±5% scrub'ом (см. раздел Hero).

## Как добавить токен

1. Объявите переменную в `:root` в [app/globals.css](../../../cozycorner/app/globals.css) в нужной группе.
2. Используйте через `var(--token)` в CSS Module компонента.
3. Задокументируйте здесь в соответствующей таблице.
