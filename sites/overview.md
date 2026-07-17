# Сайты — обзор мультисайт-платформы

> Last updated: 2026-07-17 | Source project: cozycorner + web.admin (docs/multisite-migration.md)

## Модель

Платформа — несколько публичных сайтов на одном Supabase-проекте (**base-one**,
подключение и архитектура — [../database/schema.md](../database/schema.md) §1–§2)
с общей админкой (`web.admin`). У каждого сайта:

- своя Postgres-**схема** (`cozycorner.*`, `<site2>.*`) и свой Storage-**бакет**
  (`cozycorner-photos`, `<site2>-photos`);
- свой репозиторий и деплой на Vercel (пуш в `main` → авто-деплой);
- запись в реестре админки `web.admin/src/config/sites.ts`
  ({slug, label, projectUrl, anonKey, schema, bucket}).

## Действующие сайты

| Сайт | Репозиторий | Схема / бакет | Прод | Детали |
|---|---|---|---|---|
| CozyCorner | `cozycorner/` | `cozycorner` / `cozycorner-photos` | https://cozycorner-omega.vercel.app | [cozycorner.md](cozycorner.md) |

## Общие свойства сайтов

- **Только publishable/anon ключ** + RLS; service-role сайтам не нужен (запись —
  через админку под Supabase Auth).
- **Контент правится без редеплоя**: контентные страницы — SSG + ISR
  (`revalidate = 60`), правка в БД из админки появляется на сайте в пределах
  ~минуты. Не передавать в Supabase-клиент свои опции кэша (`force-cache`
  заморозит данные).
- **Пути картинок** — относительные плоские ключи бакета или внешние URL
  (контракт — [../database/schema.md](../database/schema.md) §4).
- SEO-посты (`post_type='seo'`): скрыты из лент/поиска, доступны по прямому URL,
  включены в sitemap; **без cloaking** (редиректы/UA-логика/noindex запрещены).

## Как добавить новый сайт (чек-лист)

1. **БД**: миграции в репозитории сайта — новая схема `<site>` (гранты по шаблону
   миграции 0017 сайта cozycorner), бакет `<site>-photos` (public read, write —
   `is_admin()`, шаблон — 0018); схему добавить в **Exposed schemas** (дашборд).
2. **Админка**: запись в `SITES` (`web.admin/src/config/sites.ts`); если сайт в
   другом Supabase-проекте — завести там админов и Auth (см.
   [../database/schema.md](../database/schema.md) §2).
3. **Репозиторий сайта**: env `NEXT_PUBLIC_SUPABASE_URL` +
   `NEXT_PUBLIC_SUPABASE_ANON_KEY` (или аналог), клиент на схему сайта.
4. **Документация**: создать `platform-docs/sites/<site>.md`, добавить сайт в
   таблицу выше, в `platform-docs/AGENTS.md` («Projects») и в корневой
   `AGENTS.md` воркспейса.
