# Кросс-проектные стандарты

> Last updated: 2026-07-17 | Source project: cozycorner + web.admin (CLAUDE.md обоих репо)

Правила, действующие во **всех** проектах воркспейса. Проектные конвенции — в
`AGENTS.md` соответствующего репозитория.

## Языки

- **Текст интерфейса** (лейблы, сообщения, тосты, контент сайтов) — английский.
- Документация и комментарии в коде — русский ок.
- В UI сайтов не добавлять русский текст (фолбэки шрифтов на разных платформах
  различаются — теряется единообразие).

## Безопасность и ключи

- **Только publishable/anon ключи** во фронтенде и env с префиксами
  `NEXT_PUBLIC_*` / `VITE_*`. Ключ `sb_secret_…` (service role) — никогда:
  он уедет в браузер. Защита данных — Supabase Auth + RLS (`public.is_admin()`).
- Запись в БД/Storage — только под сессией админа из whitelist `public.admin_users`.
- Cookie-согласие на сайтах: необязательные скрипты (аналитика) подключаются
  только через гейт согласия (`hasAnalyticsConsent()`).

## Данные

- **Схема БД меняется только миграциями в репозитории cozycorner**
  (`supabase/migrations/`). Применённое через MCP/дашборд — продублировать файлом
  миграции (идемпотентно, `if not exists`). Полный workflow —
  [../database/schema.md](../database/schema.md) §7.
- **Пути картинок** в БД — относительные плоские ключи бакета или внешние URL;
  абсолютные URL своего Storage запрещены ([schema.md §4](../database/schema.md)).
- Пустые optional-поля → `null`, не пустые строки.
- `slug` и `updated_at` генерируют БД-триггеры — вручную не задавать.
- **Детерминированная сортировка/пагинация**: всегда вторичный ключ
  (`id desc` / `title asc`) — timestamps совпадают пачками, без tiebreaker'а
  порядок нестабилен, строки «плавают» между страницами.
- Легаси-поля с явным запретом (`posts.content`, `pages.meta`) — не читать и не
  писать; использовать явные списки колонок вместо `*`.

## SEO

- `seo_title` / `seo_description` — «живые» фолбэки: пустое значение в БД = сайт
  использует title/description сущности; не замораживать копией.
- SEO-посты: **никакого cloaking'а** — без редиректов, User-Agent-логики и
  noindex; canonical указывает на самого себя. Контракт создания —
  `../admin-panel/seo-posts-skill.md`.

## Рабочий процесс

- **Не коммитить и не пушить без явного разрешения пользователя.**
- Перед правками раздела админки — прочитать его доку
  ([../admin-panel/](../admin-panel/api.md)); перед фронтом cozycorner —
  [../sites/cozycorner/design-system.md](../sites/cozycorner/design-system.md) и
  [project-structure.md](../sites/cozycorner/project-structure.md).
- Проверка перед завершением: cozycorner — `npm run build` + `npm run lint`;
  web.admin — `npm run build` + `npm run lint` (oxlint).
- После изменения архитектуры / API / схемы БД — обновить соответствующий файл
  в `platform-docs/` (правило из корневого AGENTS.md).
- Переиспользование прежде нового кода: generic-компоненты (`RichTextEditor`,
  `ImagePickerDialog`, `FoldersPanel`/`useFolders`) и паттерны (гард
  несохранённых изменений, deferred-поиск, SEO-фолбэк) — расширять, не плодить
  локальные копии.
