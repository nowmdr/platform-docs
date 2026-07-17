---
name: seo-posts-cozycorner
description: Create SEO posts for the CozyCorner site via the Supabase connector. Use when asked to create, draft, or publish an SEO post. Contains the exact data contract for inserting into posts/post_text_sections/post_product_sections and a mandatory SEO checklist (seo_title and seo_description are always required).
---

# Creating SEO Posts (CozyCorner)

Instruction for a claude.ai agent with the Supabase connector. Typical request:
"Create a new SEO post about X — here is a similar post as a reference."

## 1. What SEO posts are and why they exist

SEO posts are blog-like articles targeting **long-tail search queries**
(programmatic SEO). They are stored next to regular blog posts but marked with
`post_type = 'seo'`:

- They do NOT appear in the blog feed or site navigation.
- They ARE publicly readable at `/blog/<slug>` and listed in the sitemap —
  that is how search engines find and index them.
- A visitor who lands on one from a search result sees a normal, complete
  article. **Never suggest redirects, User-Agent detection, or noindex for
  these pages** — serving bots and humans different content is cloaking, which
  can get the entire domain penalized by Google.

Because Google's helpful-content system evaluates sites **holistically**, one
batch of thin, low-value SEO posts can drag down rankings of the whole domain,
including the real blog and product pages. Quality rules below are therefore
hard requirements, not suggestions.

## 2. Data contract (follow exactly)

All tables live in the `cozycorner` schema. A post = one row in `posts` + its
content sections in two tables.

### 2.1 `posts` — insert one row

| Column | Value |
|---|---|
| `title` | Post title (also used as H1 and slug source) |
| `post_type` | Always `'seo'` |
| `excerpt` | 1–2 sentence summary. Always fill it (feeds the description fallback) |
| `is_published` | `false` by default — a human reviews and publishes. Set `true` only if explicitly told to publish immediately |
| `published_at` | ISO timestamp (now, unless told otherwise) |
| `cover_image_path` | Flat file key from the `media` table (`path` column) or external URL; may be `null` |
| `seo_title` | REQUIRED — see §3 |
| `seo_description` | REQUIRED — see §3 |
| `folder_id` | `null` (Unsorted) unless asked to file it into a folder (`admin_folders`, `section='seo_posts'`) |

Rules:
- **Do NOT send `slug`** — the database trigger generates it from `title`.
- **Do NOT touch the legacy `content` column.**
- **Do NOT set `created_at`/`updated_at`** — handled by the database.

### 2.2 Content sections — the post body

Two tables, merged and ordered by `position` when the site renders the post.
**`position` is a single sequence across BOTH tables**: number the modules
0, 1, 2… in reading order, regardless of which table each row goes to.

`post_text_sections` (markdown text module):
- `post_id`, `position`, `is_published` (`true`), `heading` (H2 for the
  section, nullable), `body` (markdown: paragraphs, `- ` lists, `1. ` lists,
  `**bold**`, `*italic*`, links).

`post_product_sections` (product grid module):
- `post_id`, `position`, `is_published` (`true`), `heading` (nullable),
  `columns` (`2` or `3`), `product_ids` (uuid[] — **real ids from the
  `products` table**; query it first, never invent ids; no duplicates).

### 2.3 Workflow

1. If given a reference post, read its row and sections first and mirror its
   structure and tone.
2. Check existing posts (`select title, slug from posts`) — do not create a
   post duplicating an existing topic; extend or differentiate instead.
3. Insert the `posts` row, get its `id`, then insert the sections.
4. Report back: title, generated slug, section count, and that the post is a
   draft awaiting review.

## 3. SEO checklist (apply to EVERY post)

**Meta tags — mandatory, never skip:**
- **`seo_title`: always set it. ≤ 60 characters**, unique across the site,
  primary keyword phrase at the front, written to match search intent (not
  clickbait — misleading titles raise bounce rate, which Google punishes).
- **`seo_description`: always set it. 120–155 characters**, unique, front-load
  the key information in the first ~110 characters (mobile truncates early),
  include the target query naturally, end with a reason to click.
- These are "the tags" for this system — posts have no separate keyword/tag
  field, and empty meta fields fall back to title/excerpt, which is almost
  always weaker. Filling both explicitly is a hard requirement.

**Content:**
- One post = **one search intent**, phrased as a long-tail query someone
  actually types. Answer that query directly in the first paragraph — AI
  search and featured snippets extract from the opening section.
- Structure with `heading` on each text section (rendered as H2): scannable,
  logical order, tight sections.
- Substance over volume: each post must contain genuinely useful, specific,
  unique information — not a template with swapped words. Thin or
  near-duplicate posts are worse than no posts (they harm the whole domain).
- Write for people (E-E-A-T): concrete details, honest claims, no keyword
  stuffing; keywords appear naturally.
- **Internal links**: add at least one product module with genuinely relevant
  products, and link to related existing blog posts in the markdown where it
  helps the reader — internal linking builds topical authority.
- Fill `excerpt` and, when a relevant image exists in `media`, set
  `cover_image_path`.

**Publication:**
- Create as draft (`is_published = false`). A human reviews in the admin
  ("SEO Posts" section) and flips the switch. Publish immediately only on an
  explicit instruction.
