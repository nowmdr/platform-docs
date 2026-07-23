# About page — design (cozycorner + web.admin)

> 2026-07-23 · Status: approved, in implementation · Scope: cozycorner `/about`
> page + editable content in web.admin Pages section + DB table.

## Goal

Build a real `/about` page for cozycorner (today it's a `PagePlaceholder`) using the
structure of the reference screenshot (eyebrow + heading + intro; image + story block;
three value cards; newsletter band) but rendered in **cozycorner's own design tokens**
(indigo/orange/white-grey, SF Pro, `--radius-lg`, `--shadow-card`), with **all page
content editable from the admin**. Sections are static — fixed set, no add/remove/reorder.

Copy is home-goods / English (cozycorner is a cozy home-goods catalog, `<html lang="en">`),
derived from the reference structure — not the plant-site wording or cream/green palette.

## Sections (static)

1. **Intro** — eyebrow + heading + lead paragraph (left-aligned text block).
2. **Story** — image (left) + heading + two paragraphs (right); stacks on mobile.
3. **Values** — three cards, each: emoji icon + title + description (3-col → 1-col).

The reference's bottom "Grow with us" newsletter band = cozycorner's **existing global
footer** `NewsletterForm` (already on every page, text editable via `footer_settings`).
It is **not** re-added as an About section.

## Data model — `cozycorner.about_content` (new singleton table)

Rationale: matches the "static, non-repeating" requirement and the endorsed
`footer_settings`/`header_settings` singleton pattern (schema.md §5: new params = new
columns). One row; the site reads the first row; the admin edits it. All text `nullable`;
empty → code fallback (safe-fallback pattern used across the site).

Columns: `id, created_at, updated_at, eyebrow, heading, intro, story_heading,
story_body_1, story_body_2, story_image_path, card1_icon, card1_title, card1_text,
card2_icon, card2_title, card2_text, card3_icon, card3_title, card3_text`.

- `story_image_path` — image-path contract (schema.md §4): flat bucket key or external URL.
- RLS: public read + admin write (`is_admin()`); `updated_at` trigger; anon/authenticated
  grants — copied from `header_settings` (0024).
- Migration `0026_about_content.sql` in `cozycorner/supabase/migrations/` (source of truth),
  applied to `base-one` via MCP. Seeds one row with the home-goods copy so both site and
  admin have content immediately.

## cozycorner site

- `lib/content.ts`: `AboutContent` type + `fetchAboutContent(client)` (reads first row).
- `components/about/`: `AboutIntro`, `AboutStory`, `AboutValues` — one file + `.module.css`
  each, RSC, tokens only, wrapped in `Reveal`. Story image via `ImageWithFallback` +
  `resolveProductImage`. Each renders a code fallback string when its field is null/empty.
  The intro `h1` uses the rounded display font (`--font-rounded` + `--fw-heavy`, like
  `Hero .title`) — per user request; design-system.md updated so rounded is no longer
  hero-only.
- `app/about/page.tsx`: thin — `fetchAboutContent` + `fetchPageContent("about")` (SEO),
  compose the three sections, `revalidate = 60`, `generateMetadata` via `pageMetadata`.
  Replaces `PagePlaceholder`.

## web.admin (edit on the existing About page editor)

One coherent screen / one Save, following the `HeroSectionsEditor` embed pattern:

- `lib/about.ts`: `getAboutContent(site)` / `updateAboutContent(site, id, input)`
  (singleton; empty → `null` via `orNull`). Mirrors `lib/header.ts`.
- `lib/pages.ts`: `getPage` also returns `about: AboutContent | null` (fetched only for
  slug `about`); `updatePage` also updates the `about_content` row when `about` is present;
  fresh re-fetch re-syncs the form. Query key stays `['pages', site.slug, id]`.
- `features/pages/AboutContentEditor.tsx`: rendered only when `slug === 'about'` (like the
  `hasBody` gate). Text `Input`/`Textarea` per field; small emoji `Input` per card; story
  image via `ImagePreviewPicker` + `ImagePickerDialog`. Registers its fields into the page
  form under an `about` key; extends `pageSchema`, `toFormValues`, `toInput`.
- `lib/media.ts` `USAGE_SOURCES`: add
  `{ table: "about_content", label: "About", name: "heading", columns: ["story_image_path"] }`.

## Out of scope

Dynamic/repeating sections, editing card count, a dedicated on-page newsletter section,
`pages.meta`, adding About content to any other site.

## Docs to update after implementation

`database/schema.md` (§5 table + §6 migration 0026), `admin-panel/pages.md` (About editor),
`sites/cozycorner/design-system.md` (new About components + any new token).

## Testing (workspace protocol — delegated to test-engineer, per project)

- cozycorner: component tests for `AboutIntro`/`AboutStory`/`AboutValues` (RTL, next/image
  mock) + `fetchAboutContent`. E2E only on request (needs live Supabase).
- web.admin: `AboutContentEditor` component + `about.ts` empty→null mapping.
