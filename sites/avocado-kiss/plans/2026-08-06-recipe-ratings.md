# Recipe Ratings Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let anonymous visitors rate a recipe 1–5 stars; show a floored aggregate (min 4.1) as a badge under the title and an interactive picker at the end of the recipe, with the real votes stored truthfully and admin-seedable.

**Architecture:** Real votes and an admin seed baseline live as columns on `avocado_kiss.recipes`; the displayed average/count are DB generated columns applying the floor. A per-vote table `recipe_ratings` records each vote. Writes go through a Next.js Route Handler that verifies a Cloudflare Turnstile token and calls a `service_role`-only RPC (the anon key stays read-only). The client remembers its own vote in localStorage.

**Tech Stack:** Next.js 16 (App Router, RSC + Route Handler), TypeScript, Supabase (Postgres, schema `avocado_kiss`), Vitest + RTL, Cloudflare Turnstile. Spec: `platform-docs/sites/avocado-kiss/specs/2026-08-06-recipe-ratings-design.md`.

**Execution gates (read before starting):**
- **No git commits / push** during execution — deferred until the user explicitly asks. Ignore the "Commit" convention; each task ends at "tests green".
- **Task 1 (apply migration to live Supabase)** touches the shared DB — get explicit user confirmation before applying. Writing the migration file is fine; applying it is gated.
- **Tasks 6, 12 live-run** need `TURNSTILE_SECRET_KEY` / `SUPABASE_SECRET_KEY` / `NEXT_PUBLIC_TURNSTILE_SITE_KEY` in env, which are not present locally. Those tasks' **unit tests (mocked) are the verifiable deliverable**; live integration is verified by the user once keys are provisioned.
- Next.js here is NOT the version in training data — **before writing the Route Handler (Task 6), read `node_modules/next/dist/docs/` on Route Handlers** (per AGENTS.md).

---

## File Structure

New:
- `avocado.kiss/supabase/migrations/0012_recipe_ratings.sql` — schema: recipe columns, generated display columns, `recipe_ratings` table, RLS, `rate_recipe` RPC.
- `avocado.kiss/lib/rating.ts` — pure display/format helpers (`formatRatingCount`), `RATING_MIN` constant. `server-only`-free (used by client + server).
- `avocado.kiss/lib/rating-client.ts` — `'use client'`-safe localStorage helpers (`getClientId`, `getRated`, `setRated`).
- `avocado.kiss/lib/supabase/service.ts` — service-role client factory (server-only).
- `avocado.kiss/app/api/recipes/[slug]/rate/route.ts` — POST handler (Turnstile verify + RPC).
- `avocado.kiss/components/icons/StarIcon.tsx` — star glyph (CSS-fillable).
- `avocado.kiss/components/RatingBadge.tsx` + `.module.css` — read-only badge (RSC).
- `avocado.kiss/components/RatingSection.tsx` + `.module.css` — interactive picker (`'use client'`).

Modified:
- `avocado.kiss/lib/types.ts` — extend `Recipe`.
- `avocado.kiss/app/recipes/[slug]/page.tsx` — render badge + section + JSON-LD.
- `avocado.kiss/app/globals.css` — new tokens if needed.
- `avocado.kiss/.env.example`, `avocado.kiss/AGENTS.md` — env policy.
- `platform-docs/database/schema.md`, `platform-docs/sites/avocado-kiss.md` — docs.

Test files (co-located per project convention — mirror existing `*.test.ts(x)` placement; confirm by checking an existing test in the repo before Task 3):
- `lib/rating.test.ts`, `lib/rating-client.test.ts`
- `app/api/recipes/[slug]/rate/route.test.ts`
- `components/RatingBadge.test.tsx`, `components/RatingSection.test.tsx`

---

## Task 0: Recon (no code)

- [ ] **Step 1: Learn the repo's test layout and mocks.**

Run: `cd avocado.kiss && ls vitest.config.ts vitest.setup.ts && sed -n '1,60p' vitest.setup.ts`
Then find one existing component test and one lib test to copy conventions from:
Run: `git -C avocado.kiss ls-files '*.test.ts' '*.test.tsx' | head`
Note: how `next/image` is mocked, where tests live (co-located vs `__tests__`), and how the Supabase client is mocked. Follow whatever you find. Also read `platform-docs/methodology/testing.md`.

- [ ] **Step 2: Confirm the Turnstile + secret env story.** Confirm `.env.local` does not already carry these keys; they will be added by the user. No action, just awareness for Tasks 6/12.

---

## Task 1: Database migration (GATED — confirm before applying)

**Files:**
- Create: `avocado.kiss/supabase/migrations/0012_recipe_ratings.sql`

- [ ] **Step 1: Write the migration.**

```sql
-- 0012_recipe_ratings.sql — recipe star ratings (spec 2026-08-06-recipe-ratings)
set search_path = '';

-- 1. Real votes + admin seed baseline + per-recipe display floor on recipes.
alter table avocado_kiss.recipes
  add column if not exists ratings_count      integer      not null default 0,
  add column if not exists ratings_sum        integer      not null default 0,
  add column if not exists seed_count         integer      not null default 0,
  add column if not exists seed_sum           integer      not null default 0,
  add column if not exists min_display_rating numeric(2,1) not null default 4.1;

do $$ begin
  alter table avocado_kiss.recipes
    add constraint recipes_min_display_rating_range
      check (min_display_rating >= 1 and min_display_rating <= 5);
exception when duplicate_object then null; end $$;

-- 2. Generated display columns (single source of the display formula).
alter table avocado_kiss.recipes
  add column if not exists display_count integer
    generated always as (seed_count + ratings_count) stored;
alter table avocado_kiss.recipes
  add column if not exists display_avg numeric(2,1)
    generated always as (
      case when (seed_count + ratings_count) = 0 then min_display_rating
      else greatest(
        min_display_rating,
        round((seed_sum + ratings_sum)::numeric / (seed_count + ratings_count), 1)
      ) end
    ) stored;

-- 3. Per-vote table.
create table if not exists avocado_kiss.recipe_ratings (
  id         uuid primary key default gen_random_uuid(),
  recipe_id  uuid not null references avocado_kiss.recipes(id) on delete cascade,
  value      smallint not null check (value between 1 and 5),
  client_id  text,
  created_at timestamptz not null default now()
);
create index if not exists recipe_ratings_recipe_idx
  on avocado_kiss.recipe_ratings (recipe_id, created_at desc);

alter table avocado_kiss.recipe_ratings enable row level security;

-- Admin read + write (existing is_admin() whitelist pattern).
drop policy if exists "Admin all recipe_ratings" on avocado_kiss.recipe_ratings;
create policy "Admin all recipe_ratings" on avocado_kiss.recipe_ratings
  for all to authenticated using (public.is_admin()) with check (public.is_admin());

-- 4. Write RPC — service_role only (anon key is public; must not reach this).
create or replace function avocado_kiss.rate_recipe(
  p_slug text, p_value integer, p_client_id text
) returns table (display_avg numeric, display_count integer)
language plpgsql security definer set search_path = '' as $$
declare v_id uuid;
begin
  if p_value < 1 or p_value > 5 then
    raise exception 'value out of range';
  end if;
  select id into v_id from avocado_kiss.recipes
    where slug = p_slug and is_published = true;
  if v_id is null then
    raise exception 'recipe not found';
  end if;
  insert into avocado_kiss.recipe_ratings (recipe_id, value, client_id)
    values (v_id, p_value, p_client_id);
  update avocado_kiss.recipes
    set ratings_count = ratings_count + 1,
        ratings_sum   = ratings_sum + p_value
    where id = v_id;
  return query
    select r.display_avg, r.display_count
    from avocado_kiss.recipes r where r.id = v_id;
end $$;

revoke all on function avocado_kiss.rate_recipe(text, integer, text) from public, anon, authenticated;
grant execute on function avocado_kiss.rate_recipe(text, integer, text) to service_role;
```

- [ ] **Step 2: (GATED) Apply to Supabase.** STOP and get user confirmation. Then apply via the Supabase MCP `apply_migration` (name `0012_recipe_ratings`) against the avocado.kiss project. If the stored generated column with the nested `case` is rejected by the DB, fall back to a view `avocado_kiss.recipes_with_rating` exposing `display_avg`/`display_count` and repoint `fetchRecipeBySlug` (note the choice in the migration + spec §4.1).

- [ ] **Step 3: Verify the schema.** Via MCP `execute_sql`:
Run: `select display_avg, display_count, min_display_rating from avocado_kiss.recipes limit 1;`
Expected: columns exist; a recipe with 0 votes/seed shows `display_avg = 4.1`, `display_count = 0`.
Run: `select proname from pg_proc where proname = 'rate_recipe';` → one row.

---

## Task 2: Extend the `Recipe` type

**Files:**
- Modify: `avocado.kiss/lib/types.ts` (the `Recipe` type, lib/types.ts:1-20)

- [ ] **Step 1: Add the fields.** Insert after `steps: string[];` (before `seo_title`) — or anywhere inside the type; keep comments in English to match new code (existing comments are RU, leave them):

```ts
  ratings_count: number;      // real votes count (read-only aggregate)
  ratings_sum: number;        // sum of real vote values
  seed_count: number;         // admin baseline count
  seed_sum: number;           // admin baseline sum
  min_display_rating: number; // per-recipe display floor (default 4.1)
  display_count: number;      // generated: seed_count + ratings_count
  display_avg: number;        // generated: floored displayed average
```

- [ ] **Step 2: Typecheck.**
Run: `cd avocado.kiss && npx tsc -p tsconfig.json --noEmit`
Expected: no new errors (fields are additive; `select("*")` returns them).

---

## Task 3: Rating display/format helper (`lib/rating.ts`)

**Files:**
- Create: `avocado.kiss/lib/rating.ts`
- Test: `avocado.kiss/lib/rating.test.ts`

- [ ] **Step 1: Write the failing test.**

```ts
import { describe, it, expect } from "vitest";
import { RATING_MIN, formatRatingCount, ratingLabel } from "./rating";

describe("rating helpers", () => {
  it("RATING_MIN is 4.1", () => {
    expect(RATING_MIN).toBe(4.1);
  });
  it("formats singular and plural counts", () => {
    expect(formatRatingCount(0)).toBe("0 Ratings");
    expect(formatRatingCount(1)).toBe("1 Rating");
    expect(formatRatingCount(3)).toBe("3 Ratings");
  });
  it("builds an a11y label", () => {
    expect(ratingLabel(4.3, 3)).toBe("Rated 4.3 out of 5, 3 Ratings");
  });
});
```

- [ ] **Step 2: Run it — expect FAIL** (module not found).
Run: `cd avocado.kiss && npx vitest run lib/rating.test.ts`

- [ ] **Step 3: Implement.**

```ts
// Pure rating display helpers. Display math lives in the DB; this only formats.
export const RATING_MIN = 4.1;

export function formatRatingCount(count: number): string {
  return `${count} ${count === 1 ? "Rating" : "Ratings"}`;
}

export function ratingLabel(avg: number, count: number): string {
  return `Rated ${avg.toFixed(1)} out of 5, ${formatRatingCount(count)}`;
}
```

- [ ] **Step 4: Run it — expect PASS.**
Run: `cd avocado.kiss && npx vitest run lib/rating.test.ts`

---

## Task 4: localStorage client helper (`lib/rating-client.ts`)

**Files:**
- Create: `avocado.kiss/lib/rating-client.ts`
- Test: `avocado.kiss/lib/rating-client.test.ts`

- [ ] **Step 1: Write the failing test.** (jsdom provides `localStorage`; stub `crypto.randomUUID`.)

```ts
import { describe, it, expect, beforeEach, vi } from "vitest";
import { getClientId, getRated, setRated } from "./rating-client";

beforeEach(() => {
  localStorage.clear();
  vi.stubGlobal("crypto", { randomUUID: () => "uuid-1" });
});

describe("rating-client", () => {
  it("creates and persists a client id", () => {
    expect(getClientId()).toBe("uuid-1");
    expect(localStorage.getItem("avk_client_id")).toBe("uuid-1");
  });
  it("reuses an existing client id", () => {
    localStorage.setItem("avk_client_id", "existing");
    expect(getClientId()).toBe("existing");
  });
  it("stores and reads the per-recipe rating", () => {
    expect(getRated("r1")).toBeNull();
    setRated("r1", 4);
    expect(getRated("r1")).toBe(4);
  });
  it("ignores corrupt stored values", () => {
    localStorage.setItem("avk_rated_r1", "not-a-number");
    expect(getRated("r1")).toBeNull();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL.**
Run: `cd avocado.kiss && npx vitest run lib/rating-client.test.ts`

- [ ] **Step 3: Implement.**

```ts
// Browser-only helpers. All access is guarded so imports are SSR-safe.
const CLIENT_KEY = "avk_client_id";
const ratedKey = (recipeId: string) => `avk_rated_${recipeId}`;

function safeLocalStorage(): Storage | null {
  try {
    return typeof window !== "undefined" ? window.localStorage : null;
  } catch {
    return null;
  }
}

export function getClientId(): string {
  const ls = safeLocalStorage();
  if (!ls) return "";
  let id = ls.getItem(CLIENT_KEY);
  if (!id) {
    id = crypto.randomUUID();
    ls.setItem(CLIENT_KEY, id);
  }
  return id;
}

export function getRated(recipeId: string): number | null {
  const ls = safeLocalStorage();
  if (!ls) return null;
  const raw = ls.getItem(ratedKey(recipeId));
  if (raw == null) return null;
  const n = Number(raw);
  return Number.isInteger(n) && n >= 1 && n <= 5 ? n : null;
}

export function setRated(recipeId: string, value: number): void {
  safeLocalStorage()?.setItem(ratedKey(recipeId), String(value));
}

export function clearRated(recipeId: string): void {
  safeLocalStorage()?.removeItem(ratedKey(recipeId));
}
```

- [ ] **Step 4: Run it — expect PASS.**
Run: `cd avocado.kiss && npx vitest run lib/rating-client.test.ts`

---

## Task 5: Service-role Supabase client (`lib/supabase/service.ts`)

**Files:**
- Create: `avocado.kiss/lib/supabase/service.ts`
- Reference: `avocado.kiss/lib/supabase/server.ts` (copy its shape)

- [ ] **Step 1: Read `lib/supabase/server.ts`** to match import path/options exactly (schema `avocado_kiss`, `auth.persistSession: false`).

- [ ] **Step 2: Implement.** (No unit test — it only wires env; it is exercised via the Route Handler test with a mock.)

```ts
import "server-only";
import { createClient } from "@supabase/supabase-js";

// Service-role client — SERVER ONLY. Never import from a client component.
// The secret must be a non-public env var (never NEXT_PUBLIC_).
export function createServiceClient() {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const secret = process.env.SUPABASE_SECRET_KEY;
  if (!url || !secret) {
    throw new Error("Supabase service env not configured");
  }
  return createClient(url, secret, {
    auth: { persistSession: false },
    db: { schema: "avocado_kiss" },
  });
}
```

- [ ] **Step 3: Typecheck.**
Run: `cd avocado.kiss && npx tsc -p tsconfig.json --noEmit` → no new errors.

---

## Task 6: Rate Route Handler (unit-tested; live run gated on env)

**Files:**
- Create: `avocado.kiss/app/api/recipes/[slug]/rate/route.ts`
- Test: `avocado.kiss/app/api/recipes/[slug]/rate/route.test.ts`

- [ ] **Step 1: Read the Route Handlers guide.**
Run: `ls avocado.kiss/node_modules/next/dist/docs/ && find avocado.kiss/node_modules/next/dist/docs -iname '*route*' -o -iname '*api*' | head`
Read the relevant file(s). Confirm `params` is a `Promise` and the export shape for `POST`.

- [ ] **Step 2: Write the failing test.** Mock the Turnstile verify (global `fetch`) and the service client.

```ts
import { describe, it, expect, beforeEach, vi } from "vitest";

const rpc = vi.fn();
vi.mock("@/lib/supabase/service", () => ({
  createServiceClient: () => ({ rpc }),
}));

import { POST } from "./route";

function req(body: unknown) {
  return new Request("http://localhost/api/recipes/x/rate", {
    method: "POST",
    body: JSON.stringify(body),
    headers: { "content-type": "application/json" },
  });
}
const ctx = { params: Promise.resolve({ slug: "orange-sherbet" }) };

beforeEach(() => {
  rpc.mockReset();
  vi.stubEnv("TURNSTILE_SECRET_KEY", "secret");
  vi.stubGlobal(
    "fetch",
    vi.fn(async () => new Response(JSON.stringify({ success: true }))),
  );
});

describe("POST /api/recipes/[slug]/rate", () => {
  it("400 on out-of-range value", async () => {
    const res = await POST(req({ value: 9, token: "t", clientId: "c" }), ctx);
    expect(res.status).toBe(400);
    expect(rpc).not.toHaveBeenCalled();
  });

  it("403 when Turnstile verification fails", async () => {
    (fetch as unknown as ReturnType<typeof vi.fn>).mockResolvedValueOnce(
      new Response(JSON.stringify({ success: false })),
    );
    const res = await POST(req({ value: 4, token: "bad", clientId: "c" }), ctx);
    expect(res.status).toBe(403);
    expect(rpc).not.toHaveBeenCalled();
  });

  it("200 returns DB-computed display values", async () => {
    rpc.mockResolvedValueOnce({
      data: [{ display_avg: 4.3, display_count: 3 }],
      error: null,
    });
    const res = await POST(req({ value: 5, token: "t", clientId: "c" }), ctx);
    expect(res.status).toBe(200);
    await expect(res.json()).resolves.toEqual({ displayAvg: 4.3, displayCount: 3 });
    expect(rpc).toHaveBeenCalledWith("rate_recipe", {
      p_slug: "orange-sherbet",
      p_value: 5,
      p_client_id: "c",
    });
  });

  it("404 when RPC reports recipe not found", async () => {
    rpc.mockResolvedValueOnce({ data: null, error: { message: "recipe not found" } });
    const res = await POST(req({ value: 4, token: "t", clientId: "c" }), ctx);
    expect(res.status).toBe(404);
  });
});
```

- [ ] **Step 3: Run it — expect FAIL** (route not found).
Run: `cd avocado.kiss && npx vitest run "app/api/recipes/[slug]/rate/route.test.ts"`

- [ ] **Step 4: Implement.** (Adjust the signature to whatever the Next docs from Step 1 specify; this is the Next 16 Promise-params shape.)

```ts
import { NextResponse } from "next/server";
import { createServiceClient } from "@/lib/supabase/service";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

const VERIFY_URL = "https://challenges.cloudflare.com/turnstile/v0/siteverify";

async function verifyTurnstile(token: string, ip: string | null): Promise<boolean> {
  const secret = process.env.TURNSTILE_SECRET_KEY;
  if (!secret) return false;
  const form = new URLSearchParams({ secret, response: token });
  if (ip) form.set("remoteip", ip);
  const res = await fetch(VERIFY_URL, { method: "POST", body: form });
  const data = (await res.json()) as { success?: boolean };
  return data.success === true;
}

export async function POST(
  request: Request,
  { params }: { params: Promise<{ slug: string }> },
) {
  const { slug } = await params;
  let body: { value?: unknown; token?: unknown; clientId?: unknown };
  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ error: "bad request" }, { status: 400 });
  }

  const value = Number(body.value);
  const token = typeof body.token === "string" ? body.token : "";
  const clientId = typeof body.clientId === "string" ? body.clientId : null;

  if (!Number.isInteger(value) || value < 1 || value > 5) {
    return NextResponse.json({ error: "value out of range" }, { status: 400 });
  }
  if (!token) {
    return NextResponse.json({ error: "missing token" }, { status: 403 });
  }

  const ip = request.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ?? null;
  if (!(await verifyTurnstile(token, ip))) {
    return NextResponse.json({ error: "challenge failed" }, { status: 403 });
  }

  const supabase = createServiceClient();
  const { data, error } = await supabase.rpc("rate_recipe", {
    p_slug: slug,
    p_value: value,
    p_client_id: clientId,
  });

  if (error) {
    const notFound = /not found/i.test(error.message ?? "");
    return NextResponse.json(
      { error: "could not record rating" },
      { status: notFound ? 404 : 500 },
    );
  }

  const row = Array.isArray(data) ? data[0] : data;
  return NextResponse.json({
    displayAvg: Number(row.display_avg),
    displayCount: Number(row.display_count),
  });
}
```

- [ ] **Step 5: Run it — expect PASS.**
Run: `cd avocado.kiss && npx vitest run "app/api/recipes/[slug]/rate/route.test.ts"`

---

## Task 7: StarIcon

**Files:**
- Create: `avocado.kiss/components/icons/StarIcon.tsx`
- Reference: an existing icon, e.g. `components/icons/ClockIcon.tsx` (match prop signature `{ className?: string }`).

- [ ] **Step 1: Read `components/icons/ClockIcon.tsx`** to match the exact prop/style convention.

- [ ] **Step 2: Implement** (single filled path; fill controlled by `currentColor` so CSS can render empty vs filled via color/clip).

```tsx
export default function StarIcon({ className }: { className?: string }) {
  return (
    <svg
      className={className}
      viewBox="0 0 24 24"
      fill="currentColor"
      aria-hidden="true"
      focusable="false"
    >
      <path d="M12 17.3l-6.16 3.7 1.64-7.03L2 9.24l7.19-.61L12 2l2.81 6.63 7.19.61-5.48 4.73 1.64 7.03z" />
    </svg>
  );
}
```

- [ ] **Step 3: Typecheck** — part of Task 8's test run.

---

## Task 8: RatingBadge (RSC)

**Files:**
- Create: `avocado.kiss/components/RatingBadge.tsx`, `avocado.kiss/components/RatingBadge.module.css`
- Test: `avocado.kiss/components/RatingBadge.test.tsx`

- [ ] **Step 1: Write the failing test.**

```tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import RatingBadge from "./RatingBadge";

describe("RatingBadge", () => {
  it("shows avg, plural count and an a11y label", () => {
    render(<RatingBadge avg={4.3} count={3} />);
    expect(screen.getByText("4.3")).toBeInTheDocument();
    expect(screen.getByText("3 Ratings")).toBeInTheDocument();
    expect(
      screen.getByText("Rated 4.3 out of 5, 3 Ratings"),
    ).toBeInTheDocument();
  });
  it("uses singular for one rating", () => {
    render(<RatingBadge avg={5} count={1} />);
    expect(screen.getByText("1 Rating")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL.**
Run: `cd avocado.kiss && npx vitest run components/RatingBadge.test.tsx`

- [ ] **Step 3: Implement component.** Fractional fill: a full-color star row clipped to `avg/5` width over a muted row.

```tsx
import StarIcon from "@/components/icons/StarIcon";
import { formatRatingCount, ratingLabel } from "@/lib/rating";
import styles from "./RatingBadge.module.css";

export default function RatingBadge({ avg, count }: { avg: number; count: number }) {
  const pct = Math.max(0, Math.min(100, (avg / 5) * 100));
  return (
    <div className={styles.badge}>
      <span className={styles.value}>{avg.toFixed(1)}</span>
      <span className={styles.stars} aria-hidden="true">
        <span className={styles.starsEmpty}>
          {[0, 1, 2, 3, 4].map((i) => (
            <StarIcon key={i} className={styles.star} />
          ))}
        </span>
        <span className={styles.starsFill} style={{ width: `${pct}%` }}>
          {[0, 1, 2, 3, 4].map((i) => (
            <StarIcon key={i} className={styles.star} />
          ))}
        </span>
      </span>
      <span className={styles.count}>{formatRatingCount(count)}</span>
      <span className={styles.srOnly}>{ratingLabel(avg, count)}</span>
    </div>
  );
}
```

- [ ] **Step 4: Implement CSS** (tokens only; add any missing token to `globals.css` — do not hardcode). Reference existing `.meta` styling in `app/recipes/[slug]/page.module.css` for size/color tone.

```css
.badge { display: inline-flex; align-items: center; gap: 0.5rem; }
.value { font-weight: 600; color: var(--color-text); }
.stars { position: relative; display: inline-block; line-height: 0; }
.starsEmpty, .starsFill { display: inline-flex; }
.starsEmpty { color: var(--color-border, #d9d4cc); }
.starsFill {
  position: absolute; inset: 0; overflow: hidden;
  color: var(--color-accent);
}
.star { width: 1.1rem; height: 1.1rem; flex: 0 0 auto; }
.count { color: var(--color-muted); font-size: 0.95rem; }
.srOnly {
  position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; border: 0;
}
```

> Confirm the actual token names in `app/globals.css` (`--color-text`, `--color-muted`, `--color-accent`, border) and use the real ones; add a token if a needed value is missing.

- [ ] **Step 5: Run it — expect PASS.**
Run: `cd avocado.kiss && npx vitest run components/RatingBadge.test.tsx`

---

## Task 9: RatingSection (`'use client'`)

**Files:**
- Create: `avocado.kiss/components/RatingSection.tsx`, `avocado.kiss/components/RatingSection.module.css`
- Test: `avocado.kiss/components/RatingSection.test.tsx`

- [ ] **Step 1: Write the failing test.** Mock the localStorage helper, Turnstile hook, and `fetch`.

```tsx
import { describe, it, expect, beforeEach, vi } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";

const setRated = vi.fn();
let ratedValue: number | null = null;
vi.mock("@/lib/rating-client", () => ({
  getClientId: () => "client-1",
  getRated: () => ratedValue,
  setRated: (...a: unknown[]) => setRated(...a),
  clearRated: vi.fn(),
}));
// Turnstile: resolve a token immediately.
vi.mock("./Turnstile", () => ({
  default: ({ onToken }: { onToken: (t: string) => void }) => {
    onToken("tok");
    return null;
  },
}));

import RatingSection from "./RatingSection";

beforeEach(() => {
  ratedValue = null;
  setRated.mockReset();
  vi.stubGlobal(
    "fetch",
    vi.fn(async () =>
      new Response(JSON.stringify({ displayAvg: 4.4, displayCount: 4 })),
    ),
  );
});

describe("RatingSection", () => {
  it("submits a rating and shows the thank-you state", async () => {
    render(<RatingSection recipeId="r1" slug="orange-sherbet" initialAvg={4.3} initialCount={3} />);
    fireEvent.click(screen.getByRole("radio", { name: /4 stars/i }));
    await waitFor(() =>
      expect(screen.getByText(/You rated 4/i)).toBeInTheDocument(),
    );
    expect(setRated).toHaveBeenCalledWith("r1", 4);
    expect(fetch).toHaveBeenCalledWith(
      "/api/recipes/orange-sherbet/rate",
      expect.objectContaining({ method: "POST" }),
    );
    expect(screen.getByText("4 Ratings")).toBeInTheDocument();
  });

  it("renders the already-rated state without a request", () => {
    ratedValue = 5;
    render(<RatingSection recipeId="r1" slug="orange-sherbet" initialAvg={4.3} initialCount={3} />);
    expect(screen.getByText(/You rated 5/i)).toBeInTheDocument();
    expect(fetch).not.toHaveBeenCalled();
  });

  it("shows an error and does not persist on a failed response", async () => {
    (fetch as unknown as ReturnType<typeof vi.fn>).mockResolvedValueOnce(
      new Response(JSON.stringify({ error: "challenge failed" }), { status: 403 }),
    );
    render(<RatingSection recipeId="r1" slug="orange-sherbet" initialAvg={4.3} initialCount={3} />);
    fireEvent.click(screen.getByRole("radio", { name: /3 stars/i }));
    await waitFor(() =>
      expect(screen.getByText(/couldn.t save/i)).toBeInTheDocument(),
    );
    expect(setRated).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it — expect FAIL.**
Run: `cd avocado.kiss && npx vitest run components/RatingSection.test.tsx`

- [ ] **Step 3: Implement the Turnstile wrapper** `components/Turnstile.tsx` (`'use client'`). Loads the CF script, renders the widget, calls `onToken` with the response token. Keep it minimal and typed; site key from `process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY`. It must accept `{ onToken: (t: string) => void }`. (Render nothing visible for the invisible/managed widget beyond the CF container.)

```tsx
"use client";
import { useEffect, useRef } from "react";

declare global {
  interface Window {
    turnstile?: {
      render: (el: HTMLElement, opts: { sitekey: string; callback: (t: string) => void }) => void;
    };
  }
}

export default function Turnstile({ onToken }: { onToken: (t: string) => void }) {
  const ref = useRef<HTMLDivElement>(null);
  const sitekey = process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY;
  useEffect(() => {
    if (!sitekey || !ref.current) return;
    const el = ref.current;
    const src = "https://challenges.cloudflare.com/turnstile/v0/api.js";
    function mount() {
      window.turnstile?.render(el, { sitekey: sitekey!, callback: onToken });
    }
    if (window.turnstile) {
      mount();
    } else {
      const s = document.createElement("script");
      s.src = src;
      s.async = true;
      s.onload = mount;
      document.head.appendChild(s);
    }
  }, [sitekey, onToken]);
  return <div ref={ref} />;
}
```

- [ ] **Step 4: Implement `RatingSection.tsx`.** Props: `{ recipeId: string; slug: string; initialAvg: number; initialCount: number }`. States: idle / submitting / done / already-rated / error. Guard localStorage read behind a mounted flag to avoid hydration mismatch.

```tsx
"use client";
import { useEffect, useState } from "react";
import StarIcon from "@/components/icons/StarIcon";
import Turnstile from "./Turnstile";
import { getClientId, getRated, setRated } from "@/lib/rating-client";
import { formatRatingCount } from "@/lib/rating";
import styles from "./RatingSection.module.css";

type Props = { recipeId: string; slug: string; initialAvg: number; initialCount: number };

export default function RatingSection({ recipeId, slug, initialAvg, initialCount }: Props) {
  const [mounted, setMounted] = useState(false);
  const [rated, setRatedState] = useState<number | null>(null);
  const [hover, setHover] = useState(0);
  const [status, setStatus] = useState<"idle" | "submitting" | "error">("idle");
  const [avg, setAvg] = useState(initialAvg);
  const [count, setCount] = useState(initialCount);
  const [token, setToken] = useState<string | null>(null);

  useEffect(() => {
    setMounted(true);
    setRatedState(getRated(recipeId));
  }, [recipeId]);

  async function submit(value: number) {
    setStatus("submitting");
    try {
      const res = await fetch(`/api/recipes/${slug}/rate`, {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({ value, token, clientId: getClientId() }),
      });
      if (!res.ok) throw new Error(String(res.status));
      const data = (await res.json()) as { displayAvg: number; displayCount: number };
      setAvg(data.displayAvg);
      setCount(data.displayCount);
      setRated(recipeId, value);
      setRatedState(value);
      setStatus("idle");
    } catch {
      setStatus("error");
    }
  }

  if (mounted && rated != null) {
    return (
      <section className={styles.section} aria-label="Your rating">
        <p className={styles.thanks}>Thanks! You rated {rated}★</p>
        <p className={styles.aggregate}>
          {avg.toFixed(1)} · {formatRatingCount(count)}
        </p>
      </section>
    );
  }

  return (
    <section className={styles.section} aria-label="Rate this recipe">
      <h2 className={styles.heading}>Rate this recipe</h2>
      <div className={styles.picker} role="radiogroup" aria-label="Rate this recipe">
        {[1, 2, 3, 4, 5].map((v) => (
          <button
            key={v}
            type="button"
            role="radio"
            aria-checked={false}
            aria-label={`${v} stars`}
            disabled={status === "submitting"}
            className={styles.starBtn}
            data-active={v <= hover}
            onMouseEnter={() => setHover(v)}
            onMouseLeave={() => setHover(0)}
            onFocus={() => setHover(v)}
            onClick={() => submit(v)}
          >
            <StarIcon className={styles.star} />
          </button>
        ))}
      </div>
      {status === "error" && (
        <p className={styles.error} role="alert">
          Sorry, we couldn&apos;t save your rating. Please try again.
        </p>
      )}
      <Turnstile onToken={setToken} />
    </section>
  );
}
```

- [ ] **Step 5: Implement `RatingSection.module.css`** (tokens only; match page tone — spacing like `.content` sections). Minimal:

```css
.section {
  border-top: 1px solid var(--color-border, #e6e1d8);
  margin-top: 3rem;
  padding-top: 2rem;
  text-align: center;
}
.heading { font-size: 1.25rem; margin-bottom: 1rem; }
.picker { display: inline-flex; gap: 0.25rem; }
.starBtn {
  background: none; border: 0; cursor: pointer; padding: 0.15rem;
  color: var(--color-border, #d9d4cc); line-height: 0;
}
.starBtn[data-active="true"] { color: var(--color-accent); }
.starBtn:disabled { cursor: default; }
.star { width: 2rem; height: 2rem; }
.thanks { font-weight: 600; }
.aggregate { color: var(--color-muted); }
.error { color: var(--color-danger, #b3261e); margin-top: 0.75rem; }
```

- [ ] **Step 6: Run it — expect PASS.**
Run: `cd avocado.kiss && npx vitest run components/RatingSection.test.tsx`

> Note: in the test the Turnstile mock calls `onToken("tok")` on render, so `token` is set before the click. Keep the mock in sync with the wrapper's prop name (`onToken`).

---

## Task 10: Wire into the recipe page

**Files:**
- Modify: `avocado.kiss/app/recipes/[slug]/page.tsx`

- [ ] **Step 1: Import the components** at the top:

```ts
import RatingBadge from "@/components/RatingBadge";
import RatingSection from "@/components/RatingSection";
```

- [ ] **Step 2: Render the badge** inside `<header>`, immediately after the `<h1>` (page.tsx:50), before the excerpt:

```tsx
<RatingBadge avg={recipe.display_avg} count={recipe.display_count} />
```

- [ ] **Step 3: Render the section** just before `</article>` closes (after the `.content` div, page.tsx:115):

```tsx
<RatingSection
  recipeId={recipe.id}
  slug={recipe.slug}
  initialAvg={recipe.display_avg}
  initialCount={recipe.display_count}
/>
```

- [ ] **Step 4: Add aggregateRating JSON-LD** (only when `display_count > 0`). Inside the returned JSX, e.g. right after `<main>` opens:

```tsx
{recipe.display_count > 0 && (
  <script
    type="application/ld+json"
    dangerouslySetInnerHTML={{
      __html: JSON.stringify({
        "@context": "https://schema.org",
        "@type": "Recipe",
        name: recipe.title,
        aggregateRating: {
          "@type": "AggregateRating",
          ratingValue: recipe.display_avg,
          ratingCount: recipe.display_count,
        },
      }),
    }}
  />
)}
```

- [ ] **Step 5: Typecheck + build the page.**
Run: `cd avocado.kiss && npx tsc -p tsconfig.json --noEmit`
Expected: no errors. (Full `npm run build` in Task 12.)

---

## Task 11: Env, policy, and docs

**Files:**
- Modify: `avocado.kiss/.env.example`, `avocado.kiss/AGENTS.md`, `platform-docs/database/schema.md`, `platform-docs/sites/avocado-kiss.md`

- [ ] **Step 1: `.env.example`** — append:

```
# Recipe ratings (server-only secrets — never NEXT_PUBLIC_, never shipped to browser)
NEXT_PUBLIC_TURNSTILE_SITE_KEY=1x00000000000000000000AA   # Cloudflare Turnstile site key (public)
TURNSTILE_SECRET_KEY=                                     # Turnstile secret (server-only)
SUPABASE_SECRET_KEY=                                      # Supabase service-role key (server-only)
```

- [ ] **Step 2: `AGENTS.md`** — change the env line under "When working on this project" to note the exception:

Replace:
`- Env: NEXT_PUBLIC_SUPABASE_URL + NEXT_PUBLIC_SUPABASE_ANON_KEY (publishable only — never sb_secret_…)`
With:
`- Env: NEXT_PUBLIC_SUPABASE_URL + NEXT_PUBLIC_SUPABASE_ANON_KEY (publishable) for all reads. Server-only exception: the rate Route Handler uses SUPABASE_SECRET_KEY + TURNSTILE_SECRET_KEY — non-NEXT_PUBLIC_ env, server-only, never shipped to the browser.`

- [ ] **Step 3: `platform-docs/database/schema.md`** — document the new `recipes` columns (real vs seed vs display), the `recipe_ratings` table, and the `rate_recipe` RPC (service_role only). Match the file's existing section style.

- [ ] **Step 4: `platform-docs/sites/avocado-kiss.md`** — add a "Recipe ratings" subsection: badge + end-of-recipe picker, floor 4.1, seed baseline, Turnstile-guarded Route Handler, localStorage dedup. Link the spec.

- [ ] **Step 5: No sitemap change** — confirm: the rate endpoint is an API route (no page), no new public page route, so `app/sitemap.ts` is untouched. State this in the task's notes.

---

## Task 12: Full verification

- [ ] **Step 1: Run the whole unit/component suite.**
Run: `cd avocado.kiss && npm test`
Expected: all pass, including the new rating tests.

- [ ] **Step 2: Lint.**
Run: `cd avocado.kiss && npm run lint`
Expected: clean.

- [ ] **Step 3: Build.**
Run: `cd avocado.kiss && npm run build`
Expected: TS + prerender succeed. Note: with the migration applied (Task 1) and `select("*")`, recipe pages read the new columns. If the build fetches recipes and the columns are missing, the build reveals it here.

- [ ] **Step 4: Live write-path check (user-run, needs env).** Document for the user: set `NEXT_PUBLIC_TURNSTILE_SITE_KEY`, `TURNSTILE_SECRET_KEY`, `SUPABASE_SECRET_KEY` (+ existing Supabase envs), run `npm run dev`, open a recipe, submit a rating, confirm: thank-you state, count increments, reload shows "already rated", and a row appears in `avocado_kiss.recipe_ratings`. Cloudflare provides always-pass test keys for local dev.

- [ ] **Step 5: Report.** Summarize: files created/changed, test results (N passed), what still needs the user (apply migration if not yet, provision env keys, live check), and any spec/code mismatches found.

---

## Self-review notes

- **Spec coverage:** math/floor/seed → Task 1 (+2,3); votes table → Task 1; write path + Turnstile → Tasks 5,6; localStorage dedup → Tasks 4,9; badge → Tasks 7,8,10; section → Task 9,10; JSON-LD → Task 10; admin compatibility → Task 1 columns (no code); env policy → Task 11; docs → Task 11; testing → every task + Task 12. E2E is proposed (spec §10) — omitted from tasks intentionally (build only on request).
- **Type consistency:** `Recipe.display_avg`/`display_count` (Task 2) used identically in Tasks 8/9/10; RPC returns `display_avg`/`display_count` (Task 1) mapped to `displayAvg`/`displayCount` in the Route Handler (Task 6) and consumed by that shape in RatingSection (Task 9). `rating-client` exports `getClientId`/`getRated`/`setRated`/`clearRated` used consistently.
- **Gates:** Task 1-apply and live runs are explicitly gated; no unattended live-DB or secret-dependent action.
