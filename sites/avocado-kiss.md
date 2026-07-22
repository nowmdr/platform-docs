# Avocado Kiss — архитектура сайта

> Last updated: 2026-07-20 | Source project: avocado.kiss (AGENTS.md,
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
  RecipeCard.tsx        # карточка рецепта: варианты large / medium / list / wide
  EditorsPicks.tsx      # секция «Editor's Picks»: нумерованный список + sticky-карточка
  NewsletterBlock.tsx / NewsletterForm.tsx  # декоративный блок рассылки (submit никуда не пишет)
  Footer.tsx            # подвал (текст из footer_settings)
  Reveal.tsx            # GSAP reveal-обёртка (prefers-reduced-motion учтён)
  icons/                # ChevronLeftIcon, ChevronRightIcon, ClockIcon, MenuIcon, SearchIcon, UsersIcon
lib/
  supabase/client.ts / server.ts  # браузерный/серверный клиенты (db.schema='avocado_kiss'), тип DbClient
  content.ts            # fetchCategories, fetchCategoryBySlug, fetchHomeSlots (сгруппировано по слоту),
                        # fetchRecipeBySlug, fetchRecipesByCategory, fetchRecipeSlugs,
                        # fetchPageSeo, fetchFooterSettings
  images.ts             # resolveRecipeImage() (контракт путей картинок)
  types.ts              # Recipe/Category/HomeSlot/PageSeo/FooterSettings, HOME_SLOTS
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

Слоты и ёмкость по макету (валидирует админка фазы B; сайт рендерит что есть,
не проверяет лимиты):

| Слот | Ёмкость | Где на странице |
|---|---|---|
| `hero` | 3 | Hero-карусель (слайды) |
| `grid_large` | 1 | Большая карточка сетки Recipes (6 колонок) |
| `grid_medium` | 2 | Колонка средних карточек (3 колонки, aspect 1/1) |
| `grid_list` | 5 | Список текстовых ссылок (3 колонки, border-l) |
| `wide` | 2 | Широкие карточки под сеткой (фото 4/5 + текст) |
| `pick` | 3 | Editor's Picks — нумерованный список 1–3 |
| `pick_feature` | 1 | Editor's Picks — sticky-карточка справа |

Оверрайды и фолбэки (поля `home_slots`, не `recipes`):
- `eyebrow` — оверрайд лейбла над карточкой/hero-заголовком; пусто →
  категория рецепта (или строка «Recipe of the week» для hero — фолбэк в
  коде `app/page.tsx`, а не в БД).
- `eyebrow_secondary` — второй лейбл в Editor's Picks («Food | Drinks»).
- `description` — оверрайд описания (используется у `grid_large`); пусто →
  `recipe.excerpt`.

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
- Единственное поле картинки в v1 — `recipes.hero_image_path` (используется и
  как обложка карточки, и как hero-фото страницы рецепта); отдельного поля
  под картинку категории в v1 нет.
- Битый/пустой путь → `resolveRecipeImage` возвращает `null`, компонент
  просто не рендерит `<Image>` (в отличие от cozycorner там нет отдельного
  fallback-компонента с плейсхолдером — сделать при появлении такой
  потребности).

## 5. GSAP-анимации

- **`HeroCarousel`** (`components/HeroCarousel.tsx`, клиентский): слайды
  наложены друг на друга (`position: absolute`), активный проявляется
  GSAP-кроссфейдом (`autoAlpha`, `power2.inOut`, 900ms). Автоплей каждые 6с
  с паузой при hover (`hovered` ref, не state — не триггерит лишний рендер);
  точки + стрелки-кнопки переключают вручную. `prefers-reduced-motion` →
  мгновенное переключение (`duration: 0`).
- **`Reveal`** (`components/Reveal.tsx`, клиентский): обёртка вокруг секций/
  карточек — GSAP `from()` (`autoAlpha: 0, y: 24`) через `ScrollTrigger`
  (`start: "top 88%", once: true`), `@gsap/react`'s `useGSAP` со `scope`.
  Контент рендерится на сервере как обычно — прячет и анимирует его только
  клиентский GSAP после гидрации, поэтому SEO/no-JS не страдают.
  `prefers-reduced-motion` → анимация не запускается, контент остаётся
  видимым. Пропс `delay` — для стаггера соседних карточек (`wide`-ряд).
- Hover-зум картинок карточек и смена цвета заголовков — чистый CSS
  (transition), без GSAP — как в cozycorner.

## 6. Навигация и структура страниц

- **Header**: sticky, blur-фон; строка поиска (декоративная — не работает,
  `aria-label` для a11y), логотип по центру, кнопки поиска/меню на мобильном;
  под ней — `CategoryNav` из `fetchCategories()` (сортировка `position, name`).
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
