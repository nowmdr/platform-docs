# Админ-панель — статус, отклонения от спеки, следующие шаги

> Last updated: 2026-07-22 | Source project: web.admin (CLAUDE.md «Статус», сверено с фактическим состоянием репо cozycorner)

## Сделано (хронология по фазам)

- **Каркас**: Vite 8 + React 19 + TS, Tailwind v4 (vite-плагин), shadcn/ui.
  Шрифт — системный стек SF (Geist удалён).
- **Фаза 1 (Auth)**: `sites.ts`, `supabase.ts` (`getClient`/`getDb`), `auth.ts`,
  LoginPage (RHF+Zod+field), RequireAuth (сессия + is_admin + «Access denied»).
  Маршруты `/login` → `/dashboard` (карточки сайтов) → `/:siteSlug`. SiteSwitcher
  в шапке удалён — выбор сайта картами на /dashboard. Исправлен баг «первый логин
  не срабатывает» (сессию в кэш TanStack до navigate). `vercel.json` с SPA-rewrite;
  прод на Vercel задеплоен.
- **Фаза 2 (Media)**: менеджер фото по спеке §7 + доработки (грид 4 колонки,
  маркеры «не используется»/«>1 МБ», поиск, модалка деталей с rename, габаритами).
  Правила — [media.md](media.md); UX — [components.md](components.md) §3.6.
- **Фаза 3, шаг 1 (Products, v1–v4)**: CRUD товаров, `ImagePickerDialog`, generic
  `RichTextEditor` (TipTap v3 + @tiptap/markdown), SEO-фолбэк, гард несохранённых
  изменений (переход на data router). Правила — [products.md](products.md).
- **Фаза 3, шаг 2 (Blog)**: CRUD постов + конструктор модулей (diff-сохранение
  секций, пересинхронизация после Save). Правила — [blog.md](blog.md).
- **Итерация v2 раскладки редакторов**: sticky-панель действий, грид-шапка,
  фильтр статуса в списке постов.
- **Фаза 3, шаг 3 (Pages)**: SEO + hero-блоки (только update) + `pages.body`
  (markdown-тело terms/privacy). Правила — [pages.md](pages.md).
- **Переезд на Supabase-проект base-one (2026-07-16) — ЗАВЕРШЁН**: `sites.ts` →
  новый projectUrl/anonKey (коммит 569bcee), MCP-ref обновлён, строки
  `cozycorner.media` пересажены, оба админа заведены заново, публичная регистрация
  закрыта, e2e проверено на проде. Детали и диагностика —
  [../database/schema.md](../database/schema.md) §1, §8.
- **Папки в Media (2026-07-16)**: таблица `admin_folders` + `media.folder_id`
  (миграция `add_admin_folders` через MCP), FoldersPanel/MoveToFolderMenu,
  multi-select + bulk move.
- **Папки в Products/Blog + рефакторинг (2026-07-16)**: `products.folder_id`/
  `posts.folder_id` (миграция `add_content_folders` через MCP), общий код
  `src/features/folders/` (media переведён на него, `moveImages` удалён).
- **SEO Posts (2026-07-16)**: `posts.post_type` (миграция `add_seo_post_type`),
  разделы Blog и SEO Posts на одних компонентах (конфиг `sections.ts`), маршруты
  `/:siteSlug/seo-posts`, папки-секция `seo_posts`. Правила — [blog.md](blog.md) §1a.
- **Дубли MCP-миграций папок в репо сайта (2026-07-17)**: `0021_admin_folders.sql` +
  `0022_content_folders.sql` в `cozycorner/supabase/migrations/` — схемный дрейф
  закрыт (см. [../database/schema.md](../database/schema.md) §6).
- **UX-доработки медиа-выбора (2026-07-22)** — только фронт, без изменений схемы
  (спека/план `../archive/web.admin/superpowers/{specs,plans}/2026-07-22-picker-folders-bulk-delete-hover*`):
  - **Фильтр папок в `ImagePickerDialog`** — Select All/Unsorted/папки, upload в
    выбранную папку (пикер раньше папок не знал). Задействован в товарах, постах,
    страницах и hero-секциях (media.md §3).
  - **Bulk delete** во всех разделах (media/products/posts) — общий
    `BulkDeleteButton` + хук `useBulkDelete`; для media confirm предупреждает о
    занятых картинках. Дополняет прежний bulk move.
  - **Hover-превью** во всех точках добавления картинки — общий
    `ImagePreviewPicker` (ProductEditPage, PostEditPage cover, PageEditPage OG,
    hero bg); сам ведёт broken-состояние. У hero-секций превью раньше не было.
  - Код-ревью (память/эффективность): мемоизация counts/visible в пикере, убран
    двойной `form.watch` в hero, дедуп трёх мутаций удаления в `useBulkDelete`.

## Отклонения от спеки (актуальные версии инструментов)

- Алиасы `@/*` — только `paths` в tsconfig, без `baseUrl` (deprecated в TS 6).
- shadcn v3: библиотека radix + пресет nova; вместо `form` — `field`.
- Один supabase-клиент на проект (schema public, сессия/rpc/storage) + `getDb(site)` =
  `.schema(site.schema)` per-query — вместо двух клиентов из спеки §6.
- Линтер шаблона — oxlint (не ESLint). 2 warning в shadcn-файлах — известные, игнорируем.

## Выполнено на стороне сайта (сверено с репо cozycorner 2026-07-17)

- ~~Рендер markdown в `products.description` и `post_text_sections.body`~~ —
  сделано: компонент `MarkdownText` + `stripMarkdown()` для мета-тегов.
- ~~`pages.body` для terms/privacy + ISR~~ — сделано: `LegalArticle` с пропом
  `body`, все контентные страницы ISR `revalidate = 60`; миграция
  `0019_pages_body.sql` в репо сайта.
- ~~SEO-посты на сайте~~ — сделано: лента/поиск фильтруют `post_type='blog'`,
  sitemap включает оба типа; миграция `0020_posts_post_type.sql` в репо сайта.

## Следующие шаги

1. Ручные e2e-чек-листы (после переезда media/products в основном проверены):
   - blog — `../archive/web.admin/superpowers/plans/2026-07-10-blog.md` Task 6.1;
   - pages — `…/2026-07-10-pages.md` Task 5.3;
   - папки Media — `…/2026-07-16-media-folders.md` Task 6.3 (регресс после
     рефакторинга — важно);
   - папки Products/Blog — `…/2026-07-16-content-folders.md` Task 6.5;
   - SEO Posts — `…/2026-07-16-seo-posts.md` Task 11.
2. Фаза 3 дальше: **categories → footer_settings** (CRUD в админке).
3. От пользователя:
   - удалить старый Supabase-проект `nkaobsivfzsjqypuaamw` (этап G) + его
     MCP-сервер из `~/.claude.json`;
   - прод-URL админки добавить в base-one → Auth → URL Configuration (нужно для
     ссылок сброса пароля);
   - загрузить `seo-posts-skill.md` в проект claude.ai, создающий
     SEO-посты;
   - ротация secret key base-one, использованного для импорта при переезде.

## На потом (не забыть)

- **Cloudflare Turnstile**. Ключи у пользователя уже есть: Secret Key → Supabase
  Auth → Attack Protection (Turnstile); Site Key → фронт (виджет в LoginPage +
  `captchaToken` в `signInWithPassword`). Включать CAPTCHA в Supabase ТОЛЬКО после
  деплоя фронта с виджетом, иначе логин сломается для всех.
- Удалить легаси-колонку `posts.content` миграцией в репо сайта.
- RPC-атомарное сохранение поста (см. [blog.md](blog.md) §5).
- Тёмная тема (dark-токены есть, переключателя нет).
