# Blog Post Preview Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an editor open a shareable, non-indexed link to preview a blog post (draft or published) on the real cozycorner frontend without touching the public grid, gated by a per-post capability token.

**Architecture:** A `preview_token uuid` column on `cozycorner.posts` is a capability, not a visibility flag. cozycorner adds a `force-dynamic`, `noindex` route `app/preview/blog/[slug]/page.tsx` that reads the post with the existing **service-role** admin client and renders only when `preview_token` matches — showing **all** sections (ignoring per-section `is_published`). web.admin reads the token via its normal admin session and offers a "Copy preview link" button. The public grid, ISR, `generateStaticParams`, and the `is_published` gate are all untouched. The token is **revoked from the anon key** so it never ships to the public site.

**Tech Stack:** cozycorner (Next.js 16 App Router, RSC, Supabase), web.admin (Vite + React SPA, Supabase, RHF/Zod, shadcn/ui, sonner), Supabase Postgres migrations (schema `cozycorner`, project `zwrkphynupdubevzwdzy`).

---

## Critical sequencing constraint (read before starting)

The live prod cozycorner does `posts .select("*")` in three places. Adding `preview_token` and **revoking** anon `SELECT` on it would make `select("*")` fail on prod (`permission denied for column preview_token`). Therefore:

1. **Migration 0029 (ADD COLUMN)** is safe to apply early — `select("*")` just returns one extra column. Needed so admin + preview route can be developed against the live DB.
2. **Migration 0030 (REVOKE anon)** must be applied **only after** the cozycorner code that replaces `select("*")` with an explicit column list is **deployed to prod** (Task 3). It is created as a file now but applied last (Phase 7, with user approval).

During dev (after 0029, before 0030 + deploy) a *published* post's `preview_token` is briefly readable by the anon key. Drafts remain protected by RLS the whole time. This window is acceptable for the test-mode site; it closes at deploy.

**Commit/push policy for this work:** committing on feature branches is allowed. **Push, Vercel deploy, and applying migration 0030 require explicit user confirmation** (Phase 7).

## File structure

**cozycorner** (`feature/blog-post-preview`)
- `supabase/migrations/0029_posts_preview_token.sql` — new; add column.
- `supabase/migrations/0030_posts_preview_token_revoke_anon.sql` — new; revoke anon.
- `lib/types.ts` — add `Post.preview_token`.
- `lib/posts.ts` — add `PUBLIC_POST_COLUMNS`; switch `fetchPosts`/`fetchPostBySlug` off `select("*")`; add `fetchPostForPreview`; extend `fetchPostSections` with `{ includeUnpublished }`.
- `lib/search.ts` — `searchPosts` uses `PUBLIC_POST_COLUMNS`.
- `lib/posts.test.ts` — new; unit tests for the above.
- `app/preview/blog/[slug]/page.tsx` — new; dynamic, noindex, token-checked render.

**web.admin** (`feature/blog-post-preview`)
- `src/config/sites.ts` — add `frontendUrl?` to `SiteConfig` + cozycorner value.
- `src/lib/posts.ts` — `Post.preview_token`, exclude it from `PostInput`, add to `POST_COLUMNS`.
- `src/features/posts/CopyPreviewLinkButton.tsx` — new component.
- `src/features/posts/CopyPreviewLinkButton.test.tsx` — new test.
- `src/features/posts/PostEditPage.tsx` — render the button in the sticky action panel.

**platform-docs** (`feature/blog-post-preview`)
- `database/schema.md` — document `preview_token`, RLS note, migration history.
- `admin-panel/blog.md` — document the preview flow.
- `sites/cozycorner.md` — document the `/preview/blog/[slug]` route.

---

## Phase 0 — Setup

### Task 0: Feature branches

- [ ] **Step 1: Create branches in all three repos**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner checkout -b feature/blog-post-preview
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin checkout -b feature/blog-post-preview
git -C /Users/mdr/Documents/Projects/workspace-one/platform-docs checkout -b feature/blog-post-preview
```

- [ ] **Step 2: Verify**

```bash
for r in cozycorner web.admin platform-docs; do echo "[$r] $(git -C /Users/mdr/Documents/Projects/workspace-one/$r branch --show-current)"; done
```

Expected: each prints `feature/blog-post-preview`.

---

## Phase 1 — DB: add column (cozycorner repo + live DB)

### Task 1: Migration 0029 — add `preview_token`

**Files:**
- Create: `cozycorner/supabase/migrations/0029_posts_preview_token.sql`

- [ ] **Step 1: Write the migration file**

```sql
-- Превью-токен поста: capability-токен для просмотра черновика на реальном фронте
-- (спека platform-docs/superpowers/specs/2026-07-27-blog-post-preview-design.md).
-- Это НЕ флаг видимости — RLS, сетка /blog и generateStaticParams не меняются.
-- Бэкфилл существующих строк выполняется через default gen_random_uuid().
alter table cozycorner.posts
  add column preview_token uuid not null default gen_random_uuid();
```

- [ ] **Step 2: Apply to the live DB via Supabase MCP**

Use `mcp__supabase__apply_migration` with `project_id: "zwrkphynupdubevzwdzy"`, `name: "0029_posts_preview_token"`, and the SQL above.

- [ ] **Step 3: Verify the column exists and every row has a token**

Use `mcp__supabase__execute_sql` (`project_id: "zwrkphynupdubevzwdzy"`):

```sql
select count(*) as total, count(preview_token) as with_token,
       count(distinct preview_token) as distinct_tokens
from cozycorner.posts;
```

Expected: `total = with_token = distinct_tokens` (every row has a unique non-null token).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner add supabase/migrations/0029_posts_preview_token.sql
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner commit -m "feat(db): add posts.preview_token capability column (0029)"
```

### Task 2: Migration 0030 file — revoke anon (created now, applied in Phase 7)

**Files:**
- Create: `cozycorner/supabase/migrations/0030_posts_preview_token_revoke_anon.sql`

- [ ] **Step 1: Write the migration file (do NOT apply yet)**

```sql
-- preview_token — capability: анону выдавать нельзя (иначе можно собрать
-- превью-ссылку ОПУБЛИКОВАННОГО поста и увидеть его неопубликованные секции).
-- Черновики анону не видны и так (RLS is_published). Превью читает пост
-- service-role клиентом, которому column-grant не нужен.
--
-- ⚠️ ПОРЯДОК: применять к живой БД ТОЛЬКО ПОСЛЕ деплоя cozycorner, где анон-выборки
-- постов больше НЕ используют select("*") (иначе select * упадёт на этой колонке:
-- "permission denied for column preview_token"). См. план, Phase 7.
revoke select (preview_token) on cozycorner.posts from anon;
```

- [ ] **Step 2: Commit (file only — application is Phase 7)**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner add supabase/migrations/0030_posts_preview_token_revoke_anon.sql
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner commit -m "chore(db): add 0030 revoke anon select(preview_token) migration (apply at deploy)"
```

---

## Phase 2 — cozycorner: harden anon fetchers (deployable before revoke)

### Task 3: Replace `select("*")` on posts with an explicit column list

**Files:**
- Modify: `cozycorner/lib/types.ts` (Post type)
- Modify: `cozycorner/lib/posts.ts:14-53` (`fetchPosts`, `fetchPostBySlug`) + new constant
- Modify: `cozycorner/lib/search.ts:89-110` (`searchPosts`)
- Test: `cozycorner/lib/posts.test.ts` (new)

- [ ] **Step 1: Add `preview_token` to the cozycorner `Post` type**

In `cozycorner/lib/types.ts`, inside `export type Post = { ... }`, add after `seo_description`:

```ts
  // Capability-токен превью черновика (миграция 0029). НЕ отдаётся анон-выборкам
  // (revoke select, 0030) — присутствует только в результатах fetchPostForPreview.
  preview_token: string;
```

- [ ] **Step 2: Write the failing test** (`cozycorner/lib/posts.test.ts`)

```ts
import { describe, expect, it, vi } from 'vitest'
import { fetchPostBySlug, fetchPosts, PUBLIC_POST_COLUMNS } from './posts'
import type { DbClient } from './supabase/client'

// Мок цепочечного query-builder-а Supabase (паттерн products.test.ts): каждый
// метод возвращает тот же builder, builder — thenable (работает и `await query`,
// и `await query.order().range()` / `.maybeSingle()`).
type QueryResult = { data: unknown; error: unknown }

function makeBuilder(result: QueryResult) {
  const builder: Record<string, ReturnType<typeof vi.fn>> & { then?: unknown } = {}
  for (const m of ['select', 'eq', 'in', 'order', 'range', 'limit', 'maybeSingle']) {
    builder[m] = vi.fn(() => builder)
  }
  builder.then = (onF: (v: QueryResult) => unknown, onR?: (e: unknown) => unknown) =>
    Promise.resolve(result).then(onF, onR)
  return builder as unknown as Record<string, ReturnType<typeof vi.fn>>
}

function makeClient(resultsByTable: Record<string, QueryResult[]>) {
  const builders: { table: string; builder: Record<string, ReturnType<typeof vi.fn>> }[] = []
  const from = vi.fn((table: string) => {
    const queue = resultsByTable[table]
    if (!queue || queue.length === 0) throw new Error(`неожиданный from('${table}')`)
    const builder = makeBuilder(queue.shift() as QueryResult)
    builders.push({ table, builder })
    return builder
  })
  const client = { from } as unknown as DbClient
  const b = (n: number) => builders[n].builder
  return { client, from, b }
}

describe('anon post fetchers use an explicit column list (no select("*"))', () => {
  it('PUBLIC_POST_COLUMNS не содержит preview_token', () => {
    expect(PUBLIC_POST_COLUMNS).not.toContain('preview_token')
    expect(PUBLIC_POST_COLUMNS).not.toBe('*')
  })

  it('fetchPosts выбирает PUBLIC_POST_COLUMNS и фильтрует published/blog', async () => {
    const { client, b } = makeClient({ posts: [{ data: [], error: null }] })
    await fetchPosts(client, 0)
    expect(b(0).select).toHaveBeenCalledWith(PUBLIC_POST_COLUMNS)
    expect(b(0).eq).toHaveBeenCalledWith('is_published', true)
    expect(b(0).eq).toHaveBeenCalledWith('post_type', 'blog')
  })

  it('fetchPostBySlug выбирает PUBLIC_POST_COLUMNS + is_published', async () => {
    const { client, b } = makeClient({ posts: [{ data: null, error: null }] })
    await fetchPostBySlug(client, 'x')
    expect(b(0).select).toHaveBeenCalledWith(PUBLIC_POST_COLUMNS)
    expect(b(0).eq).toHaveBeenCalledWith('is_published', true)
  })
})
```

- [ ] **Step 3: Run it, verify it fails**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner test -- lib/posts.test.ts
```

Expected: FAIL — `PUBLIC_POST_COLUMNS` is not exported / `select` called with `"*"`.

- [ ] **Step 4: Add the constant and switch the fetchers**

In `cozycorner/lib/posts.ts`, add near the top (after imports):

```ts
// Явный список публичных колонок поста — зеркалит тип Post БЕЗ preview_token
// (анону токен не выдаётся: revoke select, миграция 0030). Заменяет select("*"),
// который упал бы на отозванной колонке. Держать в синхроне с типом Post.
export const PUBLIC_POST_COLUMNS =
  "id,created_at,published_at,is_published,slug,post_type,title,excerpt,content,cover_image_path,seo_title,seo_description";
```

In `fetchPosts`, change `.select("*")` → `.select(PUBLIC_POST_COLUMNS)`.
In `fetchPostBySlug`, change `.select("*")` → `.select(PUBLIC_POST_COLUMNS)`.

- [ ] **Step 5: Switch `searchPosts`**

In `cozycorner/lib/search.ts`, add to the existing imports from `./posts` (or a new import line):

```ts
import { PUBLIC_POST_COLUMNS } from "./posts";
```

Change `.select("*", { count: "exact" })` → `.select(PUBLIC_POST_COLUMNS, { count: "exact" })`.

- [ ] **Step 6: Run tests, verify pass**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner test -- lib/posts.test.ts lib/search.test.ts
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner add lib/types.ts lib/posts.ts lib/search.ts lib/posts.test.ts
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner commit -m "refactor(posts): explicit public column list for anon post fetchers"
```

---

## Phase 3 — cozycorner: preview data layer

### Task 4: `fetchPostForPreview` + `fetchPostSections({ includeUnpublished })`

**Files:**
- Modify: `cozycorner/lib/posts.ts` (`fetchPostSections` at :101-153; add `fetchPostForPreview`)
- Test: `cozycorner/lib/posts.test.ts`

- [ ] **Step 1: Write the failing tests** (append to `cozycorner/lib/posts.test.ts`)

```ts
import { fetchPostForPreview, fetchPostSections } from './posts'

describe('fetchPostForPreview', () => {
  const row = (over = {}) => ({
    id: 'p1', slug: 'my-post', is_published: false, title: 'T', excerpt: null,
    cover_image_path: null, preview_token: 'good-token', ...over,
  })

  it('токен совпал → возвращает пост (без фильтра is_published)', async () => {
    const { client, b } = makeClient({ posts: [{ data: row(), error: null }] })
    const result = await fetchPostForPreview(client, 'my-post', 'good-token')
    expect(result).not.toBeNull()
    expect(result!.id).toBe('p1')
    expect(b(0).select).toHaveBeenCalledWith(`${PUBLIC_POST_COLUMNS},preview_token`)
    expect(b(0).eq).toHaveBeenCalledWith('slug', 'my-post')
    // Превью НЕ фильтрует по is_published (черновик должен находиться).
    expect(b(0).eq).not.toHaveBeenCalledWith('is_published', true)
  })

  it('токен не совпал → null', async () => {
    const { client } = makeClient({ posts: [{ data: row(), error: null }] })
    await expect(fetchPostForPreview(client, 'my-post', 'wrong')).resolves.toBeNull()
  })

  it('пост не найден → null', async () => {
    const { client } = makeClient({ posts: [{ data: null, error: null }] })
    await expect(fetchPostForPreview(client, 'nope', 'good-token')).resolves.toBeNull()
  })
})

describe('fetchPostSections includeUnpublished', () => {
  const empty = () => ({
    post_text_sections: [{ data: [], error: null }],
    post_product_sections: [{ data: [], error: null }],
  })

  it('по умолчанию фильтрует is_published=true на обеих таблицах', async () => {
    const { client, b } = makeClient(empty())
    await fetchPostSections(client, 'p1')
    expect(b(0).eq).toHaveBeenCalledWith('is_published', true)
    expect(b(1).eq).toHaveBeenCalledWith('is_published', true)
  })

  it('includeUnpublished:true → НЕ фильтрует is_published (все секции)', async () => {
    const { client, b } = makeClient(empty())
    await fetchPostSections(client, 'p1', { includeUnpublished: true })
    expect(b(0).eq).not.toHaveBeenCalledWith('is_published', true)
    expect(b(1).eq).not.toHaveBeenCalledWith('is_published', true)
  })
})
```

- [ ] **Step 2: Run, verify it fails**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner test -- lib/posts.test.ts
```

Expected: FAIL — `fetchPostForPreview` not exported; `fetchPostSections` ignores the option.

- [ ] **Step 3: Add `fetchPostForPreview`** (in `cozycorner/lib/posts.ts`, after `fetchPostBySlug`)

```ts
/**
 * Пост для превью-роута: БЕЗ фильтра is_published, читается service-role клиентом.
 * Возвращает пост ТОЛЬКО при совпадении preview_token (capability), иначе null —
 * невалидный/чужой токен ведёт себя как «не найдено» (роут делает notFound()).
 * Не для анон-клиента: preview_token анону не выдаётся (миграция 0030).
 */
export async function fetchPostForPreview(
  client: DbClient,
  slug: string,
  token: string,
): Promise<Post | null> {
  const { data, error } = await client
    .from("posts")
    .select(`${PUBLIC_POST_COLUMNS},preview_token`)
    .eq("slug", slug)
    .maybeSingle();
  if (error) throw error;
  const post = (data ?? null) as Post | null;
  if (!post || post.preview_token !== token) return null;
  return post;
}
```

- [ ] **Step 4: Extend `fetchPostSections` with an options arg** (preserving current default)

Replace the signature + query construction at the top of `fetchPostSections` so that the `is_published` filter is applied only when `includeUnpublished` is false (default). Keep the product-hydration and merge logic below unchanged:

```ts
/** Секции поста, слитые в один список по position.
 *  По умолчанию — только опубликованные (живая страница). includeUnpublished:true —
 *  все секции (превью черновика показывает и неопубликованные секции). */
export async function fetchPostSections(
  client: DbClient,
  postId: string,
  opts: { includeUnpublished?: boolean } = {},
): Promise<PostSection[]> {
  let textQuery = client
    .from("post_text_sections")
    .select("id, position, heading, body")
    .eq("post_id", postId);
  let productQuery = client
    .from("post_product_sections")
    .select("id, position, heading, columns, product_ids")
    .eq("post_id", postId);
  if (!opts.includeUnpublished) {
    textQuery = textQuery.eq("is_published", true);
    productQuery = productQuery.eq("is_published", true);
  }
  const [textRes, productRes] = await Promise.all([
    textQuery.order("position", { ascending: true }),
    productQuery.order("position", { ascending: true }),
  ]);
  // ↓↓↓ остальное тело функции (проверка error, hydration товаров, merge, sort)
  //     оставить БЕЗ изменений.
```

Leave everything from `if (textRes.error) throw textRes.error;` onward exactly as it is today. The existing live call site `fetchPostSections(supabase, post.id)` in `app/blog/[slug]/page.tsx` keeps its behavior (default filters published).

- [ ] **Step 5: Run, verify pass**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner test -- lib/posts.test.ts
```

Expected: PASS (all posts.test.ts describes green).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner add lib/posts.ts lib/posts.test.ts
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner commit -m "feat(posts): fetchPostForPreview + includeUnpublished sections"
```

---

## Phase 4 — cozycorner: preview route

### Task 5: `app/preview/blog/[slug]/page.tsx`

**Files:**
- Create: `cozycorner/app/preview/blog/[slug]/page.tsx`

- [ ] **Step 1: Read the Next 16 route docs first** (project rule — this is a real Next 16, APIs differ)

```bash
ls /Users/mdr/Documents/Projects/workspace-one/cozycorner/node_modules/next/dist/docs/01-app
```

Read the guides relevant to this route before writing it: route segment config (`dynamic`), dynamic route params, `searchParams`, and the `metadata`/`robots` API. Confirm that in this version `params` and `searchParams` are async (Promises) — the sibling `app/blog/[slug]/page.tsx` already awaits `params`, so mirror that.

- [ ] **Step 2: Write the route**

```tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import { createAdminClient } from "@/lib/supabase/admin";
import { fetchPostForPreview, fetchPostSections, fetchPosts } from "@/lib/posts";
import Hero, { type HeroContent } from "@/components/Hero";
import PostTextSection from "@/components/PostTextSection";
import PostProductsSection from "@/components/PostProductsSection";
import RecommendedPosts from "@/components/RecommendedPosts";
import styles from "@/app/blog/[slug]/page.module.css";

// Превью черновика на реальном фронте: динамический (не ISR, не в
// generateStaticParams), не индексируется. Читает пост service-role клиентом
// ТОЛЬКО при совпадении preview_token. Сетка /blog, ISR и generateStaticParams
// не затрагиваются — превью живёт в отдельном сегменте /preview.
export const dynamic = "force-dynamic";

// noindex: даже если токенизированная ссылка утечёт краулеру — не индексировать.
export const metadata: Metadata = {
  robots: { index: false, follow: false },
};

export default async function PreviewPostPage({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ token?: string }>;
}) {
  const { slug } = await params;
  const { token } = await searchParams;
  if (!token) notFound();

  const admin = createAdminClient();
  const post = await fetchPostForPreview(admin, slug, token);
  if (!post) notFound();

  // Превью показывает ВСЕ секции (в т.ч. per-section is_published=false); свежие
  // посты для блока рекомендаций читаются тем же клиентом (fetchPosts фильтрует
  // published сам). Живая /blog/[slug] секции по-прежнему фильтрует.
  const [sections, latestPosts] = await Promise.all([
    fetchPostSections(admin, post.id, { includeUnpublished: true }),
    fetchPosts(admin, 0, 4),
  ]);
  const recommended = latestPosts.filter((p) => p.id !== post.id).slice(0, 3);

  const hero: HeroContent = {
    badge: "Post",
    title: post.title,
    description: post.excerpt,
    bg_image_path: post.cover_image_path,
    align: "left",
  };

  return (
    <main>
      <Hero hero={hero} />
      <div className={styles.content}>
        {sections.map((section) =>
          section.kind === "text" ? (
            <div key={section.id} className={styles.narrow}>
              <PostTextSection section={section} />
            </div>
          ) : (
            <PostProductsSection key={section.id} section={section} />
          ),
        )}
        <RecommendedPosts posts={recommended} />
      </div>
    </main>
  );
}
```

Notes for the implementer:
- The CSS module is reused from the live route via the `@/` alias (maps to repo root). If the alias does not resolve for a nested `app/` path, use a relative import `../../../blog/[slug]/page.module.css` instead — do not duplicate the stylesheet.
- `createAdminClient()` returns the same supabase-js client shape as `DbClient` (both `schema: "cozycorner"`, no `Database` generic), so passing it to `fetchPosts`/`fetchPostSections`/`fetchPostForPreview` type-checks. If TS objects, that is a signal something changed — do not paper over it with `any`; investigate.
- `robots.ts` needs **no change**: the preview URL is never linked or in the sitemap, and carries `noindex`. Adding a `Disallow: /preview` would hide the `noindex` from crawlers (same reasoning as the existing `/search` comment in `robots.ts`), so do not add one.

- [ ] **Step 3: Build (TS + prerender) and lint**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner run build
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner run lint
```

Expected: build succeeds; the preview route is reported as dynamic (ƒ), not statically prerendered. Lint clean.

> Async RSC pages are not rendered by Vitest in this repo (see platform-docs/methodology/testing.md), so this route has no unit test. Its behavior (renders draft with valid token; 404 on wrong/absent token; absent from grid) is covered by the build and by the **proposed** e2e in Phase 6 notes.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner add app/preview/blog/[slug]/page.tsx
git -C /Users/mdr/Documents/Projects/workspace-one/cozycorner commit -m "feat(preview): noindex dynamic /preview/blog/[slug] token-gated route"
```

---

## Phase 5 — web.admin: config + data + UI

### Task 6: Add `frontendUrl` to `SiteConfig`

**Files:**
- Modify: `web.admin/src/config/sites.ts`

- [ ] **Step 1: Add the field to the type and the cozycorner value**

In `web.admin/src/config/sites.ts`, add to `SiteConfig`:

```ts
  // Публичный базовый URL сайта (для ссылок на реальный фронт, напр. превью поста).
  // Необязателен: если не задан — превью-ссылка в редакторе скрыта.
  // ⚠️ Обновить при переезде на кастомный домен.
  frontendUrl?: string;
```

And in the cozycorner entry, add:

```ts
    frontendUrl: "https://cozycorner-omega.vercel.app",
```

- [ ] **Step 2: Type-check**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run build
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin add src/config/sites.ts
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin commit -m "feat(config): add SiteConfig.frontendUrl (cozycorner prod URL)"
```

### Task 7: Read `preview_token` in the admin post layer

**Files:**
- Modify: `web.admin/src/lib/posts.ts:10-31,56-57` (Post type, PostInput, POST_COLUMNS)

- [ ] **Step 1: Add `preview_token` to the admin `Post` type**

In `web.admin/src/lib/posts.ts`, inside `export type Post = { ... }`, add after `seo_description`:

```ts
  // Capability-токен превью (миграция 0029). Админка его только ЧИТАЕТ (строит
  // превью-ссылку); никогда не пишет. Service-role в web.admin нет.
  preview_token: string;
```

- [ ] **Step 2: Exclude `preview_token` from `PostInput`** (never written from admin)

Change:

```ts
export type PostInput = Omit<Post, "id" | "created_at" | "updated_at" | "slug">;
```

to:

```ts
export type PostInput = Omit<Post, "id" | "created_at" | "updated_at" | "slug" | "preview_token">;
```

- [ ] **Step 3: Add the column to `POST_COLUMNS`**

Change the `POST_COLUMNS` string to append `,preview_token`:

```ts
const POST_COLUMNS =
  "id,created_at,updated_at,published_at,is_published,post_type,slug,title,excerpt,cover_image_path,seo_title,seo_description,preview_token";
```

(`getPost` already selects `POST_COLUMNS`, so it now returns `preview_token`. `toInput` in `PostEditPage.tsx` builds `PostInput` field-by-field and never sets `preview_token`, so create/update stay correct.)

- [ ] **Step 4: Type-check**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run build
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin add src/lib/posts.ts
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin commit -m "feat(posts): read preview_token in admin post layer"
```

### Task 8: `CopyPreviewLinkButton` + wire into the editor

**Files:**
- Create: `web.admin/src/features/posts/CopyPreviewLinkButton.tsx`
- Test: `web.admin/src/features/posts/CopyPreviewLinkButton.test.tsx`
- Modify: `web.admin/src/features/posts/PostEditPage.tsx` (imports + sticky panel ~line 349-366)

- [ ] **Step 1: Write the failing test** (`CopyPreviewLinkButton.test.tsx`)

```tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import type { SiteConfig } from '@/config/sites'
import { CopyPreviewLinkButton } from './CopyPreviewLinkButton'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
import { toast } from 'sonner'

const baseSite: SiteConfig = {
  slug: 'cozycorner', label: 'CozyCorner', projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test', schema: 'cozycorner', bucket: 'cozycorner-photos',
  frontendUrl: 'https://cozycorner-omega.vercel.app',
}

beforeEach(() => {
  vi.clearAllMocks()
  Object.assign(navigator, {
    clipboard: { writeText: vi.fn().mockResolvedValue(undefined) },
  })
})

describe('<CopyPreviewLinkButton />', () => {
  it('копирует URL из frontendUrl + slug + token и тостит', async () => {
    render(<CopyPreviewLinkButton site={baseSite} slug="my-post" previewToken="tok-123" />)
    fireEvent.click(screen.getByRole('button', { name: /Copy preview link/ }))
    await waitFor(() =>
      expect(navigator.clipboard.writeText).toHaveBeenCalledWith(
        'https://cozycorner-omega.vercel.app/preview/blog/my-post?token=tok-123',
      ),
    )
    expect(toast.success).toHaveBeenCalledWith('Preview link copied')
  })

  it('скрыт, если у сайта нет frontendUrl', () => {
    render(
      <CopyPreviewLinkButton
        site={{ ...baseSite, frontendUrl: undefined }}
        slug="my-post"
        previewToken="tok-123"
      />,
    )
    expect(screen.queryByRole('button', { name: /Copy preview link/ })).toBeNull()
  })

  it('скрыт для несохранённого поста (нет slug/token)', () => {
    render(<CopyPreviewLinkButton site={baseSite} slug={undefined} previewToken={undefined} />)
    expect(screen.queryByRole('button', { name: /Copy preview link/ })).toBeNull()
  })
})
```

- [ ] **Step 2: Run, verify it fails**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin test -- CopyPreviewLinkButton
```

Expected: FAIL — module does not exist.

- [ ] **Step 3: Write the component** (`CopyPreviewLinkButton.tsx`)

```tsx
import { Eye } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { Button } from '@/components/ui/button'

// Строит ссылку на превью поста на реальном фронте и копирует её в буфер обмена.
// Скрыт, если у сайта не задан frontendUrl или у поста ещё нет slug/preview_token
// (несохранённый пост). Никакого service-role/подписи — только чтение значения.
export function CopyPreviewLinkButton({
  site,
  slug,
  previewToken,
}: {
  site: SiteConfig
  slug: string | undefined
  previewToken: string | undefined
}) {
  if (!site.frontendUrl || !slug || !previewToken) return null
  const url = `${site.frontendUrl}/preview/blog/${slug}?token=${previewToken}`
  async function onCopy() {
    try {
      await navigator.clipboard.writeText(url)
      toast.success('Preview link copied')
    } catch {
      toast.error('Could not copy link')
    }
  }
  return (
    <Button type="button" variant="outline" onClick={onCopy}>
      <Eye />
      Copy preview link
    </Button>
  )
}
```

- [ ] **Step 4: Run, verify pass**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin test -- CopyPreviewLinkButton
```

Expected: PASS (3 tests).

- [ ] **Step 5: Wire into the editor sticky panel** (`PostEditPage.tsx`)

Add to the existing lucide import (line 13) — keep it alphabetical-ish, just add `Eye` is inside the child component, so no import change needed in PostEditPage. Add the component import after the other feature imports (near line 31):

```tsx
import { CopyPreviewLinkButton } from './CopyPreviewLinkButton'
```

In the sticky action panel, inside `<div className="ml-auto flex items-center gap-3">` (line 349), add the button **before** the Save `<Button>` (line 364):

```tsx
          <CopyPreviewLinkButton
            site={site}
            slug={data?.post.slug}
            previewToken={data?.post.preview_token}
          />
```

(For a new/unsaved post `data` is null → `slug`/`previewToken` undefined → the component renders nothing. For a saved post it renders whenever `site.frontendUrl` is set.)

- [ ] **Step 6: Build, lint, full test run**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run build
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run lint
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin test
```

Expected: build clean; lint clean (2 known shadcn warnings OK); all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin add src/features/posts/CopyPreviewLinkButton.tsx src/features/posts/CopyPreviewLinkButton.test.tsx src/features/posts/PostEditPage.tsx
git -C /Users/mdr/Documents/Projects/workspace-one/web.admin commit -m "feat(posts): Copy preview link button in post editor"
```

---

## Phase 6 — Documentation (platform-docs)

### Task 9: Update schema.md, blog.md, cozycorner.md

**Files:**
- Modify: `platform-docs/database/schema.md` (§3 RLS ~line 60, §5 posts table ~line 173, §6 history ~line 303)
- Modify: `platform-docs/admin-panel/blog.md`
- Modify: `platform-docs/sites/cozycorner.md`

- [ ] **Step 1: schema.md — posts table row** (add after the `folder_id` row in the `### posts` table, ~line 173):

```markdown
| `preview_token` | uuid (not null, default `gen_random_uuid()`) | capability-токен превью черновика (0029). НЕ флаг видимости. Анону НЕ выдаётся (`revoke select`, 0030) — читается только service-role клиентом на роуте `/preview/blog/[slug]` |
```

- [ ] **Step 2: schema.md — RLS note** (append to the `posts` bullet in §3, ~line 60): after "анону только `is_published = true`" add:

```markdown
 . Колонка `preview_token` дополнительно отозвана у `anon` (`revoke select`, 0030): это capability для превью, доступ к ней — только у service-role. Поэтому анон-выборки постов используют явный список колонок (`PUBLIC_POST_COLUMNS`), а не `select("*")`.
```

- [ ] **Step 3: schema.md — migration history** (extend the line ending at `0028 drop products.category …`, ~line 303):

```markdown
0029 posts.preview_token (capability-токен превью черновика; default gen_random_uuid, бэкфилл автоматом) ·
0030 revoke select(preview_token) от anon (токен читает только service-role; применена ПОСЛЕ деплоя явного списка колонок в анон-выборках).
```

- [ ] **Step 4: blog.md — new section** (append after §5, a new `## 6. Превью поста (preview link)`):

```markdown
## 6. Превью поста (preview link, 2026-07-27)

Редактор может открыть черновик на реальном фронте cozycorner по спец-ссылке, не
публикуя его в ленту. Спека — `../superpowers/specs/2026-07-27-blog-post-preview-design.md`.

- **Токен** — `posts.preview_token` (uuid, миграция 0029): capability, НЕ флаг
  видимости. Админка его только ЧИТАЕТ (`getPost` → `POST_COLUMNS` включает
  `preview_token`; `PostInput` его исключает — админка токен не пишет). Service-role
  ключа в web.admin нет.
- **Кнопка** «Copy preview link» — `src/features/posts/CopyPreviewLinkButton.tsx`, в
  sticky-панели редактора. Строит `${site.frontendUrl}/preview/blog/${slug}?token=${preview_token}`
  и копирует в буфер. Скрыта, если у сайта не задан `frontendUrl`
  (`src/config/sites.ts`) или пост ещё не сохранён (нет slug/token).
- **Фронт** — cozycorner роут `app/preview/blog/[slug]/page.tsx` (force-dynamic,
  noindex): читает пост service-role клиентом ТОЛЬКО при совпадении токена, показывает
  ВСЕ секции (игнорирует per-section `is_published`). Сетка/ISR/`generateStaticParams`
  не затрагиваются. Детали — `../sites/cozycorner.md`.
- **Отзыв ссылки** (регенерация токена) — отложено; колонка есть, добавляется без
  миграции.
```

- [ ] **Step 5: cozycorner.md — document the route.** Read the routes/SEO section of `platform-docs/sites/cozycorner.md` and add an entry describing:

```markdown
- `/preview/blog/[slug]?token=<uuid>` — превью черновика поста. `force-dynamic`,
  `noindex` (robots meta), не в sitemap, не в `generateStaticParams`. Читает пост
  **service-role** клиентом (`lib/supabase/admin.ts`) через `fetchPostForPreview`
  ТОЛЬКО при совпадении `posts.preview_token`; секции — `fetchPostSections(…, { includeUnpublished: true })`
  (показывает и неопубликованные секции). Невалидный/отсутствующий токен → `notFound()`.
  Ссылку генерирует web.admin (кнопка «Copy preview link»).
```

Place it alongside the other blog route entries; match the surrounding formatting.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdr/Documents/Projects/workspace-one/platform-docs add database/schema.md admin-panel/blog.md sites/cozycorner.md
git -C /Users/mdr/Documents/Projects/workspace-one/platform-docs commit -m "docs: blog post preview (preview_token, /preview route, copy link)"
```

---

## Phase 7 — Deploy + revoke (REQUIRES USER CONFIRMATION)

> Do not run this phase without explicit user approval. It pushes branches, triggers the cozycorner Vercel deploy, and applies the anon revoke to the live DB. Order matters (see the sequencing constraint at the top).

### Task 10: Ship

- [ ] **Step 1: Confirm with the user** that they want to push + deploy + apply the revoke now. If merging to `main` is required for the Vercel deploy, confirm the merge strategy for cozycorner and web.admin.

- [ ] **Step 2: Final green check across repos**

```bash
npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner run build && npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner run lint && npm --prefix /Users/mdr/Documents/Projects/workspace-one/cozycorner test
npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run build && npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin run lint && npm --prefix /Users/mdr/Documents/Projects/workspace-one/web.admin test
```

Expected: all green.

- [ ] **Step 3: Push + deploy cozycorner** (per confirmed strategy). Wait for the Vercel deploy to go live. The deployed code no longer does `select("*")` on posts, so the revoke is now safe.

- [ ] **Step 4: Apply migration 0030 to the live DB** via `mcp__supabase__apply_migration` (`project_id: "zwrkphynupdubevzwdzy"`, `name: "0030_posts_preview_token_revoke_anon"`, SQL from Task 2).

- [ ] **Step 5: Verify anon can no longer read the token, but the site still works**

```sql
-- as anon: token column must be denied; other columns fine
set local role anon;
select preview_token from cozycorner.posts limit 1;  -- expect: permission denied for column preview_token
reset role;
```

Then load prod `/blog`, a published `/blog/<slug>`, and `/search?q=...` — all must render (no column-permission errors). Load a draft's `/preview/blog/<slug>?token=<uuid>` copied from the admin — it must render the draft; changing the token must give a 404.

- [ ] **Step 6: Push web.admin** (per confirmed strategy).

---

## Proposed follow-ups (not in this iteration)

- **e2e (cozycorner, :3000, live Supabase)** — seed a draft with a known token; assert: preview URL renders the draft; wrong token → 404; draft absent from `/blog`. Deferred per the testing protocol (needs live seeding); propose in the completion report.
- **Regenerate preview link** (revoke a shared link by rotating `preview_token`) — deferred; column exists, no migration needed.
- **SEO-post preview** — same mechanism would extend; out of scope (blog only).

## Self-review notes (author)

- Spec coverage: DB column ✓ (T1), anon token protection ✓ (T2 file + T3 explicit columns + T10 revoke), preview route ✓ (T5), preview fetchers incl. all-sections ✓ (T4), noindex/sitemap ✓ (T5 metadata; sitemap already excludes /preview), web.admin frontendUrl ✓ (T6), admin reads token ✓ (T7), copy-link UI ✓ (T8), docs ✓ (T9), tests ✓ (T3/T4/T8), migration-before-frontend verified ✓ (T1 step 3).
- Type consistency: `PUBLIC_POST_COLUMNS`, `fetchPostForPreview(client,slug,token)`, `fetchPostSections(client,postId,{includeUnpublished})`, `Post.preview_token`, `SiteConfig.frontendUrl`, `CopyPreviewLinkButton({site,slug,previewToken})` used identically across tasks.
- Extra scope beyond the spec, justified: switching the 3 anon `select("*")` calls to `PUBLIC_POST_COLUMNS` is required by the chosen "revoke from anon" decision (Supabase errors on `*` over a revoked column). Encoded as its own deployable task (T3) with the deploy-ordering constraint (T2/T10).
