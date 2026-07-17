# Login Flow (Phase 1: Каркас + Auth) Implementation Plan

> **СТАТУС: ВЫПОЛНЕН (2026-07-08).** Все задачи закрыты, e2e проверен (логин, is_admin,
> RLS-запись админа / отказ анону). Позже поверх плана: SiteSwitcher заменён на
> /dashboard с карточками сайтов; фикс бага первого логина (setQueryData до navigate);
> UI переведён на английский; системный шрифт SF. Итог — в CLAUDE.md.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Фаза 1 спеки (`docs/admin-app-spec.md` §4–6, §10.1): реестр сайтов, фабрика клиентов, auth-слой, LoginPage, RequireAuth, AppShell + SiteSwitcher, маршруты. Критерий: логин админа проходит, не-админ/аноним отсекается.

**Architecture:** Один Supabase-клиент на проект (schema `public` — держит сессию, auth/rpc/storage) + `getDb(site)` = `.schema(site.schema)` для данных (вариант из `multisite-migration.md`; избегаем двух GoTrue-инстансов на одном storageKey). Активный сайт — из URL `/:siteSlug/*`, прокидывается через outlet-context. Auth-состояние — TanStack Query (`["auth", projectUrl]`) + инвалидация по `onAuthStateChange`.

**Tech Stack:** react-router-dom v7 (BrowserRouter/Routes), TanStack Query, RHF + Zod v4 (`zodResolver`), shadcn: card, field, input, button, dropdown-menu, sonner, skeleton.

**Верификация:** тестов в проекте нет (спека их не предусматривает; критерии фазы — поведенческие). Проверки: `npm run build`, `npm run lint`, dev-сервер, curl-проба GoTrue-эндпоинта (провайдер Email), e2e-логин после заведения пользователя.

---

### Task 1: Реестр сайтов + фабрика клиентов

**Files:**
- Create: `src/config/sites.ts` — тип `SiteConfig` + массив `SITES` (спека §5, дословно).
- Create: `src/lib/supabase.ts`:

```ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";
import { SITES, type SiteConfig } from "@/config/sites";

const clients = new Map<string, SupabaseClient>();

// Один клиент на Supabase-проект (schema public): держит сессию, auth/rpc/storage.
// Сессии разных проектов изолированы уникальным storageKey.
export function getClient(site: SiteConfig): SupabaseClient {
  let client = clients.get(site.projectUrl);
  if (!client) {
    client = createClient(site.projectUrl, site.anonKey, {
      auth: {
        persistSession: true,
        autoRefreshToken: true,
        storageKey: `sb-admin-${site.projectUrl.replace(/\W+/g, "_")}`,
      },
    });
    clients.set(site.projectUrl, client);
  }
  return client;
}

// Данные сайта — через его схему, per-query.
export const getDb = (site: SiteConfig) => getClient(site).schema(site.schema);

export const getSiteBySlug = (slug: string | undefined) =>
  SITES.find((s) => s.slug === slug);
```

- [ ] Step 1: создать оба файла. Step 2: `npm run build` — проходит. Step 3: commit `feat: add site registry and per-project supabase client factory`.

### Task 2: Auth-слой

**Files:**
- Create: `src/lib/auth.ts` (спека §6; `rpc("is_admin")` идёт через клиент со schema public — наш `getClient`):

```ts
import type { Session } from "@supabase/supabase-js";
import type { SiteConfig } from "@/config/sites";
import { getClient } from "@/lib/supabase";

export async function signIn(site: SiteConfig, email: string, password: string) {
  const { error } = await getClient(site).auth.signInWithPassword({ email, password });
  if (error) throw error;
}

export async function signOut(site: SiteConfig) {
  await getClient(site).auth.signOut();
}

export async function getSession(site: SiteConfig): Promise<Session | null> {
  const { data } = await getClient(site).auth.getSession();
  return data.session;
}

export async function isAdmin(site: SiteConfig): Promise<boolean> {
  const { data, error } = await getClient(site).rpc("is_admin");
  if (error) return false;
  return data === true;
}
```

- [ ] Step 1: создать файл. Step 2: `npm run build`. Step 3: commit `feat: add auth layer (signIn/signOut/getSession/isAdmin)`.

### Task 3: LoginPage + providers + маршруты (минимум)

**Files:**
- Modify: `src/main.tsx` — StrictMode + QueryClientProvider + BrowserRouter + `<App/>` + `<Toaster/>`.
- Modify: `src/App.tsx` — Routes: `/login`, `/` → redirect на первый сайт, `/:siteSlug` (guard + shell, Task 4), `*` → `/`.
- Create: `src/features/auth/LoginPage.tsx` — Card + FieldGroup/Field/FieldLabel/FieldError + Input + Button; RHF + `zodResolver`; сабмит: `signIn` → `isAdmin` (false ⇒ `signOut` + toast «Нет доступа») → `navigate(from ?? /${site.slug})`. Ошибку `Invalid login credentials` мапить в понятное сообщение. Сайт логина: из `location.state.from` или `SITES[0]`.
- Modify: `index.html` — `lang="ru"`, `<title>Site Admin</title>`.

- [ ] Step 1: main.tsx + App.tsx (маршруты `/login` и заглушки). Step 2: LoginPage. Step 3: `npm run build` + dev-проверка `/login`. Step 4: commit `feat: add login page with providers and routing`.

### Task 4: RequireAuth + AppShell + SiteSwitcher + защищённые маршруты

**Files:**
- Create: `src/components/RequireAuth.tsx` — из `useParams().siteSlug` резолвит сайт (нет ⇒ `Navigate /`); внутренний `Guard`: `useQuery(["auth", projectUrl])` → `{session, admin}`; подписка `onAuthStateChange` ⇒ инвалидация; pending ⇒ Skeleton; нет сессии ⇒ `Navigate /login state={{from}}`; не админ ⇒ экран «Нет доступа» + кнопка «Выйти»; иначе `<Outlet context={site}/>`.
- Create: `src/components/AppShell.tsx` — топбар (название, SiteSwitcher, «Выйти» ⇒ `signOut` + navigate `/login`), `<main><Outlet context={site}/></main>`.
- Create: `src/components/SiteSwitcher.tsx` — DropdownMenu по `SITES`, переключение слага с сохранением подпути.
- Modify: `src/App.tsx` — `/:siteSlug` = `<RequireAuth/>` → вложенный `<AppShell/>` → index-заглушка Dashboard.

- [ ] Step 1: RequireAuth. Step 2: AppShell + SiteSwitcher. Step 3: маршруты в App.tsx. Step 4: `npm run build` + `npm run lint`. Step 5: commit `feat: add auth guard, app shell and site switcher`.

### Task 5: Верификация поведения

- [ ] Step 1: dev-сервер: `/` без сессии редиректит на `/login`; `/login` рендерится.
- [ ] Step 2: curl-проба GoTrue: `POST {projectUrl}/auth/v1/token?grant_type=password` с фейковыми кредами. `invalid_credentials` ⇒ Email-провайдер включён; `email_provider_disabled` ⇒ попросить пользователя включить.
- [ ] Step 3: у пользователя: создать Auth-пользователя (Dashboard → Authentication → Add user). Затем через MCP: `insert into public.admin_users (user_id, email) values (...)`.
- [ ] Step 4: e2e: логин в браузере пользователем-админом; проверка отсечения анона (запись в БД без сессии отклоняется RLS).
