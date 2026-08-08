# Avocado Kiss — Blog (Article) editor в web.admin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Построить в общей админке `web.admin` раздел Blog для сайта Avocado Kiss (3 шаблона поста + единая таблица секций из 6 блоков + теги/авторы/Read-also/баннер архива/папки).

**Architecture:** Отдельная feature-папка `src/features/articles/` + слой данных `src/lib/articles.ts`; UI-механику (конструктор блоков, diff-сохранение, dirty-guard, image/FK-пикеры, SEO customize/reset) портируем из `src/features/posts/` (блог CozyCorner), модель данных берём из контракта `platform-docs/admin-panel/blog-avocado-kiss.md`. Маршрут `/:siteSlug/blog` диспетчеризуется по `site.schema`, чтобы CozyCorner остался на `PostsPage`, а Avocado получил `ArticlesPage`.

**Tech Stack:** Vite + React 19 + TS, Tailwind v4 + shadcn/ui, react-router-dom (data router), TanStack Query, React Hook Form + Zod v4, sonner, TipTap; `@supabase/supabase-js` (anon key, RLS + `is_admin()`); тесты Vitest + RTL.

**Spec:** [../specs/2026-08-08-blog-admin-design.md](../specs/2026-08-08-blog-admin-design.md)
**Контракт формы:** [../../../admin-panel/blog-avocado-kiss.md](../../../admin-panel/blog-avocado-kiss.md)

---

## Соглашения для всех задач

- **UI-текст — английский**; комментарии/коммиты — русский ок (AGENTS.md web.admin).
- Пустые optional → `null` (`orNull`); `slug`/`updated_at` — триггеры БД, не пишем;
  картинки — `resolveImageUrl`/`toStoragePath`; SEO живой фолбэк (`seoOrNull`).
- Все запросы через `getDb(site).schema(...)`; явные списки колонок, не `select('*')`.
- Тесты рядом с кодом (`*.test.ts[x]`), запуск `npm test` (Vitest). Проверка перед
  финишем: `npm run build` + `npm run lint` (oxlint; 2 известных shadcn-варнинга ок).
- **Не коммитить без явного разрешения пользователя.** Шаги «Commit» ниже —
  подготовленные сообщения; исполнять только когда пользователь разрешит (или
  собирать в один коммит на ревью — по выбранному режиму исполнения).
- Работать **только в одном проекте за задачу**: Task 0 — репо `avocado.kiss`; все
  остальные — репо `web.admin`.

---

## Task 0: (репо avocado.kiss) Миграция — `posts.folder_id` + секция `posts` в admin_folders

Предусловие для папок (Task 6). Делается в репозитории **avocado.kiss**, не в web.admin.
Схему меняем миграцией (гард: админка схему не трогает).

**Files:**
- Create: `avocado.kiss/supabase/migrations/0015_posts_folders.sql` (проверь актуальный
  максимальный номер миграции в каталоге — следующий по порядку; здесь 0015, т.к.
  последняя по schema.md — 0014).

- [ ] **Step 1: Проверить текущее состояние**

Run: `ls avocado.kiss/supabase/migrations/ | sort | tail -3`
Expected: последняя миграция `0014_*` (drop products.category). Новый файл — `0015`.

- [ ] **Step 2: Написать миграцию**

```sql
-- 0015_posts_folders.sql
-- Папки админки для блога Avocado: posts.folder_id + секция 'posts' в admin_folders.
-- Аддитивно; по образцу cozycorner 0022 (posts.folder_id) и avocado 0013 (секция products).

alter table avocado_kiss.posts
  add column if not exists folder_id uuid
    references avocado_kiss.admin_folders(id) on delete set null;

create index if not exists posts_folder_id_idx
  on avocado_kiss.posts (folder_id);

alter table avocado_kiss.admin_folders
  drop constraint if exists admin_folders_section_check;

alter table avocado_kiss.admin_folders
  add constraint admin_folders_section_check
    check (section in ('media', 'recipes', 'products', 'posts'));
```

- [ ] **Step 3: Применить локально/через supabase CLI и проверить**

Run: `supabase db push` (или применить через MCP `apply_migration` на проект base-one).
Проверка: `select column_name from information_schema.columns where table_schema='avocado_kiss' and table_name='posts' and column_name='folder_id';` → 1 строка.
Проверка CHECK: `insert into avocado_kiss.admin_folders(section,name) values('posts','__test__');` → успех; затем `delete ... where name='__test__';`.

- [ ] **Step 4: Обновить `platform-docs/database/schema.md`**

В секции avocado `admin_folders` заменить `check ('media' | 'recipes' | 'products')`
на `('media' | 'recipes' | 'products' | 'posts')`; в описании `posts` добавить строку
`folder_id`; в §10 «История миграций» добавить `· 0015 posts.folder_id + секция posts
в admin_folders (папки блога в web.admin)`.

- [ ] **Step 5: Commit (в репо avocado.kiss, с разрешения)**

```bash
cd avocado.kiss && git add supabase/migrations/0015_posts_folders.sql
git commit -m "feat(db): posts.folder_id + posts section in admin_folders (blog folders)"
```

---

## Task 1: Слой данных — `src/lib/articles.ts` (типы + чтение)

**Files:**
- Create: `web.admin/src/lib/articles.ts`
- Test: `web.admin/src/lib/articles.test.ts`

- [ ] **Step 1: Свериться с живой схемой перед написанием (golden rule)**

Проверь точные имена колонок `avocado_kiss.post_sections`, `posts`, `authors`,
`post_related` — через коннектор или в `avocado.kiss/supabase/migrations/0008_blog.sql`
и `0009_blog_more_templates.sql`. Ожидаемые (из контракта): `post_sections(id, post_id,
position, is_published, type, body, text_variant, quote, quote_attribution, image_path,
caption, credit, recipe_id, card_eyebrow, question, answer, rank, heading)`; `posts(...,
template, author_id, subtitle, read_minutes, hero_image_path, hero_caption, excerpt,
seo_title, seo_description, is_published, published_at, title, slug, folder_id)`;
`authors(id, name, slug, avatar_path, bio)`; `post_related(id, post_id, position,
recipe_id, related_post_id)`. **Если имена отличаются — правь код по БД и отметь в отчёте.**

- [ ] **Step 2: Написать типы и read-функции**

```ts
// web.admin/src/lib/articles.ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

export type ArticleTemplate = "essay" | "interview" | "roundup";

// Строка posts (схема avocado_kiss). slug/updated_at — триггеры БД. Автор — join.
export type Article = {
  id: string;
  created_at: string;
  updated_at: string;
  published_at: string;
  is_published: boolean;
  slug: string;
  title: string;
  template: ArticleTemplate;
  author_id: string | null;
  excerpt: string | null;
  subtitle: string | null;
  read_minutes: number | null;
  hero_image_path: string | null;
  hero_caption: string | null;
  seo_title: string | null;
  seo_description: string | null;
};

export type ArticleListItem = Pick<
  Article,
  "id" | "title" | "template" | "published_at" | "is_published"
> & { folder_id: string | null };

// Payload create/update: без служебных и slug (генерирует триггер).
export type ArticleInput = Omit<
  Article,
  "id" | "created_at" | "updated_at" | "slug"
>;

export type Author = {
  id: string;
  name: string;
  slug: string;
  avatar_path: string | null;
  bio: string | null;
};

// Дискриминированный union блока тела. sectionId отсутствует у новых (несохранённых).
export type ArticleSection =
  | { kind: "text"; sectionId?: string; isPublished: boolean; body: string; variant: "lead" | "body" }
  | { kind: "quote"; sectionId?: string; isPublished: boolean; quote: string; attribution: string | null }
  | { kind: "image"; sectionId?: string; isPublished: boolean; imagePath: string | null; caption: string | null; credit: string | null }
  | { kind: "recipe_card"; sectionId?: string; isPublished: boolean; recipeId: string | null; eyebrow: string | null }
  | { kind: "qa"; sectionId?: string; isPublished: boolean; question: string; answer: string }
  | { kind: "list_item"; sectionId?: string; isPublished: boolean; rank: number | null; heading: string; body: string | null; recipeId: string | null; eyebrow: string | null };

// Ручной пин Read also: ровно одна ссылка (recipe XOR post).
export type RelatedPin =
  | { kind: "recipe"; id: string }
  | { kind: "post"; id: string };

export type ArticleWithRelations = {
  article: Article;
  author: Author | null;
  sections: ArticleSection[];
  tagIds: string[]; // порядок = post_tags.position (= порядок эйброу)
  related: RelatedPin[]; // порядок = post_related.position
};

const ARTICLE_COLUMNS =
  "id,created_at,updated_at,published_at,is_published,slug,title,template,author_id,excerpt,subtitle,read_minutes,hero_image_path,hero_caption,seo_title,seo_description";

const SECTION_COLUMNS =
  "id,position,is_published,type,body,text_variant,quote,quote_attribution,image_path,caption,credit,recipe_id,card_eyebrow,question,answer,rank,heading";

export async function listArticles(site: SiteConfig): Promise<ArticleListItem[]> {
  const { data, error } = await getDb(site)
    .from("posts")
    .select("id,title,template,published_at,is_published,folder_id")
    .order("published_at", { ascending: false })
    .order("title", { ascending: true });
  if (error) throw error;
  return (data ?? []) as ArticleListItem[];
}

export async function listAuthors(site: SiteConfig): Promise<Author[]> {
  const { data, error } = await getDb(site)
    .from("authors")
    .select("id,name,slug,avatar_path,bio")
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Author[];
}

type SectionRow = {
  id: string;
  position: number;
  is_published: boolean;
  type: ArticleSection["kind"];
  body: string | null;
  text_variant: "lead" | "body" | null;
  quote: string | null;
  quote_attribution: string | null;
  image_path: string | null;
  caption: string | null;
  credit: string | null;
  recipe_id: string | null;
  card_eyebrow: string | null;
  question: string | null;
  answer: string | null;
  rank: number | null;
  heading: string | null;
};

// БД-строка post_sections → ArticleSection (по type).
export function rowToSection(r: SectionRow): ArticleSection {
  const base = { sectionId: r.id, isPublished: r.is_published };
  switch (r.type) {
    case "text":
      return { ...base, kind: "text", body: r.body ?? "", variant: r.text_variant ?? "body" };
    case "quote":
      return { ...base, kind: "quote", quote: r.quote ?? "", attribution: r.quote_attribution };
    case "image":
      return { ...base, kind: "image", imagePath: r.image_path, caption: r.caption, credit: r.credit };
    case "recipe_card":
      return { ...base, kind: "recipe_card", recipeId: r.recipe_id, eyebrow: r.card_eyebrow };
    case "qa":
      return { ...base, kind: "qa", question: r.question ?? "", answer: r.answer ?? "" };
    case "list_item":
      return { ...base, kind: "list_item", rank: r.rank, heading: r.heading ?? "", body: r.body, recipeId: r.recipe_id, eyebrow: r.card_eyebrow };
  }
}

export async function getArticle(
  site: SiteConfig,
  id: string,
): Promise<ArticleWithRelations | null> {
  const db = getDb(site);
  const [postRes, secRes, tagRes, relRes] = await Promise.all([
    db.from("posts").select(ARTICLE_COLUMNS).eq("id", id).maybeSingle(),
    db.from("post_sections").select(SECTION_COLUMNS).eq("post_id", id).order("position"),
    db.from("post_tags").select("tag_id,position").eq("post_id", id).order("position"),
    db.from("post_related").select("recipe_id,related_post_id,position").eq("post_id", id).order("position"),
  ]);
  if (postRes.error) throw postRes.error;
  if (secRes.error) throw secRes.error;
  if (tagRes.error) throw tagRes.error;
  if (relRes.error) throw relRes.error;
  if (!postRes.data) return null;

  const article = postRes.data as Article;
  let author: Author | null = null;
  if (article.author_id) {
    const { data: a, error } = await db
      .from("authors")
      .select("id,name,slug,avatar_path,bio")
      .eq("id", article.author_id)
      .maybeSingle();
    if (error) throw error;
    author = (a as Author) ?? null;
  }

  const sections = ((secRes.data ?? []) as SectionRow[]).map(rowToSection);
  const tagIds = ((tagRes.data ?? []) as { tag_id: string }[]).map((r) => r.tag_id);
  const related: RelatedPin[] = ((relRes.data ?? []) as {
    recipe_id: string | null;
    related_post_id: string | null;
  }[]).map((r) =>
    r.recipe_id
      ? { kind: "recipe", id: r.recipe_id }
      : { kind: "post", id: r.related_post_id as string },
  );

  return { article, author, sections, tagIds, related };
}
```

- [ ] **Step 3: Написать unit-тест на `rowToSection`**

```ts
// web.admin/src/lib/articles.test.ts
import { describe, expect, it } from "vitest";
import { rowToSection } from "./articles";

const base = {
  id: "s1", position: 0, is_published: true,
  body: null, text_variant: null, quote: null, quote_attribution: null,
  image_path: null, caption: null, credit: null, recipe_id: null,
  card_eyebrow: null, question: null, answer: null, rank: null, heading: null,
};

describe("rowToSection", () => {
  it("maps text with default variant 'body'", () => {
    const s = rowToSection({ ...base, type: "text", body: "Hello" });
    expect(s).toEqual({ kind: "text", sectionId: "s1", isPublished: true, body: "Hello", variant: "body" });
  });
  it("maps list_item preserving rank and optional recipe", () => {
    const s = rowToSection({ ...base, type: "list_item", rank: 3, heading: "Pie", recipe_id: "r1", card_eyebrow: "Recipe" });
    expect(s).toEqual({ kind: "list_item", sectionId: "s1", isPublished: true, rank: 3, heading: "Pie", body: null, recipeId: "r1", eyebrow: "Recipe" });
  });
  it("maps qa", () => {
    const s = rowToSection({ ...base, type: "qa", question: "Q?", answer: "A." });
    expect(s).toMatchObject({ kind: "qa", question: "Q?", answer: "A." });
  });
});
```

- [ ] **Step 4: Прогнать тест**

Run: `npm --prefix web.admin test -- articles.test`
Expected: PASS (3 теста).

- [ ] **Step 5: Commit**

```bash
git add web.admin/src/lib/articles.ts web.admin/src/lib/articles.test.ts
git commit -m "feat(articles): data-layer types + read (getArticle/listArticles/rowToSection)"
```

---

## Task 2: Слой данных — запись (create/update/delete + diff-синк)

**Files:**
- Modify: `web.admin/src/lib/articles.ts`
- Test: `web.admin/src/lib/articles.test.ts`

- [ ] **Step 1: Добавить section→row маппинг и diff-синк секций**

Дописать в `articles.ts`:

```ts
// ArticleSection → значения строки post_sections (только колонки своего типа,
// прочие null). position задаётся вызывающим.
export function sectionToValues(
  postId: string,
  s: ArticleSection,
  position: number,
): Record<string, unknown> {
  const nulls = {
    body: null as string | null, text_variant: null as string | null,
    quote: null as string | null, quote_attribution: null as string | null,
    image_path: null as string | null, caption: null as string | null, credit: null as string | null,
    recipe_id: null as string | null, card_eyebrow: null as string | null,
    question: null as string | null, answer: null as string | null,
    rank: null as number | null, heading: null as string | null,
  };
  const common = { post_id: postId, position, is_published: s.isPublished, type: s.kind, ...nulls };
  switch (s.kind) {
    case "text": return { ...common, body: s.body, text_variant: s.variant };
    case "quote": return { ...common, quote: s.quote, quote_attribution: s.attribution };
    case "image": return { ...common, image_path: s.imagePath, caption: s.caption, credit: s.credit };
    case "recipe_card": return { ...common, recipe_id: s.recipeId, card_eyebrow: s.eyebrow };
    case "qa": return { ...common, question: s.question, answer: s.answer };
    case "list_item": return { ...common, rank: s.rank, heading: s.heading, body: s.body, recipe_id: s.recipeId, card_eyebrow: s.eyebrow };
  }
}

// Единая таблица post_sections: insert новых (без sectionId) → update существующих
// (контент И position пишем всегда) → delete пропавших строго в конце.
async function syncSections(
  site: SiteConfig,
  postId: string,
  sections: ArticleSection[],
  loaded: ArticleSection[],
): Promise<void> {
  const db = getDb(site);
  const inserts = sections
    .map((s, i) => ({ s, i }))
    .filter(({ s }) => !s.sectionId)
    .map(({ s, i }) => sectionToValues(postId, s, i));
  const updates = sections
    .map((s, i) => ({ s, i }))
    .filter(({ s }) => Boolean(s.sectionId));
  const keep = new Set(updates.map(({ s }) => s.sectionId));
  const deleteIds = loaded
    .map((s) => s.sectionId)
    .filter((id): id is string => Boolean(id) && !keep.has(id));

  if (inserts.length) {
    const { error } = await db.from("post_sections").insert(inserts);
    if (error) throw error;
  }
  for (const { s, i } of updates) {
    const { error } = await db
      .from("post_sections")
      .update(sectionToValues(postId, s, i))
      .eq("id", s.sectionId!);
    if (error) throw error;
  }
  if (deleteIds.length) {
    const { error } = await db.from("post_sections").delete().in("id", deleteIds);
    if (error) throw error;
  }
}
```

- [ ] **Step 2: Добавить синк тегов и Read-also (упорядоченные m2m — replace-all)**

Дописать в `articles.ts`. Для упорядоченных join-таблиц (position важен) —
delete-all-then-insert (проще и корректнее diff при перестановке порядка):

```ts
// post_tags: position = порядок в массиве (= порядок эйброу). Replace-all.
async function syncTags(site: SiteConfig, postId: string, tagIds: string[]): Promise<void> {
  const db = getDb(site);
  const { error: delErr } = await db.from("post_tags").delete().eq("post_id", postId);
  if (delErr) throw delErr;
  if (tagIds.length) {
    const rows = tagIds.map((tag_id, position) => ({ post_id: postId, tag_id, position }));
    const { error } = await db.from("post_tags").insert(rows);
    if (error) throw error;
  }
}

// post_related: полиморфно recipe_id XOR related_post_id; position = порядок пинов.
async function syncRelated(site: SiteConfig, postId: string, pins: RelatedPin[]): Promise<void> {
  const db = getDb(site);
  const { error: delErr } = await db.from("post_related").delete().eq("post_id", postId);
  if (delErr) throw delErr;
  if (pins.length) {
    const rows = pins.map((p, position) => ({
      post_id: postId,
      position,
      recipe_id: p.kind === "recipe" ? p.id : null,
      related_post_id: p.kind === "post" ? p.id : null,
    }));
    const { error } = await db.from("post_related").insert(rows);
    if (error) throw error;
  }
}
```

- [ ] **Step 3: Добавить create/update/delete**

```ts
export async function createArticle(
  site: SiteConfig,
  input: ArticleInput,
  sections: ArticleSection[],
  tagIds: string[],
  related: RelatedPin[],
): Promise<Article> {
  const db = getDb(site);
  // slug не отправляем — триггер posts_set_slug. is_published задаём явно (БД default true).
  const { data, error } = await db.from("posts").insert(input).select(ARTICLE_COLUMNS).single();
  if (error) throw error;
  const article = data as Article;
  try {
    await syncSections(site, article.id, sections, []);
    await syncTags(site, article.id, tagIds);
    await syncRelated(site, article.id, related);
  } catch (e) {
    await db.from("posts").delete().eq("id", article.id); // откат пост-огрызка
    throw e;
  }
  return article;
}

// Возвращает свежие данные: у новых секций появились sectionId — форма ОБЯЗАНА
// пересинхронизироваться, иначе повторный Save продублирует их.
export async function updateArticle(
  site: SiteConfig,
  id: string,
  input: ArticleInput,
  sections: ArticleSection[],
  loadedSections: ArticleSection[],
  tagIds: string[],
  related: RelatedPin[],
): Promise<ArticleWithRelations> {
  const { error } = await getDb(site).from("posts").update(input).eq("id", id);
  if (error) throw error;
  await syncSections(site, id, sections, loadedSections);
  await syncTags(site, id, tagIds);
  await syncRelated(site, id, related);
  const fresh = await getArticle(site, id);
  if (!fresh) throw new Error("Article not found after save");
  return fresh;
}

export async function deleteArticle(site: SiteConfig, id: string): Promise<void> {
  // секции/теги/пины уберёт FK ON DELETE CASCADE
  const { error } = await getDb(site).from("posts").delete().eq("id", id);
  if (error) throw error;
}
```

- [ ] **Step 4: Тест `sectionToValues`**

Дописать в `articles.test.ts`:

```ts
import { sectionToValues } from "./articles";

describe("sectionToValues", () => {
  it("text: fills only text columns, others null, correct type/position", () => {
    const v = sectionToValues("p1", { kind: "text", isPublished: true, body: "Hi", variant: "lead" }, 2);
    expect(v).toMatchObject({ post_id: "p1", position: 2, is_published: true, type: "text", body: "Hi", text_variant: "lead", quote: null, recipe_id: null });
  });
  it("recipe_card: fills recipe_id + card_eyebrow only", () => {
    const v = sectionToValues("p1", { kind: "recipe_card", isPublished: false, recipeId: "r1", eyebrow: null }, 0);
    expect(v).toMatchObject({ type: "recipe_card", recipe_id: "r1", card_eyebrow: null, body: null });
  });
});
```

- [ ] **Step 5: Прогнать тесты и закоммитить**

Run: `npm --prefix web.admin test -- articles.test`
Expected: PASS (5 тестов).

```bash
git add web.admin/src/lib/articles.ts web.admin/src/lib/articles.test.ts
git commit -m "feat(articles): write layer (create/update/delete + section/tag/related sync)"
```

---

## Task 3: Хелперы рецептов/тегов + `RecipePickerDialog`

**Files:**
- Create: `web.admin/src/lib/recipes.ts`
- Create: `web.admin/src/lib/tags.ts`
- Create: `web.admin/src/features/articles/RecipePickerDialog.tsx`

- [ ] **Step 1: `recipes.ts` (лёгкий список для пикера)**

```ts
// web.admin/src/lib/recipes.ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

// Лёгкая строка рецепта для FK-пикера секций/Read-also (поиск по title + превью).
export type RecipeListItem = {
  id: string;
  title: string;
  slug: string;
  hero_image_path: string | null;
  is_published: boolean;
};

export async function listRecipes(site: SiteConfig): Promise<RecipeListItem[]> {
  const { data, error } = await getDb(site)
    .from("recipes")
    .select("id,title,slug,hero_image_path,is_published")
    .order("title", { ascending: true });
  if (error) throw error;
  return (data ?? []) as RecipeListItem[];
}
```

- [ ] **Step 2: `tags.ts` (справочник тегов)**

```ts
// web.admin/src/lib/tags.ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

export type Tag = { id: string; name: string; slug: string; position: number };

export async function listTags(site: SiteConfig): Promise<Tag[]> {
  const { data, error } = await getDb(site)
    .from("tags")
    .select("id,name,slug,position")
    .order("position", { ascending: true })
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as Tag[];
}
```

- [ ] **Step 3: `RecipePickerDialog` (single-select, порт `ProductPickerDialog`)**

Скопировать `web.admin/src/features/posts/ProductPickerDialog.tsx` в новый файл и
изменить: (а) источник — `listRecipes` вместо `listProducts`; (б) **single-select**
(`onSelect(id)` вместо мультивыбора `onAdd(ids)`); (в) заголовок «Add recipe»;
(г) показывать бейдж Draft у неопубликованных (напоминание: карточка на сайте
рендерится только у published рецепта).

```tsx
// web.admin/src/features/articles/RecipePickerDialog.tsx
import { useDeferredValue, useEffect, useMemo, useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { ImageOff, Search } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listRecipes } from '@/lib/recipes'
import { resolveImageUrl } from '@/lib/images'
import { cn } from '@/lib/utils'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Dialog, DialogBody, DialogContent, DialogFooter, DialogHeader, DialogTitle,
} from '@/components/ui/dialog'

// Пикер рецептов для секций recipe_card/list_item и Read-also. Single-select:
// клик по строке сразу выбирает рецепт и закрывает диалог.
export function RecipePickerDialog({
  site, open, onOpenChange, onSelect,
}: {
  site: SiteConfig
  open: boolean
  onOpenChange: (open: boolean) => void
  onSelect: (id: string) => void
}) {
  const [search, setSearch] = useState('')
  const deferredSearch = useDeferredValue(search)
  useEffect(() => { if (open) setSearch('') }, [open])

  const { data: recipes, isPending, error } = useQuery({
    queryKey: ['recipes', site.slug],
    queryFn: () => listRecipes(site),
    enabled: open,
  })

  const query = deferredSearch.trim().toLowerCase()
  const visible = useMemo(
    () => (recipes ?? []).filter((r) => !query || r.title.toLowerCase().includes(query)),
    [recipes, query],
  )

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-2xl">
        <DialogHeader><DialogTitle>Add recipe</DialogTitle></DialogHeader>

        <div className="relative w-full max-w-56">
          <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          <Input
            type="search" placeholder="Search by title…" value={search}
            onChange={(e) => setSearch(e.target.value)} className="h-9 pl-8"
          />
        </div>

        {error && (
          <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
            Failed to load recipes: {error.message}
          </p>
        )}

        <DialogBody>
          {isPending && !error && (
            <div className="flex flex-col gap-1">
              {Array.from({ length: 6 }, (_, i) => (
                <Skeleton key={i} className="h-12 w-full rounded-md" />
              ))}
            </div>
          )}
          {recipes && visible.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              {recipes.length === 0 ? 'No recipes yet.' : 'No recipes match.'}
            </p>
          )}
          {visible.length > 0 && (
            <ul className="flex flex-col gap-1">
              {visible.map((r) => (
                <li key={r.id}>
                  <button
                    type="button"
                    onClick={() => { onSelect(r.id); onOpenChange(false) }}
                    className={cn(
                      'flex w-full items-center gap-3 rounded-md px-2 py-1.5 text-left transition-colors hover:bg-accent/50',
                    )}
                  >
                    {r.hero_image_path ? (
                      <img src={resolveImageUrl(site, r.hero_image_path)} alt="" loading="lazy"
                        className="size-9 shrink-0 rounded border object-cover" />
                    ) : (
                      <span className="flex size-9 shrink-0 items-center justify-center rounded border bg-muted/30">
                        <ImageOff className="size-4 text-muted-foreground" />
                      </span>
                    )}
                    <span className="min-w-0 flex-1 truncate text-sm font-medium">{r.title}</span>
                    {!r.is_published && <Badge variant="outline">Draft</Badge>}
                  </button>
                </li>
              ))}
            </ul>
          )}
        </DialogBody>

        <DialogFooter>
          <Button type="button" variant="outline" onClick={() => onOpenChange(false)}>Cancel</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 4: Проверка типов/сборки**

Run: `npm --prefix web.admin run build`
Expected: сборка проходит (файлы ещё не импортированы — проверяем, что типы валидны).

- [ ] **Step 5: Commit**

```bash
git add web.admin/src/lib/recipes.ts web.admin/src/lib/tags.ts web.admin/src/features/articles/RecipePickerDialog.tsx
git commit -m "feat(articles): recipes/tags data helpers + RecipePickerDialog (single-select)"
```

---

## Task 4: Zod-схемы формы + маппинг form⇄DB (`articleForm.ts`)

Вынесено в отдельный модуль (без React), чтобы покрыть unit-тестами независимо от UI.

**Files:**
- Create: `web.admin/src/features/articles/articleForm.ts`
- Test: `web.admin/src/features/articles/articleForm.test.ts`

- [ ] **Step 1: Схемы и типы формы**

```ts
// web.admin/src/features/articles/articleForm.ts
import { z } from 'zod'
import type { SiteConfig } from '@/config/sites'
import { resolveImageUrl, toStoragePath } from '@/lib/images'
import { toDatetimeLocalValue } from '@/lib/format'
import type {
  Article, ArticleInput, ArticleSection, ArticleTemplate, ArticleWithRelations, RelatedPin,
} from '@/lib/articles'

// Поле id секции в форме — sectionId (useFieldArray резервирует `id`).
const textSchema = z.object({
  kind: z.literal('text'), sectionId: z.string().optional(), isPublished: z.boolean(),
  variant: z.literal('lead').or(z.literal('body')),
  body: z.string().trim().min(1, 'Text is required'),
})
const quoteSchema = z.object({
  kind: z.literal('quote'), sectionId: z.string().optional(), isPublished: z.boolean(),
  quote: z.string().trim().min(1, 'Quote is required'),
  attribution: z.string(),
})
const imageSchema = z.object({
  kind: z.literal('image'), sectionId: z.string().optional(), isPublished: z.boolean(),
  imagePath: z.string(), caption: z.string(), credit: z.string(),
}).refine((s) => s.imagePath.trim() || s.caption.trim(), {
  message: 'Add an image or a caption', path: ['imagePath'],
})
const recipeCardSchema = z.object({
  kind: z.literal('recipe_card'), sectionId: z.string().optional(), isPublished: z.boolean(),
  recipeId: z.string().min(1, 'Pick a recipe'), eyebrow: z.string(),
})
const qaSchema = z.object({
  kind: z.literal('qa'), sectionId: z.string().optional(), isPublished: z.boolean(),
  question: z.string().trim().min(1, 'Question is required'),
  answer: z.string().trim().min(1, 'Answer is required'),
})
const listItemSchema = z.object({
  kind: z.literal('list_item'), sectionId: z.string().optional(), isPublished: z.boolean(),
  rank: z.string(), // number в инпуте храним строкой; в toSections парсим
  heading: z.string().trim().min(1, 'Heading is required'),
  body: z.string(), recipeId: z.string(), eyebrow: z.string(),
})

export const articleSchema = z.object({
  title: z.string().trim().min(1, 'Title is required'),
  template: z.enum(['essay', 'interview', 'roundup']),
  excerpt: z.string(),
  hero_image_path: z.string().trim(),
  author_id: z.string(), // '' → null
  subtitle: z.string(),
  hero_caption: z.string(),
  read_minutes: z.string(), // '' → null; иначе parseInt
  is_published: z.boolean(),
  published_at: z.string().min(1, 'Publish date is required'),
  seo_title: z.string(),
  seo_description: z.string(),
  tagIds: z.array(z.string()),
  related: z.array(
    z.object({ kind: z.enum(['recipe', 'post']), id: z.string().min(1) }),
  ),
  sections: z.array(
    z.discriminatedUnion('kind', [
      textSchema, quoteSchema, imageSchema, recipeCardSchema, qaSchema, listItemSchema,
    ]),
  ),
})

export type ArticleFormValues = z.infer<typeof articleSchema>
export type SectionFormValue = ArticleFormValues['sections'][number]

const orNull = (v: string) => (v.trim() ? v.trim() : null)
function seoOrNull(value: string, fallback: string): string | null {
  const t = value.trim()
  return t && t !== fallback.trim() ? t : null
}
```

- [ ] **Step 2: Маппинг form → DB (`toInput`, `toSections`, `toRelated`)**

Дописать:

```ts
export function toInput(site: SiteConfig, v: ArticleFormValues): ArticleInput {
  const readMin = v.read_minutes.trim() ? Number.parseInt(v.read_minutes, 10) : null
  return {
    title: v.title.trim(),
    template: v.template,
    author_id: orNull(v.author_id),
    excerpt: orNull(v.excerpt),
    hero_image_path: orNull(toStoragePath(site, v.hero_image_path.trim())),
    subtitle: orNull(v.subtitle),
    hero_caption: orNull(v.hero_caption),
    read_minutes: Number.isFinite(readMin as number) ? (readMin as number) : null,
    is_published: v.is_published,
    published_at: new Date(v.published_at).toISOString(),
    seo_title: seoOrNull(v.seo_title, v.title),
    seo_description: seoOrNull(v.seo_description, v.excerpt),
  }
}

export function toSections(v: ArticleFormValues): ArticleSection[] {
  return v.sections.map((s): ArticleSection => {
    switch (s.kind) {
      case 'text': return { kind: 'text', sectionId: s.sectionId, isPublished: s.isPublished, body: s.body, variant: s.variant }
      case 'quote': return { kind: 'quote', sectionId: s.sectionId, isPublished: s.isPublished, quote: s.quote, attribution: orNull(s.attribution) }
      case 'image': return { kind: 'image', sectionId: s.sectionId, isPublished: s.isPublished, imagePath: orNull(s.imagePath), caption: orNull(s.caption), credit: orNull(s.credit) }
      case 'recipe_card': return { kind: 'recipe_card', sectionId: s.sectionId, isPublished: s.isPublished, recipeId: orNull(s.recipeId), eyebrow: orNull(s.eyebrow) }
      case 'qa': return { kind: 'qa', sectionId: s.sectionId, isPublished: s.isPublished, question: s.question, answer: s.answer }
      case 'list_item': {
        const rank = s.rank.trim() ? Number.parseInt(s.rank, 10) : null
        return { kind: 'list_item', sectionId: s.sectionId, isPublished: s.isPublished, rank: Number.isFinite(rank as number) ? (rank as number) : null, heading: s.heading, body: orNull(s.body), recipeId: orNull(s.recipeId), eyebrow: orNull(s.eyebrow) }
      }
    }
  })
}

export function toRelated(v: ArticleFormValues): RelatedPin[] {
  return v.related.map((r) => ({ kind: r.kind, id: r.id }) as RelatedPin)
}
```

- [ ] **Step 3: Маппинг DB → form (`toFormValues`, `toSectionFormValue`)**

Дописать. Хранить image/hero как публичный URL для инпута; поля картинок сохранять
как resolveImageUrl (в секции image тоже):

```ts
function toSectionFormValue(site: SiteConfig, s: ArticleSection): SectionFormValue {
  switch (s.kind) {
    case 'text': return { kind: 'text', sectionId: s.sectionId, isPublished: s.isPublished, variant: s.variant, body: s.body }
    case 'quote': return { kind: 'quote', sectionId: s.sectionId, isPublished: s.isPublished, quote: s.quote, attribution: s.attribution ?? '' }
    case 'image': return { kind: 'image', sectionId: s.sectionId, isPublished: s.isPublished, imagePath: s.imagePath ? resolveImageUrl(site, s.imagePath) : '', caption: s.caption ?? '', credit: s.credit ?? '' }
    case 'recipe_card': return { kind: 'recipe_card', sectionId: s.sectionId, isPublished: s.isPublished, recipeId: s.recipeId ?? '', eyebrow: s.eyebrow ?? '' }
    case 'qa': return { kind: 'qa', sectionId: s.sectionId, isPublished: s.isPublished, question: s.question, answer: s.answer }
    case 'list_item': return { kind: 'list_item', sectionId: s.sectionId, isPublished: s.isPublished, rank: s.rank == null ? '' : String(s.rank), heading: s.heading, body: s.body ?? '', recipeId: s.recipeId ?? '', eyebrow: s.eyebrow ?? '' }
  }
}

export function toFormValues(
  site: SiteConfig,
  data: ArticleWithRelations | null,
): ArticleFormValues {
  const a: Article | undefined = data?.article
  return {
    title: a?.title ?? '',
    template: a?.template ?? 'essay',
    excerpt: a?.excerpt ?? '',
    hero_image_path: a?.hero_image_path ? resolveImageUrl(site, a.hero_image_path) : '',
    author_id: a?.author_id ?? '',
    subtitle: a?.subtitle ?? '',
    hero_caption: a?.hero_caption ?? '',
    read_minutes: a?.read_minutes == null ? '' : String(a.read_minutes),
    is_published: a?.is_published ?? true,
    published_at: toDatetimeLocalValue(a?.published_at ?? new Date().toISOString()),
    seo_title: a?.seo_title ?? '',
    seo_description: a?.seo_description ?? '',
    tagIds: data?.tagIds ?? [],
    related: (data?.related ?? []).map((r) => ({ kind: r.kind, id: r.id })),
    sections: (data?.sections ?? []).map((s) => toSectionFormValue(site, s)),
  }
}

// Дефолт для нового блока по типу (для кнопки Add block).
export function emptySection(kind: SectionFormValue['kind']): SectionFormValue {
  switch (kind) {
    case 'text': return { kind: 'text', isPublished: true, variant: 'body', body: '' }
    case 'quote': return { kind: 'quote', isPublished: true, quote: '', attribution: '' }
    case 'image': return { kind: 'image', isPublished: true, imagePath: '', caption: '', credit: '' }
    case 'recipe_card': return { kind: 'recipe_card', isPublished: true, recipeId: '', eyebrow: '' }
    case 'qa': return { kind: 'qa', isPublished: true, question: '', answer: '' }
    case 'list_item': return { kind: 'list_item', isPublished: true, rank: '', heading: '', body: '', recipeId: '', eyebrow: '' }
  }
}
```

- [ ] **Step 4: Unit-тесты маппинга**

```ts
// web.admin/src/features/articles/articleForm.test.ts
import { describe, expect, it } from 'vitest'
import type { SiteConfig } from '@/config/sites'
import { articleSchema, toInput, toSections, toFormValues, emptySection } from './articleForm'

const site = {
  slug: 'avocado-kiss', label: 'Avocado Kiss', projectUrl: 'https://demo.supabase.co',
  anonKey: 'k', schema: 'avocado_kiss', bucket: 'avocado-kiss-photos',
} as SiteConfig

const baseValues = {
  title: 'Post', template: 'essay' as const, excerpt: '', hero_image_path: '',
  author_id: '', subtitle: '', hero_caption: '', read_minutes: '', is_published: false,
  published_at: '2026-08-08T10:00', seo_title: '', seo_description: '',
  tagIds: [], related: [], sections: [],
}

describe('toInput', () => {
  it("empty optionals → null; read_minutes '' → null; seo falls back → null", () => {
    const input = toInput(site, baseValues)
    expect(input).toMatchObject({
      title: 'Post', template: 'essay', author_id: null, excerpt: null,
      hero_image_path: null, subtitle: null, hero_caption: null,
      read_minutes: null, seo_title: null, seo_description: null, is_published: false,
    })
  })
  it('read_minutes parses to int', () => {
    expect(toInput(site, { ...baseValues, read_minutes: '7' }).read_minutes).toBe(7)
  })
  it('keeps seo_title when different from title', () => {
    expect(toInput(site, { ...baseValues, seo_title: 'Custom' }).seo_title).toBe('Custom')
  })
})

describe('toSections', () => {
  it('list_item: rank string → number, empty optionals → null', () => {
    const [s] = toSections({ ...baseValues, sections: [emptySection('list_item')] })
    expect(s).toMatchObject({ kind: 'list_item', rank: null, heading: '', body: null, recipeId: null })
  })
})

describe('articleSchema validation', () => {
  it('rejects recipe_card without recipeId', () => {
    const r = articleSchema.safeParse({ ...baseValues, sections: [emptySection('recipe_card')] })
    expect(r.success).toBe(false)
  })
  it('accepts text block with body', () => {
    const r = articleSchema.safeParse({ ...baseValues, sections: [{ ...emptySection('text'), body: 'Hi' }] })
    expect(r.success).toBe(true)
  })
})

describe('toFormValues', () => {
  it('null data → essay default, is_published true default', () => {
    const v = toFormValues(site, null)
    expect(v.template).toBe('essay')
    expect(v.is_published).toBe(true)
    expect(v.sections).toEqual([])
  })
})
```

- [ ] **Step 5: Прогнать и закоммитить**

Run: `npm --prefix web.admin test -- articleForm.test`
Expected: PASS.

```bash
git add web.admin/src/features/articles/articleForm.ts web.admin/src/features/articles/articleForm.test.ts
git commit -m "feat(articles): form schema + form⇄DB mapping (unit-tested)"
```

---

## Task 5: Мультиселект тегов с порядком (`TagsField`)

**Files:**
- Create: `web.admin/src/features/articles/TagsField.tsx`

- [ ] **Step 1: Компонент (порт `CategoryChipsField` + ↑/↓ порядок)**

Порт `CategoryChipsField.tsx` c источником `listTags` и **сохранением порядка**
(порядок чипов = порядок эйброу): чипы с кнопками ↑/↓ и удалением, поповер «Add tag».

```tsx
// web.admin/src/features/articles/TagsField.tsx
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { ArrowLeft, ArrowRight, Plus, X } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listTags } from '@/lib/tags'
import { Button } from '@/components/ui/button'
import {
  Command, CommandEmpty, CommandGroup, CommandInput, CommandItem, CommandList,
} from '@/components/ui/command'
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover'

// Значение — упорядоченный массив tag_id (порядок = post_tags.position = порядок
// эйброу на сайте). Чипы с перестановкой ←/→ и удалением + поповер Add tag.
export function TagsField({
  site, value, onChange,
}: {
  site: SiteConfig
  value: string[]
  onChange: (ids: string[]) => void
}) {
  const [open, setOpen] = useState(false)
  const { data: tags } = useQuery({ queryKey: ['tags', site.slug], queryFn: () => listTags(site) })

  const byId = new Map((tags ?? []).map((t) => [t.id, t.name]))
  const selected = value.filter((id) => byId.has(id))
  const available = (tags ?? []).filter((t) => !value.includes(t.id))

  function move(i: number, delta: number) {
    const j = i + delta
    if (j < 0 || j >= value.length) return
    const next = [...value]
    ;[next[i], next[j]] = [next[j], next[i]]
    onChange(next)
  }

  return (
    <div className="flex flex-wrap items-center gap-2">
      {selected.length === 0 && <span className="text-sm text-muted-foreground">No tags</span>}
      {selected.map((id, i) => (
        <span key={id} className="inline-flex items-center gap-1 rounded-full border bg-accent/40 py-0.5 pr-1 pl-2.5 text-sm">
          <Button type="button" variant="ghost" size="icon-sm" className="size-5 rounded-full"
            aria-label={`Move ${byId.get(id)} earlier`} disabled={i === 0} onClick={() => move(i, -1)}>
            <ArrowLeft className="size-3" />
          </Button>
          <span className="max-w-40 truncate">{byId.get(id)}</span>
          <Button type="button" variant="ghost" size="icon-sm" className="size-5 rounded-full"
            aria-label={`Move ${byId.get(id)} later`} disabled={i === selected.length - 1} onClick={() => move(i, 1)}>
            <ArrowRight className="size-3" />
          </Button>
          <Button type="button" variant="ghost" size="icon-sm" className="size-5 rounded-full"
            aria-label={`Remove ${byId.get(id)}`} onClick={() => onChange(value.filter((x) => x !== id))}>
            <X className="size-3.5" />
          </Button>
        </span>
      ))}

      <Popover open={open} onOpenChange={setOpen}>
        <PopoverTrigger asChild>
          <Button type="button" variant="outline" size="sm" className="h-8">
            <Plus className="size-4" />
            Add tag
          </Button>
        </PopoverTrigger>
        <PopoverContent className="w-56 p-0" align="start">
          <Command>
            <CommandInput placeholder="Search tags…" />
            <CommandList>
              <CommandEmpty>No tags.</CommandEmpty>
              <CommandGroup>
                {available.map((t) => (
                  <CommandItem key={t.id} value={t.name} className="cursor-pointer"
                    onSelect={() => { onChange([...value, t.id]); setOpen(false) }}>
                    <span className="truncate">{t.name}</span>
                  </CommandItem>
                ))}
              </CommandGroup>
            </CommandList>
          </Command>
        </PopoverContent>
      </Popover>
    </div>
  )
}
```

- [ ] **Step 2: Проверка сборки и commit**

Run: `npm --prefix web.admin run build`
Expected: OK.

```bash
git add web.admin/src/features/articles/TagsField.tsx
git commit -m "feat(articles): ordered tags multiselect (TagsField)"
```

---

## Task 6: Конструктор секций (`SectionsEditor`)

**Files:**
- Create: `web.admin/src/features/articles/SectionsEditor.tsx`

- [ ] **Step 1: Каркас — useFieldArray, карточка, «Add block» с выбором типа**

Порт `PostModulesEditor` с 6 типами. Одна кнопка «Add block» открывает
`DropdownMenu` (shadcn) с 6 пунктами → `append(emptySection(kind))`. Один общий
`RecipePickerDialog` на всю форму (`recipePickerFor: number | null`).

```tsx
// web.admin/src/features/articles/SectionsEditor.tsx
import { useState } from 'react'
import { Controller, useFieldArray, type UseFormReturn } from 'react-hook-form'
import {
  AlignLeft, ArrowDown, ArrowUp, Eye, EyeOff, HelpCircle, Image as ImageIcon,
  ListOrdered, Plus, Quote, Trash2, UtensilsCrossed, X,
} from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { cn } from '@/lib/utils'
import { RichTextEditor } from '@/components/RichTextEditor'
import { ImagePreviewPicker } from '@/components/ImagePreviewPicker'
import { ImagePickerDialog } from '@/features/products/ImagePickerDialog'
import { resolveImageUrl, toStoragePath } from '@/lib/images'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Field, FieldError, FieldLabel } from '@/components/ui/field'
import {
  DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { RecipePickerDialog } from './RecipePickerDialog'
import { emptySection, type ArticleFormValues, type SectionFormValue } from './articleForm'
import { RecipeChip } from './RecipeChip'

const TYPE_META: Record<SectionFormValue['kind'], { label: string; icon: typeof AlignLeft }> = {
  text: { label: 'Text', icon: AlignLeft },
  quote: { label: 'Quote', icon: Quote },
  image: { label: 'Image', icon: ImageIcon },
  recipe_card: { label: 'Recipe card', icon: UtensilsCrossed },
  qa: { label: 'Q & A', icon: HelpCircle },
  list_item: { label: 'List item', icon: ListOrdered },
}
const ADD_ORDER: SectionFormValue['kind'][] = ['text', 'quote', 'image', 'recipe_card', 'qa', 'list_item']

export function SectionsEditor({
  site, form,
}: {
  site: SiteConfig
  form: UseFormReturn<ArticleFormValues>
}) {
  const { fields, append, remove, move } = useFieldArray({ control: form.control, name: 'sections' })
  const [recipePickerFor, setRecipePickerFor] = useState<number | null>(null)
  const [imagePickerFor, setImagePickerFor] = useState<number | null>(null)
  const { errors } = form.formState

  return (
    <div className="flex flex-col gap-3">
      <span className="text-sm font-medium">Content</span>

      {fields.length === 0 && (
        <p className="rounded-lg border border-dashed p-6 text-center text-sm text-muted-foreground">
          No content yet. Add a block below.
        </p>
      )}

      {fields.map((field, index) => {
        const kind = field.kind as SectionFormValue['kind']
        const isPublished = form.watch(`sections.${index}.isPublished`)
        const meta = TYPE_META[kind]
        const Icon = meta.icon
        const sectionErrors = errors.sections?.[index]
        return (
          <div key={field.id} className={cn('rounded-xl border', !isPublished && 'border-dashed opacity-60')}>
            <div className="flex items-center gap-2 border-b px-3 py-2">
              <Icon className="size-4 text-muted-foreground" />
              <span className="text-sm font-medium">{meta.label}</span>
              {!isPublished && <span className="text-xs text-muted-foreground">(hidden)</span>}
              <div className="ml-auto flex items-center gap-1">
                <Button type="button" variant="ghost" size="icon" className="size-7"
                  aria-label={isPublished ? 'Hide block' : 'Show block'}
                  onClick={() => form.setValue(`sections.${index}.isPublished`, !isPublished, { shouldDirty: true })}>
                  {isPublished ? <Eye className="size-4" /> : <EyeOff className="size-4" />}
                </Button>
                <Button type="button" variant="ghost" size="icon" className="size-7" aria-label="Move up"
                  disabled={index === 0} onClick={() => move(index, index - 1)}>
                  <ArrowUp className="size-4" />
                </Button>
                <Button type="button" variant="ghost" size="icon" className="size-7" aria-label="Move down"
                  disabled={index === fields.length - 1} onClick={() => move(index, index + 1)}>
                  <ArrowDown className="size-4" />
                </Button>
                <Button type="button" variant="ghost" size="icon" className="size-7 text-destructive hover:text-destructive"
                  aria-label="Remove block" onClick={() => remove(index)}>
                  <Trash2 className="size-4" />
                </Button>
              </div>
            </div>

            <div className="flex flex-col gap-3 p-3">
              <SectionFields
                site={site} form={form} index={index} kind={kind}
                sectionErrors={sectionErrors}
                onPickRecipe={() => setRecipePickerFor(index)}
                onPickImage={() => setImagePickerFor(index)}
              />
            </div>
          </div>
        )
      })}

      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button type="button" variant="outline" className="w-fit">
            <Plus />
            Add block
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="start">
          {ADD_ORDER.map((k) => {
            const M = TYPE_META[k]
            const Icon = M.icon
            return (
              <DropdownMenuItem key={k} onClick={() => append(emptySection(k))}>
                <Icon className="size-4" />
                {M.label}
              </DropdownMenuItem>
            )
          })}
        </DropdownMenuContent>
      </DropdownMenu>

      <RecipePickerDialog
        site={site}
        open={recipePickerFor !== null}
        onOpenChange={(open) => { if (!open) setRecipePickerFor(null) }}
        onSelect={(id) => {
          if (recipePickerFor === null) return
          form.setValue(`sections.${recipePickerFor}.recipeId`, id, { shouldDirty: true, shouldValidate: true })
        }}
      />

      <ImagePickerDialog
        site={site}
        open={imagePickerFor !== null}
        onOpenChange={(open) => { if (!open) setImagePickerFor(null) }}
        currentPath={
          imagePickerFor !== null
            ? toStoragePath(site, String(form.getValues(`sections.${imagePickerFor}.imagePath`) || ''))
            : null
        }
        onSelect={(path) => {
          if (imagePickerFor === null) return
          form.setValue(`sections.${imagePickerFor}.imagePath`, resolveImageUrl(site, path), { shouldDirty: true, shouldValidate: true })
        }}
      />
    </div>
  )
}
```

- [ ] **Step 2: Пофайловые поля блока (`SectionFields`) в том же файле**

Дописать компонент `SectionFields`, рендерящий контролы по `kind`. `body` у text/
list_item — `RichTextEditor` через Controller; `recipeId` — `RecipeChip` + кнопка
пикера; `imagePath` — `ImagePreviewPicker` + инпут; variant — сегмент-тоггл.

```tsx
function SectionFields({
  site, form, index, kind, sectionErrors, onPickRecipe, onPickImage,
}: {
  site: SiteConfig
  form: UseFormReturn<ArticleFormValues>
  index: number
  kind: SectionFormValue['kind']
  sectionErrors: unknown
  onPickRecipe: () => void
  onPickImage: () => void
}) {
  const reg = form.register
  const err = (name: string) =>
    (sectionErrors as Record<string, { message?: string }> | undefined)?.[name]

  if (kind === 'text') {
    return (
      <>
        <Controller control={form.control} name={`sections.${index}.variant`}
          render={({ field }) => (
            <div className="flex items-center gap-1">
              <span className="mr-1 text-sm text-muted-foreground">Style:</span>
              {(['body', 'lead'] as const).map((v) => (
                <Button key={v} type="button" size="sm"
                  variant={field.value === v ? 'default' : 'outline'}
                  aria-pressed={field.value === v} onClick={() => field.onChange(v)}>
                  {v === 'body' ? 'Body' : 'Lead'}
                </Button>
              ))}
            </div>
          )} />
        <Field data-invalid={!!err('body')}>
          <Controller control={form.control} name={`sections.${index}.body`}
            render={({ field }) => <RichTextEditor value={field.value} onChange={field.onChange} />} />
          <FieldError errors={[err('body')]} />
        </Field>
      </>
    )
  }
  if (kind === 'quote') {
    return (
      <>
        <Field data-invalid={!!err('quote')}>
          <FieldLabel htmlFor={`quote-${index}`}>Quote</FieldLabel>
          <Textarea id={`quote-${index}`} rows={2} {...reg(`sections.${index}.quote`)} />
          <FieldError errors={[err('quote')]} />
        </Field>
        <Field>
          <FieldLabel htmlFor={`attr-${index}`}>Attribution</FieldLabel>
          <Input id={`attr-${index}`} placeholder="Optional" {...reg(`sections.${index}.attribution`)} />
        </Field>
      </>
    )
  }
  if (kind === 'image') {
    const imagePath = String(form.watch(`sections.${index}.imagePath`) || '')
    return (
      <>
        <Field data-invalid={!!err('imagePath')}>
          <FieldLabel>Image</FieldLabel>
          <div className="max-w-xs">
            <ImagePreviewPicker
              url={imagePath ? resolveImageUrl(site, imagePath) : null}
              alt="Section image" aspect="video" objectFit="cover" onPick={onPickImage} />
          </div>
          <Input className="mt-2" placeholder="flat key in bucket or external URL" {...reg(`sections.${index}.imagePath`)} />
          <FieldError errors={[err('imagePath')]} />
        </Field>
        <Field>
          <FieldLabel htmlFor={`cap-${index}`}>Caption</FieldLabel>
          <Input id={`cap-${index}`} placeholder="Optional" {...reg(`sections.${index}.caption`)} />
        </Field>
        <Field>
          <FieldLabel htmlFor={`cred-${index}`}>Credit</FieldLabel>
          <Input id={`cred-${index}`} placeholder="Optional" {...reg(`sections.${index}.credit`)} />
        </Field>
      </>
    )
  }
  if (kind === 'recipe_card') {
    const recipeId = String(form.watch(`sections.${index}.recipeId`) || '')
    return (
      <>
        <Field data-invalid={!!err('recipeId')}>
          <FieldLabel>Recipe</FieldLabel>
          <RecipeChip site={site} recipeId={recipeId} onPick={onPickRecipe}
            onClear={() => form.setValue(`sections.${index}.recipeId`, '', { shouldDirty: true, shouldValidate: true })} />
          <FieldError errors={[err('recipeId')]} />
        </Field>
        <Field>
          <FieldLabel htmlFor={`eyebrow-${index}`}>Eyebrow</FieldLabel>
          <Input id={`eyebrow-${index}`} placeholder='Overrides "Recipe"' {...reg(`sections.${index}.eyebrow`)} />
        </Field>
      </>
    )
  }
  if (kind === 'qa') {
    return (
      <>
        <Field data-invalid={!!err('question')}>
          <FieldLabel htmlFor={`q-${index}`}>Question</FieldLabel>
          <Input id={`q-${index}`} {...reg(`sections.${index}.question`)} />
          <FieldError errors={[err('question')]} />
        </Field>
        <Field data-invalid={!!err('answer')}>
          <FieldLabel htmlFor={`a-${index}`}>Answer</FieldLabel>
          <Textarea id={`a-${index}`} rows={3} {...reg(`sections.${index}.answer`)} />
          <FieldError errors={[err('answer')]} />
        </Field>
      </>
    )
  }
  // list_item
  const recipeId = String(form.watch(`sections.${index}.recipeId`) || '')
  return (
    <>
      <div className="grid gap-3 sm:grid-cols-[6rem_1fr]">
        <Field>
          <FieldLabel htmlFor={`rank-${index}`}>Rank</FieldLabel>
          <Input id={`rank-${index}`} type="number" inputMode="numeric" placeholder="1" {...reg(`sections.${index}.rank`)} />
        </Field>
        <Field data-invalid={!!err('heading')}>
          <FieldLabel htmlFor={`head-${index}`}>Heading</FieldLabel>
          <Input id={`head-${index}`} {...reg(`sections.${index}.heading`)} />
          <FieldError errors={[err('heading')]} />
        </Field>
      </div>
      <Field>
        <FieldLabel>Body</FieldLabel>
        <Controller control={form.control} name={`sections.${index}.body`}
          render={({ field }) => <RichTextEditor value={field.value} onChange={field.onChange} />} />
      </Field>
      <Field>
        <FieldLabel>Recipe (optional)</FieldLabel>
        <RecipeChip site={site} recipeId={recipeId} onPick={onPickRecipe}
          onClear={() => form.setValue(`sections.${index}.recipeId`, '', { shouldDirty: true })} />
      </Field>
      <Field>
        <FieldLabel htmlFor={`li-eyebrow-${index}`}>Eyebrow</FieldLabel>
        <Input id={`li-eyebrow-${index}`} placeholder='Overrides "Recipe"' {...reg(`sections.${index}.eyebrow`)} />
      </Field>
    </>
  )
}
```

- [ ] **Step 3: `RecipeChip` (превью выбранного рецепта + «Unknown recipe (deleted?)»)**

```tsx
// web.admin/src/features/articles/RecipeChip.tsx
import { useQuery } from '@tanstack/react-query'
import { ImageOff, X } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listRecipes } from '@/lib/recipes'
import { resolveImageUrl } from '@/lib/images'
import { Button } from '@/components/ui/button'

// Показ выбранного рецепта в секции. Удалённый рецепт → «Unknown recipe (deleted?)»,
// id не чистим молча (паттерн ProductRow).
export function RecipeChip({
  site, recipeId, onPick, onClear,
}: {
  site: SiteConfig
  recipeId: string
  onPick: () => void
  onClear: () => void
}) {
  const { data: recipes } = useQuery({ queryKey: ['recipes', site.slug], queryFn: () => listRecipes(site) })
  const recipe = recipeId ? (recipes ?? []).find((r) => r.id === recipeId) : undefined
  const loaded = recipes !== undefined

  if (!recipeId) {
    return (
      <Button type="button" variant="outline" size="sm" className="w-fit" onClick={onPick}>
        Choose recipe
      </Button>
    )
  }
  return (
    <div className="flex items-center gap-2 rounded-md border px-2 py-1.5">
      {recipe?.hero_image_path ? (
        <img src={resolveImageUrl(site, recipe.hero_image_path)} alt="" loading="lazy"
          className="size-8 shrink-0 rounded border object-cover" />
      ) : (
        <span className="flex size-8 shrink-0 items-center justify-center rounded border bg-muted/30">
          <ImageOff className="size-4 text-muted-foreground" />
        </span>
      )}
      <span className="min-w-0 flex-1 truncate text-sm">
        {recipe ? recipe.title : loaded ? 'Unknown recipe (deleted?)' : 'Loading…'}
      </span>
      <Button type="button" variant="outline" size="sm" onClick={onPick}>Change</Button>
      <Button type="button" variant="ghost" size="icon" className="size-6 text-muted-foreground hover:text-destructive"
        aria-label="Clear recipe" onClick={onClear}>
        <X className="size-3.5" />
      </Button>
    </div>
  )
}
```

- [ ] **Step 4: Проверить наличие shadcn `dropdown-menu`**

Run: `ls web.admin/src/components/ui/dropdown-menu.tsx`
Если отсутствует — добавить через shadcn CLI (не редактировать руками):
`cd web.admin && npx shadcn@latest add dropdown-menu`

- [ ] **Step 5: Сборка и commit**

Run: `npm --prefix web.admin run build`
Expected: OK.

```bash
git add web.admin/src/features/articles/SectionsEditor.tsx web.admin/src/features/articles/RecipeChip.tsx web.admin/src/components/ui/dropdown-menu.tsx
git commit -m "feat(articles): SectionsEditor (6 block types, Add block menu) + RecipeChip"
```

---

## Task 7: Read-also пины (`RelatedEditor`)

**Files:**
- Create: `web.admin/src/features/articles/PostPickerDialog.tsx`
- Create: `web.admin/src/features/articles/RelatedEditor.tsx`

- [ ] **Step 1: `PostPickerDialog` (single-select постов, порт RecipePicker)**

Скопировать `RecipePickerDialog.tsx`, заменить источник на `listArticles` (title +
Draft-бейдж по `is_published`), заголовок «Add post», `onSelect(id)`.

```tsx
// web.admin/src/features/articles/PostPickerDialog.tsx
import { useDeferredValue, useEffect, useMemo, useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { Search } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listArticles } from '@/lib/articles'
import { cn } from '@/lib/utils'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Dialog, DialogBody, DialogContent, DialogFooter, DialogHeader, DialogTitle,
} from '@/components/ui/dialog'

export function PostPickerDialog({
  site, open, onOpenChange, excludeId, onSelect,
}: {
  site: SiteConfig
  open: boolean
  onOpenChange: (open: boolean) => void
  excludeId?: string // текущий пост — нельзя ссылаться на себя
  onSelect: (id: string) => void
}) {
  const [search, setSearch] = useState('')
  const deferred = useDeferredValue(search)
  useEffect(() => { if (open) setSearch('') }, [open])

  const { data: posts, isPending, error } = useQuery({
    queryKey: ['posts', site.slug], queryFn: () => listArticles(site), enabled: open,
  })

  const query = deferred.trim().toLowerCase()
  const visible = useMemo(
    () => (posts ?? []).filter((p) => p.id !== excludeId && (!query || p.title.toLowerCase().includes(query))),
    [posts, query, excludeId],
  )

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-2xl">
        <DialogHeader><DialogTitle>Add post</DialogTitle></DialogHeader>
        <div className="relative w-full max-w-56">
          <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          <Input type="search" placeholder="Search by title…" value={search}
            onChange={(e) => setSearch(e.target.value)} className="h-9 pl-8" />
        </div>
        {error && (
          <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
            Failed to load posts: {error.message}
          </p>
        )}
        <DialogBody>
          {isPending && !error && (
            <div className="flex flex-col gap-1">
              {Array.from({ length: 6 }, (_, i) => <Skeleton key={i} className="h-10 w-full rounded-md" />)}
            </div>
          )}
          {posts && visible.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              {posts.length === 0 ? 'No posts yet.' : 'No posts match.'}
            </p>
          )}
          {visible.length > 0 && (
            <ul className="flex flex-col gap-1">
              {visible.map((p) => (
                <li key={p.id}>
                  <button type="button" onClick={() => { onSelect(p.id); onOpenChange(false) }}
                    className={cn('flex w-full items-center gap-3 rounded-md px-2 py-1.5 text-left transition-colors hover:bg-accent/50')}>
                    <span className="min-w-0 flex-1 truncate text-sm font-medium">{p.title}</span>
                    {!p.is_published && <Badge variant="outline">Draft</Badge>}
                  </button>
                </li>
              ))}
            </ul>
          )}
        </DialogBody>
        <DialogFooter>
          <Button type="button" variant="outline" onClick={() => onOpenChange(false)}>Cancel</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

- [ ] **Step 2: `RelatedEditor` (список пинов recipe|post, ↑/↓/✕)**

```tsx
// web.admin/src/features/articles/RelatedEditor.tsx
import { useState } from 'react'
import { useFieldArray, type UseFormReturn } from 'react-hook-form'
import { ArrowDown, ArrowUp, Plus, X } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { Button } from '@/components/ui/button'
import { RecipeChip } from './RecipeChip'
import { RecipePickerDialog } from './RecipePickerDialog'
import { PostPickerDialog } from './PostPickerDialog'
import { PostTitle } from './PostTitle'
import type { ArticleFormValues } from './articleForm'

// Ручные пины «Read also» (post_related). Пусто → сайт авто-подбирает.
// До 3 пинов сверху авто-выдачи. Каждый пин — рецепт ИЛИ пост (полиморфно).
export function RelatedEditor({
  site, form, currentPostId,
}: {
  site: SiteConfig
  form: UseFormReturn<ArticleFormValues>
  currentPostId?: string
}) {
  const { fields, append, remove, move } = useFieldArray({ control: form.control, name: 'related' })
  const [recipePickerOpen, setRecipePickerOpen] = useState(false)
  const [postPickerOpen, setPostPickerOpen] = useState(false)

  return (
    <div className="flex flex-col gap-2">
      <div className="flex items-center justify-between">
        <span className="text-sm font-medium">Read also (manual pins)</span>
        <span className="text-xs text-muted-foreground">Optional — auto-filled if empty</span>
      </div>

      {fields.length === 0 && (
        <p className="rounded-lg border border-dashed p-4 text-center text-sm text-muted-foreground">
          No manual pins. The site auto-picks related content.
        </p>
      )}

      {fields.map((field, index) => {
        const pin = form.watch(`related.${index}`)
        return (
          <div key={field.id} className="flex items-center gap-2 rounded-md border px-2 py-1.5">
            <span className="w-16 shrink-0 text-xs text-muted-foreground">
              {pin.kind === 'recipe' ? 'Recipe' : 'Post'}
            </span>
            <span className="min-w-0 flex-1 truncate text-sm">
              {pin.kind === 'recipe'
                ? <RecipeChip site={site} recipeId={pin.id} onPick={() => {}} onClear={() => {}} />
                : <PostTitle site={site} postId={pin.id} />}
            </span>
            <Button type="button" variant="ghost" size="icon" className="size-6" aria-label="Move up"
              disabled={index === 0} onClick={() => move(index, index - 1)}>
              <ArrowUp className="size-3.5" />
            </Button>
            <Button type="button" variant="ghost" size="icon" className="size-6" aria-label="Move down"
              disabled={index === fields.length - 1} onClick={() => move(index, index + 1)}>
              <ArrowDown className="size-3.5" />
            </Button>
            <Button type="button" variant="ghost" size="icon" className="size-6 text-muted-foreground hover:text-destructive"
              aria-label="Remove pin" onClick={() => remove(index)}>
              <X className="size-3.5" />
            </Button>
          </div>
        )
      })}

      {fields.length < 3 && (
        <div className="flex gap-2">
          <Button type="button" variant="outline" size="sm" onClick={() => setRecipePickerOpen(true)}>
            <Plus /> Pin recipe
          </Button>
          <Button type="button" variant="outline" size="sm" onClick={() => setPostPickerOpen(true)}>
            <Plus /> Pin post
          </Button>
        </div>
      )}

      <RecipePickerDialog site={site} open={recipePickerOpen} onOpenChange={setRecipePickerOpen}
        onSelect={(id) => append({ kind: 'recipe', id })} />
      <PostPickerDialog site={site} open={postPickerOpen} onOpenChange={setPostPickerOpen}
        excludeId={currentPostId} onSelect={(id) => append({ kind: 'post', id })} />
    </div>
  )
}
```

Примечание: в `RelatedEditor` `RecipeChip` используется только для отображения
(onPick/onClear — пустышки; удаление пина — через ✕ строки). Простой `PostTitle`:

```tsx
// web.admin/src/features/articles/PostTitle.tsx
import { useQuery } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import { listArticles } from '@/lib/articles'

export function PostTitle({ site, postId }: { site: SiteConfig; postId: string }) {
  const { data } = useQuery({ queryKey: ['posts', site.slug], queryFn: () => listArticles(site) })
  const post = (data ?? []).find((p) => p.id === postId)
  return <span className="truncate text-sm">{post ? post.title : 'Unknown post (deleted?)'}</span>
}
```

- [ ] **Step 3: Сборка и commit**

Run: `npm --prefix web.admin run build`
Expected: OK.

```bash
git add web.admin/src/features/articles/PostPickerDialog.tsx web.admin/src/features/articles/RelatedEditor.tsx web.admin/src/features/articles/PostTitle.tsx
git commit -m "feat(articles): Read-also pins editor (RelatedEditor + PostPickerDialog)"
```

---

## Task 8: Форма поста (`ArticleEditPage` + `ArticleForm`)

**Files:**
- Create: `web.admin/src/features/articles/ArticleEditPage.tsx`
- Test: `web.admin/src/features/articles/ArticleEditPage.test.tsx`

- [ ] **Step 1: `ArticleEditPage` (загрузка + монтирование формы) — порт `PostEditPage` структура загрузки**

Взять структуру `PostEditPage` (loading/error/not-found → `ArticleForm`), но данные
через `getArticle`, query-ключ `['posts', site.slug, postId]`, back-link на `../blog`.

```tsx
// web.admin/src/features/articles/ArticleEditPage.tsx (верхняя часть)
import { useEffect, useRef, useState } from 'react'
import { Controller, useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { Link, useBlocker, useNavigate, useOutletContext, useParams } from 'react-router-dom'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { ArrowLeft, Images, Pencil, RotateCcw, Trash2 } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import {
  createArticle, deleteArticle, getArticle, listAuthors, updateArticle,
  type ArticleSection, type ArticleWithRelations, type RelatedPin,
} from '@/lib/articles'
import { resolveImageUrl, toStoragePath } from '@/lib/images'
import { firstFieldErrorMessage } from '@/lib/errors'
import { ImagePickerDialog } from '@/features/products/ImagePickerDialog'
import { ImagePreviewPicker } from '@/components/ImagePreviewPicker'
import {
  articleSchema, toFormValues, toInput, toSections, toRelated,
  type ArticleFormValues,
} from './articleForm'
import { SectionsEditor } from './SectionsEditor'
import { TagsField } from './TagsField'
import { RelatedEditor } from './RelatedEditor'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Switch } from '@/components/ui/switch'
import { Field, FieldError, FieldGroup, FieldLabel } from '@/components/ui/field'
import {
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
} from '@/components/ui/select'
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent,
  AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle,
  AlertDialogTrigger,
} from '@/components/ui/alert-dialog'

export function ArticleEditPage() {
  const site = useOutletContext<SiteConfig>()
  const { postId } = useParams()
  const isNew = !postId

  const { data, isPending, error } = useQuery({
    queryKey: ['posts', site.slug, postId],
    queryFn: () => getArticle(site, postId!),
    enabled: !isNew,
  })

  const loading = !isNew && isPending
  const showForm = !error && !loading && (isNew || data)

  return (
    <div className="flex flex-col gap-4">
      {!showForm && (
        <Link to={`/${site.slug}/blog`} className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground">
          <ArrowLeft className="size-4" /> Back to blog
        </Link>
      )}
      {error && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          Failed to load: {error.message}
        </p>
      )}
      {!error && loading && <p className="text-sm text-muted-foreground">Loading…</p>}
      {!error && !loading && !isNew && !data && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">Post not found.</p>
      )}
      {showForm && <ArticleForm site={site} data={data ?? null} />}
    </div>
  )
}
```

- [ ] **Step 2: `ArticleForm` — состояние, мутации, пересинхронизация**

Порт `PostForm` из `PostEditPage.tsx`. Ключевые отличия: `loadedSectionsRef`
(секции), передача `tagIds`/`related` в create/update, back-link/ключи под avocado.

```tsx
// ArticleEditPage.tsx (продолжение)
function ArticleForm({ site, data }: { site: SiteConfig; data: ArticleWithRelations | null }) {
  const navigate = useNavigate()
  const queryClient = useQueryClient()
  const isNew = data === null
  const leavingRef = useRef(false)
  const loadedSectionsRef = useRef<ArticleSection[]>(data?.sections ?? [])

  const form = useForm<ArticleFormValues>({
    resolver: zodResolver(articleSchema),
    defaultValues: toFormValues(site, data),
  })
  const { errors, isSubmitting, isDirty } = form.formState
  const template = form.watch('template')
  const heroPath = form.watch('hero_image_path').trim()
  const [pickerOpen, setPickerOpen] = useState(false)
  const [seoOpen, setSeoOpen] = useState(Boolean(data?.article.seo_title || data?.article.seo_description))

  const { data: authors } = useQuery({ queryKey: ['authors', site.slug], queryFn: () => listAuthors(site) })

  function openSeo() {
    if (!form.getValues('seo_title')) form.setValue('seo_title', form.getValues('title'))
    if (!form.getValues('seo_description')) form.setValue('seo_description', form.getValues('excerpt'))
    setSeoOpen(true)
  }
  function resetSeo() {
    form.setValue('seo_title', '', { shouldDirty: true })
    form.setValue('seo_description', '', { shouldDirty: true })
    setSeoOpen(false)
  }

  const save = useMutation({
    mutationFn: async (values: ArticleFormValues) => {
      const input = toInput(site, values)
      const sections = toSections(values)
      const tagIds = values.tagIds
      const related: RelatedPin[] = toRelated(values)
      if (isNew) return { created: await createArticle(site, input, sections, tagIds, related), fresh: null }
      const fresh = await updateArticle(site, data.article.id, input, sections, loadedSectionsRef.current, tagIds, related)
      return { created: null, fresh }
    },
    onSuccess: ({ created, fresh }, values) => {
      if (fresh) {
        loadedSectionsRef.current = fresh.sections
        form.reset(toFormValues(site, fresh))
        queryClient.setQueryData(['posts', site.slug, fresh.article.id], fresh)
      } else {
        form.reset(values)
      }
      queryClient.invalidateQueries({ queryKey: ['posts', site.slug], exact: true })
      toast.success(isNew ? 'Post created' : 'Post saved')
      if (created && !leavingRef.current) {
        navigate(`/${site.slug}/blog/${created.id}`, { replace: true })
      }
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to save post'),
  })

  const remove = useMutation({
    mutationFn: () => deleteArticle(site, data!.article.id),
    onSuccess: () => {
      form.reset()
      queryClient.invalidateQueries({ queryKey: ['posts', site.slug] })
      toast.success('Post deleted')
      navigate(`/${site.slug}/blog`, { replace: true })
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to delete post'),
  })

  const blocker = useBlocker(isDirty && !save.isPending && !remove.isPending)
  function saveAndLeave() {
    void form.handleSubmit(
      async (values) => {
        leavingRef.current = true
        try { await save.mutateAsync(values); blocker.proceed?.() }
        catch { blocker.reset?.() }
        finally { leavingRef.current = false }
      },
      () => blocker.reset?.(),
    )()
  }
  useEffect(() => {
    if (!isDirty) return
    const warn = (e: BeforeUnloadEvent) => e.preventDefault()
    window.addEventListener('beforeunload', warn)
    return () => window.removeEventListener('beforeunload', warn)
  }, [isDirty])
  // ... JSX ниже (Step 3)
```

- [ ] **Step 3: `ArticleForm` — JSX (sticky-панель + поля + Type-зависимые + секции + SEO + Read-also)**

```tsx
  return (
    <div className="flex flex-col gap-4">
      <div className="sticky top-0 z-10 flex items-center gap-3 border-b bg-background py-2">
        <Link to={`/${site.slug}/blog`} className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground">
          <ArrowLeft className="size-4" /> Back to blog
        </Link>
        <div className="ml-auto flex items-center gap-3">
          <Controller control={form.control} name="is_published"
            render={({ field }) => (
              <label htmlFor="is_published" className="flex items-center gap-2 text-sm">
                <Switch id="is_published" checked={field.value} onCheckedChange={field.onChange} />
                {field.value ? 'Published' : 'Draft'}
              </label>
            )} />
          <Button type="submit" form="article-form" disabled={isSubmitting || save.isPending}>
            {save.isPending ? 'Saving…' : 'Save'}
          </Button>
          {!isNew && (
            <AlertDialog>
              <AlertDialogTrigger asChild>
                <Button type="button" variant="destructive" disabled={remove.isPending}>
                  <Trash2 /> {remove.isPending ? 'Deleting…' : 'Delete'}
                </Button>
              </AlertDialogTrigger>
              <AlertDialogContent>
                <AlertDialogHeader>
                  <AlertDialogTitle>Delete this post?</AlertDialogTitle>
                  <AlertDialogDescription>
                    “{data.article.title}” and all its content blocks will be permanently deleted.
                  </AlertDialogDescription>
                </AlertDialogHeader>
                <AlertDialogFooter>
                  <AlertDialogCancel>Cancel</AlertDialogCancel>
                  <AlertDialogAction variant="destructive" onClick={() => remove.mutate()}>Delete</AlertDialogAction>
                </AlertDialogFooter>
              </AlertDialogContent>
            </AlertDialog>
          )}
        </div>
      </div>

      <form id="article-form"
        onSubmit={form.handleSubmit(
          (values) => save.mutate(values),
          (errs) => toast.error(firstFieldErrorMessage(errs) ?? 'Please fix the highlighted fields'),
        )}
        noValidate className="flex max-w-3xl flex-col gap-6">
        <div className="grid gap-6 md:grid-cols-[16rem_1fr]">
          <div>
            <ImagePreviewPicker
              url={heroPath ? resolveImageUrl(site, heroPath) : null}
              alt={form.watch('title') || 'Hero image'} aspect="video" objectFit="cover"
              onPick={() => setPickerOpen(true)} />
            {data && (
              <p className="mt-2 truncate text-xs text-muted-foreground" title={data.article.slug}>
                Slug: {data.article.slug}
              </p>
            )}
          </div>
          <FieldGroup>
            <Field data-invalid={!!errors.title}>
              <FieldLabel htmlFor="title">Title</FieldLabel>
              <Input id="title" aria-invalid={!!errors.title} {...form.register('title')} />
              <FieldError errors={[errors.title]} />
            </Field>

            <Field>
              <FieldLabel htmlFor="template">Type</FieldLabel>
              <Controller control={form.control} name="template"
                render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger id="template" className="w-56"><SelectValue /></SelectTrigger>
                    <SelectContent>
                      <SelectItem value="essay">Essay</SelectItem>
                      <SelectItem value="interview">Interview</SelectItem>
                      <SelectItem value="roundup">Roundup</SelectItem>
                    </SelectContent>
                  </Select>
                )} />
              <p className="text-xs text-muted-foreground">Changes the hero layout only.</p>
            </Field>

            <Field>
              <FieldLabel htmlFor="excerpt">Excerpt</FieldLabel>
              <Textarea id="excerpt" rows={3} {...form.register('excerpt')} />
            </Field>

            <Field>
              <FieldLabel htmlFor="hero_image_path">Hero image</FieldLabel>
              <div className="flex gap-2">
                <Input id="hero_image_path" placeholder="flat key in bucket or external URL" {...form.register('hero_image_path')} />
                <Button type="button" variant="outline" onClick={() => setPickerOpen(true)}>
                  <Images /> Gallery
                </Button>
              </div>
            </Field>
          </FieldGroup>
        </div>

        <FieldGroup>
          {/* Type-зависимые поля */}
          {(template === 'interview' || template === 'roundup') && (
            <Field>
              <FieldLabel htmlFor="subtitle">Subtitle</FieldLabel>
              <Input id="subtitle" {...form.register('subtitle')} />
            </Field>
          )}
          {template === 'roundup' && (
            <Field>
              <FieldLabel htmlFor="hero_caption">Hero caption</FieldLabel>
              <Input id="hero_caption" {...form.register('hero_caption')} />
            </Field>
          )}

          <Field>
            <FieldLabel htmlFor="author_id">Author</FieldLabel>
            <Controller control={form.control} name="author_id"
              render={({ field }) => (
                <Select value={field.value || '__none__'} onValueChange={(v) => field.onChange(v === '__none__' ? '' : v)}>
                  <SelectTrigger id="author_id" className="w-56"><SelectValue placeholder="No author" /></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="__none__">No author</SelectItem>
                    {(authors ?? []).map((a) => <SelectItem key={a.id} value={a.id}>{a.name}</SelectItem>)}
                  </SelectContent>
                </Select>
              )} />
          </Field>

          <Field>
            <FieldLabel>Tags</FieldLabel>
            <Controller control={form.control} name="tagIds"
              render={({ field }) => <TagsField site={site} value={field.value} onChange={field.onChange} />} />
            <p className="text-xs text-muted-foreground">Order = hero eyebrow order.</p>
          </Field>

          <div className="grid gap-6 sm:grid-cols-2">
            <Field className="max-w-xs">
              <FieldLabel htmlFor="read_minutes">Read time (min)</FieldLabel>
              <Input id="read_minutes" type="number" inputMode="numeric" placeholder="Optional" {...form.register('read_minutes')} />
            </Field>
            <Field data-invalid={!!errors.published_at} className="max-w-xs">
              <FieldLabel htmlFor="published_at">Published at</FieldLabel>
              <Input id="published_at" type="datetime-local" aria-invalid={!!errors.published_at} {...form.register('published_at')} />
              <FieldError errors={[errors.published_at]} />
            </Field>
          </div>

          <SectionsEditor site={site} form={form} />

          <RelatedEditor site={site} form={form} currentPostId={data?.article.id} />

          {!seoOpen ? (
            <Field>
              <FieldLabel>SEO</FieldLabel>
              <div className="flex items-center justify-between gap-3 rounded-lg border border-dashed p-3">
                <p className="text-sm text-muted-foreground">Search engines use the post title and excerpt.</p>
                <Button type="button" variant="ghost" size="sm" onClick={openSeo}><Pencil /> Customize</Button>
              </div>
            </Field>
          ) : (
            <>
              <div className="flex items-center justify-between">
                <span className="text-sm font-medium">SEO</span>
                <Button type="button" variant="ghost" size="sm" onClick={resetSeo}><RotateCcw /> Use defaults</Button>
              </div>
              <Field>
                <FieldLabel htmlFor="seo_title">SEO title</FieldLabel>
                <Input id="seo_title" {...form.register('seo_title')} />
              </Field>
              <Field>
                <FieldLabel htmlFor="seo_description">SEO description</FieldLabel>
                <Textarea id="seo_description" rows={3} {...form.register('seo_description')} />
              </Field>
            </>
          )}
        </FieldGroup>
      </form>

      <ImagePickerDialog site={site} open={pickerOpen} onOpenChange={setPickerOpen}
        currentPath={heroPath ? toStoragePath(site, heroPath) : null}
        onSelect={(path) => form.setValue('hero_image_path', resolveImageUrl(site, path), { shouldDirty: true })} />

      <AlertDialog open={blocker.state === 'blocked'} onOpenChange={(open) => { if (!open) blocker.reset?.() }}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Discard unsaved changes?</AlertDialogTitle>
            <AlertDialogDescription>You have unsaved changes. If you leave this page, they will be lost.</AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={save.isPending}>Stay</AlertDialogCancel>
            <AlertDialogAction variant="destructive" onClick={() => blocker.proceed?.()}>Discard</AlertDialogAction>
            <Button type="button" onClick={saveAndLeave} disabled={save.isPending}>{save.isPending ? 'Saving…' : 'Save'}</Button>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  )
}
```

- [ ] **Step 4: Тест формы — условная видимость Type-полей + Save-путь**

Порт `ProductEditPage.test.tsx` (data-router через `createMemoryRouter`, моки `sonner`
и `@/lib/articles`). Проверить: (а) для essay нет Subtitle/Hero caption; для interview
есть Subtitle; для roundup есть и Subtitle, и Hero caption; (б) Save зовёт
`updateArticle` c правильными аргументами.

```tsx
// web.admin/src/features/articles/ArticleEditPage.test.tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import { createMemoryRouter, Outlet, RouterProvider } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import { getArticle, listAuthors, updateArticle } from '@/lib/articles'
import { listTags } from '@/lib/tags'
import { listRecipes } from '@/lib/recipes'
import { ArticleEditPage } from './ArticleEditPage'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
vi.mock('@/lib/articles', async (io) => {
  const actual = await io<typeof import('@/lib/articles')>()
  return { ...actual, getArticle: vi.fn(), updateArticle: vi.fn(), listAuthors: vi.fn() }
})
vi.mock('@/lib/tags', () => ({ listTags: vi.fn() }))
vi.mock('@/lib/recipes', () => ({ listRecipes: vi.fn() }))

const site = {
  slug: 'avocado-kiss', label: 'Avocado Kiss', projectUrl: 'https://demo.supabase.co',
  anonKey: 'k', schema: 'avocado_kiss', bucket: 'avocado-kiss-photos',
} as SiteConfig

function article(template: 'essay' | 'interview' | 'roundup') {
  return {
    article: {
      id: 'a1', created_at: '', updated_at: '', published_at: '2026-08-08T10:00:00.000Z',
      is_published: true, slug: 'post', title: 'Post', template, author_id: null,
      excerpt: null, subtitle: null, read_minutes: null, hero_image_path: null,
      hero_caption: null, seo_title: null, seo_description: null,
    },
    author: null, sections: [], tagIds: [], related: [],
  }
}

function renderAt(url: string) {
  const qc = new QueryClient()
  const router = createMemoryRouter(
    [{ element: <Outlet context={site} />, children: [{ path: '/:siteSlug/blog/:postId', element: <ArticleEditPage /> }] }],
    { initialEntries: [url] },
  )
  render(<QueryClientProvider client={qc}><RouterProvider router={router} /></QueryClientProvider>)
}

describe('<ArticleEditPage /> template-dependent fields', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    vi.mocked(listAuthors).mockResolvedValue([])
    vi.mocked(listTags).mockResolvedValue([])
    vi.mocked(listRecipes).mockResolvedValue([])
  })

  it('essay: no Subtitle, no Hero caption', async () => {
    vi.mocked(getArticle).mockResolvedValue(article('essay') as never)
    renderAt('/avocado-kiss/blog/a1')
    await screen.findByLabelText('Title')
    expect(screen.queryByLabelText('Subtitle')).not.toBeInTheDocument()
    expect(screen.queryByLabelText('Hero caption')).not.toBeInTheDocument()
  })
  it('interview: Subtitle shown, Hero caption hidden', async () => {
    vi.mocked(getArticle).mockResolvedValue(article('interview') as never)
    renderAt('/avocado-kiss/blog/a1')
    expect(await screen.findByLabelText('Subtitle')).toBeInTheDocument()
    expect(screen.queryByLabelText('Hero caption')).not.toBeInTheDocument()
  })
  it('roundup: both Subtitle and Hero caption shown', async () => {
    vi.mocked(getArticle).mockResolvedValue(article('roundup') as never)
    renderAt('/avocado-kiss/blog/a1')
    expect(await screen.findByLabelText('Subtitle')).toBeInTheDocument()
    expect(screen.getByLabelText('Hero caption')).toBeInTheDocument()
  })
  it('Save calls updateArticle', async () => {
    vi.mocked(getArticle).mockResolvedValue(article('essay') as never)
    vi.mocked(updateArticle).mockResolvedValue(article('essay') as never)
    renderAt('/avocado-kiss/blog/a1')
    await screen.findByLabelText('Title')
    fireEvent.change(screen.getByLabelText('Title'), { target: { value: 'Post 2' } })
    fireEvent.click(screen.getByRole('button', { name: 'Save' }))
    await waitFor(() => expect(updateArticle).toHaveBeenCalled())
  })
})
```

- [ ] **Step 5: Прогнать тесты, сборка, commit**

Run: `npm --prefix web.admin test -- ArticleEditPage.test` → PASS.
Run: `npm --prefix web.admin run build` → OK.

```bash
git add web.admin/src/features/articles/ArticleEditPage.tsx web.admin/src/features/articles/ArticleEditPage.test.tsx
git commit -m "feat(articles): ArticleEditPage/ArticleForm (template select + conditional fields + sections/tags/related)"
```

---

## Task 9: Список постов (`ArticlesPage`) + папки

**Files:**
- Create: `web.admin/src/features/articles/ArticlesPage.tsx`

- [ ] **Step 1: Порт `PostsPage` под articles**

Скопировать `web.admin/src/features/posts/PostsPage.tsx` и применить изменения:
- убрать проп `section`; заголовок «Blog»; `folderSection: 'posts'` фикс;
- источник — `listArticles(site)` + `deleteArticle`;
- query-ключ `postsKey = ['posts', site.slug]` (без postType);
- `useFolders({ site, section: 'posts', items: posts, contentKey: postsKey, itemNoun: 'post' })`;
- `useBulkDelete({ deleteOne: (id) => deleteArticle(site, id), contentKey: postsKey, itemNoun: 'post', onDeleted: clearSelection })`;
- фильтр `visible` — идентичен (title + status), без изменений.

```tsx
// web.admin/src/features/articles/ArticlesPage.tsx (сигнатуры-отличия)
import { deleteArticle, listArticles } from '@/lib/articles'
// ... остальные импорты идентичны PostsPage (Badge, Button, Checkbox, Input, Skeleton,
//     Select*, FoldersPanel, MoveToFolderMenu, BulkDeleteButton, FolderActionsBar,
//     useBulkDelete, SelectionBar, intersectSelected, useFolders, formatDate)

const ALL = '__all__'

export function ArticlesPage() {
  const site = useOutletContext<SiteConfig>()
  const [search, setSearch] = useState('')
  const [status, setStatus] = useState(ALL)
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const postsKey = ['posts', site.slug]
  const { data: posts, isPending, error } = useQuery({
    queryKey: postsKey, queryFn: () => listArticles(site),
  })

  const {
    folders, foldersError, folder, activeFolderId, selectFolder, counts, matchesFolder,
    selectedIds, selectionMode, startSelection, exitSelection, selectRange, clearSelection,
    moveItems, isMoving,
  } = useFolders({ site, section: 'posts', items: posts, contentKey: postsKey, itemNoun: 'post' })

  const removeMany = useBulkDelete<string>({
    deleteOne: (id) => deleteArticle(site, id), contentKey: postsKey, itemNoun: 'post', onDeleted: clearSelection,
  })
  // ... тело JSX идентично PostsPage: heading = "Blog"; secion.folderSection → 'posts';
  //     все place, где использовался section.heading/folderSection/basePath — заменить
  //     на литералы ("Blog", 'posts'); "New post" → Link to="new".
}
```

Остальной JSX (шапка, поиск/фильтр, папки, список строк-ссылок, bulk-бар) —
**полностью как в `PostsPage.tsx`**, заменив: `section.heading` → `'Blog'`,
`section.folderSection` → `'posts'`, `section.postType` (в ключах) убрать,
`listPosts(site, section.postType)` → `listArticles(site)`,
`deletePost` → `deleteArticle`.

- [ ] **Step 2: Сборка**

Run: `npm --prefix web.admin run build`
Expected: OK.

- [ ] **Step 3: Commit**

```bash
git add web.admin/src/features/articles/ArticlesPage.tsx
git commit -m "feat(articles): ArticlesPage (list + folders + bulk)"
```

---

## Task 10: Роутинг-диспетчер + реестр сайтов + навигация

**Files:**
- Create: `web.admin/src/features/articles/BlogRoutes.tsx`
- Modify: `web.admin/src/App.tsx:51-53`
- Modify: `web.admin/src/config/sites.ts:39`

- [ ] **Step 1: Диспетчер по `site.schema`**

```tsx
// web.admin/src/features/articles/BlogRoutes.tsx
import { useOutletContext } from 'react-router-dom'
import type { SiteConfig } from '@/config/sites'
import { PostsPage } from '@/features/posts/PostsPage'
import { PostEditPage } from '@/features/posts/PostEditPage'
import { BLOG_SECTION } from '@/features/posts/sections'
import { ArticlesPage } from './ArticlesPage'
import { ArticleEditPage } from './ArticleEditPage'

// Модель блога зависит от сайта: avocado_kiss — единая таблица post_sections + 3
// шаблона (ArticlesPage/ArticleEditPage); прочие (cozycorner) — две таблицы секций
// (PostsPage/PostEditPage). URL /blog общий; выбор реализации — по схеме.
function isAvocado(site: SiteConfig) {
  return site.schema === 'avocado_kiss'
}

export function BlogListRoute() {
  const site = useOutletContext<SiteConfig>()
  return isAvocado(site) ? <ArticlesPage /> : <PostsPage section={BLOG_SECTION} />
}

export function BlogEditRoute() {
  const site = useOutletContext<SiteConfig>()
  return isAvocado(site) ? <ArticleEditPage /> : <PostEditPage section={BLOG_SECTION} />
}
```

- [ ] **Step 2: Подключить диспетчер в `App.tsx`**

Заменить три строки (текущие 51–53):

```tsx
// было:
//   <Route path="blog" element={<PostsPage section={BLOG_SECTION} />} />
//   <Route path="blog/new" element={<PostEditPage section={BLOG_SECTION} />} />
//   <Route path="blog/:postId" element={<PostEditPage section={BLOG_SECTION} />} />
// стало:
<Route path="blog" element={<BlogListRoute />} />
<Route path="blog/new" element={<BlogEditRoute />} />
<Route path="blog/:postId" element={<BlogEditRoute />} />
```

И заменить импорт `PostsPage`/`PostEditPage`/`BLOG_SECTION` (строки 18–20) на:

```tsx
import { BlogListRoute, BlogEditRoute } from '@/features/articles/BlogRoutes'
// SEO_POSTS_SECTION оставить (используется в seo-posts роутах):
import { PostsPage } from '@/features/posts/PostsPage'
import { PostEditPage } from '@/features/posts/PostEditPage'
import { SEO_POSTS_SECTION } from '@/features/posts/sections'
```

(SEO Posts роуты 54–59 не трогаем — они и дальше используют `PostsPage`/`PostEditPage`
напрямую с `SEO_POSTS_SECTION`.)

- [ ] **Step 3: Включить разделы `blog` и `pages` для avocado**

`web.admin/src/config/sites.ts:39` — заменить массив `sections`:

```ts
    sections: ["media", "products", "categories", "brands", "blog", "pages"],
```

(Nav-пункты «Blog» и «Pages» уже есть в `SiteLayout` NAV_ITEMS — правок в нём не
требуется; allowlist их откроет.)

- [ ] **Step 4: Сборка + прогон всех тестов**

Run: `npm --prefix web.admin run build` → OK.
Run: `npm --prefix web.admin test` → все зелёные (включая cozycorner-тесты постов —
диспетчер не меняет их поведение).

- [ ] **Step 5: Commit**

```bash
git add web.admin/src/App.tsx web.admin/src/config/sites.ts web.admin/src/features/articles/BlogRoutes.tsx
git commit -m "feat(articles): schema-dispatched /blog route + enable blog/pages for avocado-kiss"
```

---

## Task 11: Баннер архива /blog (раздел Pages для avocado) — проверка

Раздел Pages уже реализован и site-agnostic. Task 10 уже включил `"pages"` в
`sections`. Здесь — только проверка, что строка `pages.blog` редактируется корректно
для avocado_kiss (без хардкода cozy-строк).

**Files:**
- Verify (read-only): `web.admin/src/features/pages/PagesPage.tsx`, `PageEditPage.tsx`, `src/lib/pages.ts`

- [ ] **Step 1: Проверить, что `PagesPage` читает строки из БД текущей схемы**

Прочитать `src/lib/pages.ts` `listPages`/`getPage`. Убедиться, что список строк
приходит из `getDb(site).from('pages')` (а не из хардкод-списка cozy). Если хардкод —
это дефект: завести отдельным баг-репортом (правка вне объёма этого плана; либо
согласовать мелкую правку с пользователем).

- [ ] **Step 2: Ручная проверка (dev-сервер)**

Run: `npm --prefix web.admin run dev`, открыть `/avocado-kiss/pages`.
Expected: в списке есть строки `home`, `shop`, `blog`. Открыть `blog` → редактируются
hero-поля (`hero_eyebrow`/`hero_title`/`hero_description`/`hero_image_path`) + SEO,
сохранение пишет в `avocado_kiss.pages` строку `slug='blog'`.

- [ ] **Step 3: Зафиксировать результат в отчёте**

Если всё работает — отметить в отчёте «баннер /blog: OK через раздел Pages».
Если нашёлся хардкод/дефект — вынести багом, не чинить молча.

---

## Task 12: Финальная проверка, e2e-предложение, обновление доков

**Files:**
- Modify: `platform-docs/admin-panel/status.md`, `platform-docs/admin-panel/blog-avocado-kiss.md`

- [ ] **Step 1: Прод-проверки**

Run: `npm --prefix web.admin run build` → OK.
Run: `npm --prefix web.admin run lint` → без новых ошибок (2 известных shadcn-варнинга ок).
Run: `npm --prefix web.admin test` → всё зелёное.

- [ ] **Step 2: Ручная приёмка под админом (dev)**

По контракту §6: создать по одному посту каждого типа (essay/interview/roundup),
добавить/переупорядочить блоки всех 6 типов, добавить теги (проверить порядок эйброу),
выбрать автора, поставить 1–2 Read-also пина, сохранить как черновик, затем
опубликовать. Проверить на сайте avocado.kiss (`/blog`, `/blog/<slug>`) корректный
hero по типу и рендер блоков. Проверить папки: создать папку, переместить пост, bulk-delete.

- [ ] **Step 3: Предложить e2e (Playwright, порт 3100, live Supabase)**

Не писать автоматически (протокол: e2e только по явной просьбе). В отчёте предложить
сценарий «создать пост каждого типа → добавить/переставить блоки → опубликовать →
проверить на сайте». Реестр порта — корневой AGENTS.md.

- [ ] **Step 4: Обновить документацию**

- `platform-docs/admin-panel/blog-avocado-kiss.md` — снять пометку «раздел ещё не
  построен (Phase B)», отметить реализованные решения (отдельная feature-папка,
  Add block + выбор типа, авторы select-only, папки через миграцию 0015).
- `platform-docs/admin-panel/status.md` — добавить строку о готовности раздела Blog
  для avocado-kiss и любые отклонения от спеки.
- Если Task 11 нашёл дефект Pages — записать в status.md.

- [ ] **Step 5: Commit доков**

```bash
git add platform-docs/admin-panel/blog-avocado-kiss.md platform-docs/admin-panel/status.md
git commit -m "docs: avocado-kiss blog admin section built (contract + status)"
```

---

## Self-review чек-лист (для исполнителя перед завершением)

- [ ] Каждый из 6 типов блоков: есть в `emptySection`, `sectionToValues`, `rowToSection`,
  `toSectionFormValue`, `SectionFields`, Zod-схеме — имена полей совпадают во всех.
- [ ] `sectionId` (не `id`) используется во всех form-путях секций.
- [ ] `is_published` задаётся явно при create (БД default `true`).
- [ ] slug/updated_at нигде не отправляются на запись.
- [ ] Пересинхронизация формы после Save (`loadedSectionsRef` + `form.reset(toFormValues)`).
- [ ] Диспетчер не сломал блог cozycorner (тесты постов зелёные).
- [ ] Папки работают только после миграции Task 0.
- [ ] UI-текст английский; картинки через resolveImageUrl/toStoragePath.
