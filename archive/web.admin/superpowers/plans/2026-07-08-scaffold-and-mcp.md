# Scaffold + Supabase MCP Setup Implementation Plan

> **СТАТУС: ВЫПОЛНЕН (2026-07-08).** Все задачи закрыты. Отклонения от плана: shadcn v3
> (radix + пресет nova, `field` вместо `form`), без `baseUrl` (deprecated в TS 6),
> oxlint вместо ESLint. Итог — в CLAUDE.md.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Инициализировать сборку админки (Vite + React + TS + Tailwind v4 + shadcn/ui + зависимости данных) и настроить Supabase MCP на проект `nkaobsivfzsjqypuaamw`, строго по `docs/admin-app-spec.md` §3.

**Architecture:** Пустой репозиторий → скаффолдинг Vite (через временную папку, т.к. каталог не пуст — есть `docs/` и `README.md`) → Tailwind v4 как Vite-плагин → алиасы `@/*` → shadcn init + компоненты → зависимости (supabase-js, router, tanstack-query, RHF+Zod) → CLAUDE.md из шаблона спеки → Supabase MCP через `claude mcp add`. Код приложения (auth, фото-менеджер) — НЕ в этом плане, это следующие фазы.

**Tech Stack:** Vite 7, React 19, TypeScript, Tailwind CSS v4 (`@tailwindcss/vite`), shadcn/ui, @supabase/supabase-js, react-router-dom, @tanstack/react-query, react-hook-form + zod.

---

### Task 1: Скаффолдинг Vite React-TS в непустой каталог

**Files:**
- Create: `package.json`, `index.html`, `vite.config.ts`, `tsconfig.json`, `tsconfig.app.json`, `tsconfig.node.json`, `eslint.config.js`, `src/*`, `public/*`, `.gitignore`

- [ ] **Step 1: Сгенерировать шаблон во временной папке и перенести в репозиторий**

`npm create vite@latest . …` в непустом каталоге задаёт интерактивный вопрос — обходим через temp-папку. Существующий `README.md` репозитория сохраняем (не перезаписываем шаблонным).

```bash
SCRATCH=/private/tmp/claude-501/-Users-mdr-Documents-Projects-web-admin/b9d8bc9d-75d3-4fe8-a03e-b74a80ab09c6/scratchpad
cd "$SCRATCH" && npm create vite@latest admin-scaffold -- --template react-ts
rsync -a --exclude README.md "$SCRATCH/admin-scaffold/" /Users/mdr/Documents/Projects/web.admin/
```

- [ ] **Step 2: Установить зависимости и проверить сборку**

```bash
cd /Users/mdr/Documents/Projects/web.admin && npm install && npm run build
```

Expected: `vite build` завершился, папка `dist/` создана, ошибок TS нет.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "chore: scaffold Vite + React + TypeScript app"
```

### Task 2: Tailwind v4 (Vite-плагин)

**Files:**
- Modify: `vite.config.ts`
- Modify: `src/index.css` (полностью заменить содержимое)

- [ ] **Step 1: Установить пакеты**

```bash
npm install tailwindcss @tailwindcss/vite
```

- [ ] **Step 2: Подключить плагин и алиас в `vite.config.ts`** (сразу с алиасом `@` для Task 3)

```bash
npm install -D @types/node
```

```ts
import path from "node:path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

- [ ] **Step 3: Заменить `src/index.css`** — всё шаблонное содержимое на:

```css
@import "tailwindcss";
```

Также удалить `src/App.css` и его импорт из `src/App.tsx` (шаблонные стили не нужны, дальше стили — Tailwind/shadcn).

- [ ] **Step 4: Проверить сборку**

```bash
npm run build
```

Expected: сборка проходит.

### Task 3: Алиасы путей `@/*` в tsconfig

**Files:**
- Modify: `tsconfig.json`
- Modify: `tsconfig.app.json`

- [ ] **Step 1: В `tsconfig.json` добавить** (на верхний уровень объекта):

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

(У Vite-шаблона `tsconfig.json` — это references-обёртка; `compilerOptions` добавить в неё, shadcn CLI читает именно корневой файл.)

- [ ] **Step 2: В `tsconfig.app.json` в `compilerOptions` добавить:**

```json
"baseUrl": ".",
"paths": { "@/*": ["./src/*"] }
```

- [ ] **Step 3: Проверка** — временно заменить в `src/App.tsx` импорт `./App.css`-стиля не нужно; просто:

```bash
npm run build
```

Expected: сборка проходит.

- [ ] **Step 4: Commit (Tasks 2–3 вместе)**

```bash
git add -A && git commit -m "chore: add Tailwind v4 and @/* path aliases"
```

### Task 4: shadcn/ui init + компоненты

**Files:**
- Create: `components.json`, `src/components/ui/*`, `src/lib/utils.ts`
- Modify: `src/index.css` (shadcn допишет CSS-переменные тем)

- [ ] **Step 1: Свериться с актуальным CLI** (требование спеки — не полагаться на память):

```bash
npx shadcn@latest init --help
```

- [ ] **Step 2: Инициализация** (неинтерактивно, base color neutral):

```bash
npx shadcn@latest init --base-color neutral --yes
```

Если флаги отличаются от `--help` — скорректировать по факту.

- [ ] **Step 3: Добавить компоненты из спеки §3:**

```bash
npx shadcn@latest add button input label card table dialog dropdown-menu sonner badge skeleton form
```

- [ ] **Step 4: Проверить сборку**

```bash
npm run build
```

Expected: сборка проходит, `src/components/ui/` содержит добавленные компоненты.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: init shadcn/ui with base components"
```

### Task 5: Зависимости данных/маршрутов/форм

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Установить**

```bash
npm install @supabase/supabase-js react-router-dom @tanstack/react-query react-hook-form zod @hookform/resolvers
```

- [ ] **Step 2: Проверка + commit**

```bash
npm run build && git add -A && git commit -m "chore: add supabase-js, router, query, forms deps"
```

### Task 6: CLAUDE.md из шаблона спеки

**Files:**
- Create: `CLAUDE.md` — содержимое взять дословно из `docs/admin-app-spec.md` §«Шаблон CLAUDE.md» (строки 360–399).

- [ ] **Step 1: Создать файл, Step 2: Commit**

```bash
git add CLAUDE.md && git commit -m "docs: add CLAUDE.md from admin-app-spec template"
```

### Task 7: Supabase MCP

Официальный сервер `@supabase/mcp-server-supabase` требует **Personal Access Token** (создаётся на https://supabase.com/dashboard/account/tokens) — publishable-ключ из спеки НЕ подходит (он для API данных, не для management/MCP). Токен запросить у пользователя.

- [ ] **Step 1: Добавить MCP (project-scoped на ref из спеки), токен — от пользователя:**

```bash
claude mcp add supabase -s local \
  -e SUPABASE_ACCESS_TOKEN=<токен-от-пользователя> \
  -- npx -y @supabase/mcp-server-supabase@latest --project-ref=nkaobsivfzsjqypuaamw
```

(`-s local` — токен-секрет не попадает в коммитимый `.mcp.json`.)

- [ ] **Step 2: Проверить подключение**

```bash
claude mcp list
```

Expected: `supabase: … - ✓ Connected`.

### Task 8: Финальная проверка

- [ ] **Step 1:** `npm run lint` — чисто; `npm run build` — проходит.
- [ ] **Step 2:** Убедиться, что `.env*` в `.gitignore` (Vite-шаблон включает `*.local`), `git status` чистый.
