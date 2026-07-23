# Тестирование

> Last updated: 2026-07-23 | Source: workspace-one (web.admin + cozycorner + avocado.kiss)

Кросс-проектная тестовая стратегия воркспейса. Проектные детали — в `AGENTS.md`
каждого репо; операционные подсказки для агента — в project-skill
`.claude/skills/testing/` (корень воркспейса).

## Найденный стек

- **web.admin** — Vite 8 + React 19 SPA (react-router, TanStack Query, RHF+Zod,
  shadcn/ui), данные из Supabase.
- **cozycorner**, **avocado.kiss** — Next.js 16 (App Router, RSC), React 19,
  CSS Modules, данные из Supabase (SSG + ISR `revalidate = 60`).
- Package manager — **npm** во всех трёх (три отдельных репо, не монорепо).
- «Бэкенд» — управляемый Supabase (Postgres + RLS + Storage). Своего Node-сервиса,
  воркеров, очередей нет; серверная логика = RSC-лоадеры в `lib/*` и роут-хендлеры
  `robots.ts`/`sitemap.ts`.

## Выбранные инструменты и обоснование

| Уровень | Инструмент | Почему |
|---|---|---|
| Unit / integration | **Vitest 4 + React Testing Library** | Стандарт для App Router; в web.admin переиспользует уже настроенный Vite. Jest не вводим — на React 19 / Next 16 / Vite 8 он даёт больше трения. |
| E2E | **Playwright** | Рекомендуемый e2e и для Next, и для SPA. Cypress не добавляем, чтобы не держать два раннера под одну цель. |
| Agent-driven browser QA | **Playwright MCP** (project-level `.mcp.json`) | Контентные сайты + формы админки — реальный кейс для проверки UI-флоу агентом в браузере. |

Асинхронные Server Components Vitest пока не рендерит — их поведение проверяется
чистыми unit-тестами лоадеров и сквозными e2e, а не рендером RSC в jsdom.

## Где лежат тесты

- Unit/integration — рядом с кодом, `*.test.ts(x)`
  (web.admin: `src/**`; Next-сайты: `app/**`, `components/**`, `lib/**`).
- E2E — `e2e/*.spec.ts` в каждом репо.
- Конфиги: `vitest.config.ts` (в web.admin — блок `test` в `vite.config.ts`),
  `vitest.setup.ts`, `playwright.config.ts`, отдельный `tsconfig.vitest.json`.

## Как запускать

Из каталога нужного репо:

```bash
npm test              # unit/integration (vitest run)
npm run test:unit     # то же
npm run test:watch    # vitest watch
npm run test:coverage # покрытие (v8)
npm run test:e2e      # playwright test (поднимает свой dev-сервер)
```

Первичная установка браузеров e2e: `npx playwright install chromium`
(кеш браузеров глобальный и общий для всех репо на одной версии Playwright).

### Требования к окружению e2e

- **web.admin** — Supabase не нужен: смоук проверяет экран логина и клиентскую
  валидацию формы (dev-сервер Vite на :5173).
- **cozycorner (:3000)**, **avocado.kiss (:3100)** — нужен живой Supabase
  (`.env.local`, publishable/anon-ключи). Порты разные намеренно: иначе
  `reuseExistingServer` заставил бы один сайт тестироваться против сервера другого.

## Baseline-покрытие (эталонные примеры)

- web.admin: `resolveImageUrl`/`toStoragePath`, `formatBytes`, рендер компонента.
- cozycorner: `resolveProductImage`, `sanitizeSearchQuery`/`searchPath`, cookie-consent
  (localStorage), `<MarkdownText>` (whitelist разметки).
- avocado.kiss: `resolveRecipeImage`, `<RecipeCard>` (варианты, next/image+link мокнуты).
- E2E-смоук в каждом репо (1–2 ключевых пути).

Это задаёт паттерн, а не полное покрытие — расширять по мере фич.

## Генерация новых тестов через Claude Code

Скилл `testing` (корень воркспейса) активируется при работе с тестами и содержит
пошаговую инструкцию: выбор уровня, готовые паттерны, ловушки этого репо
(контракт картинок, моки next/image, порты e2e). Playwright MCP используется для
интерактивной проверки флоу в браузере.

## Добавление нового проекта (чек-лист)

Чтобы новый репо сразу попал в тест-маршрутизацию агентов (любых, не только
Claude), выполни по порядку. Ключевой шаг — №1: агенты берут проекты из реестра.

1. **Зарегистрируй в реестре** — добавь строку в «Реестр проектов» корневого
   `AGENTS.md`: путь, фреймворк, unit-раннер, e2e-раннер, **свободный e2e-порт**
   (не занятый другими: сейчас заняты 3000/3100/5173 — бери, напр., 3200), нужен
   ли live Supabase. После этого маршрутизация («напиши тесты для фичи в X»)
   работает без правок скилла/субагента.
2. **Документация** — добавь проект в `platform-docs/AGENTS.md` (Projects) и заведи
   `platform-docs/sites/<name>.md`.
3. **Зависимости** (в самом репо, npm): `vitest @vitest/coverage-v8 jsdom
   @testing-library/{react,dom,jest-dom,user-event} @playwright/test`; для Next —
   ещё `@vitejs/plugin-react vite-tsconfig-paths`. Vite-репо `@vitejs/plugin-react`
   обычно уже имеет.
4. **Конфиги:** `vitest.config.ts` (в Vite-репо — блок `test` в `vite.config.ts`)
   с `environment: 'jsdom'`, `setupFiles`, `include` тестов и `test.env` для
   Supabase-URL, если есть контракт картинок; `vitest.setup.ts` (jest-dom +
   cleanup; для Next — моки `next/image`/`next/link`); `playwright.config.ts`
   (`testDir: 'e2e'`, свой порт, `webServer: npm run dev -- -p <порт>`);
   `tsconfig.vitest.json`; `.gitignore` (coverage, test-results, playwright-report).
5. **Исключи тесты из прод-сборки** — добавь тест-файлы, `vitest.*`, `e2e` в
   `exclude` основного `tsconfig`.
6. **Скрипты** в `package.json`: `test`, `test:unit`, `test:watch`,
   `test:coverage`, `test:e2e` (единый naming во всех репо).
7. **Baseline-тесты** — 1–2 unit по чистым функциям + 1 компонентный; e2e-смоук
   ключевого флоу, если есть.
8. **Проверь:** `npm --prefix <project> test` зелёный (и `... run test:e2e` при
   наличии e2e + `npx playwright install chromium` один раз).

Эталон структуры — любой из существующих репо (`vitest.config.ts`,
`playwright.config.ts`, `*.test.ts`, `e2e/smoke.spec.ts`).

## Отношение к прод-проверкам

Тесты **дополняют**, а не заменяют обязательные проверки перед завершением:
`npm run build` + `npm run lint` в каждом репо (правило из
[coding-standards.md](coding-standards.md)).
