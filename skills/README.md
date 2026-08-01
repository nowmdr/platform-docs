# Skills store (Supabase-connector content operations)

Canonical, version-controlled home for the **content-ops skills** — operating
manuals for the claude.ai agent that edits each site's content through the
**Supabase connector**. They live here (visible, in git) instead of only in the
hidden `.claude/skills/` loader dir, so they can be reviewed, shared, and loaded
into Claude easily.

## Skills in this store

| Folder | Skill (frontmatter `name`) | Site / schema | Purpose |
|---|---|---|---|
| `cozycorner-content-ops/` | `cozycorner-content-ops` | CozyCorner (`cozycorner`) | Add/edit products, categories, brands, blog & SEO posts + sections; read catalog |
| `avocado-content-ops/` | `avocado-content-ops` | Avocado Kiss (`avocado_kiss`) | Add/edit recipes, categories, tags, posts; curate the home page (`home_slots` / Editor's Picks); footer & page SEO |

Folder name = loader symlink name = invocation name = frontmatter `name` — all
four match per skill.

Both target the same Supabase project **base-one** (`zwrkphynupdubevzwdzy`) but
**different schemas** — each skill is scoped to one and refuses to cross the
boundary. Each SKILL.md's golden rule is: **the live DB is the source of truth**;
inspect the real tables via the connector before writing.

## How they load into Claude

**Claude Code (this workspace)** — auto-discovered. The workspace loader dir
`../.claude/skills/` holds **symlinks** pointing here:

```
.claude/skills/cozycorner-content-ops  -> ../../platform-docs/skills/cozycorner-content-ops
.claude/skills/avocado-content-ops     -> ../../platform-docs/skills/avocado-content-ops
```

So the files live in git under `platform-docs/`, and Claude Code still finds
them under `.claude/skills/`. Edit the files here; the symlink reflects it. A new
session picks up added/renamed skills.

**claude.ai (the connector agent that actually runs these)** — open the relevant
`SKILL.md` here and load its contents as the agent's skill/instructions for the
session where the Supabase connector is attached to **base-one**.

## Adding a new content-ops skill

1. Create `platform-docs/skills/<name>/SKILL.md` (copy an existing one as a
   template; keep the golden-rule + guardrails structure).
2. Symlink it into the loader dir so Claude Code discovers it:
   `ln -s ../../platform-docs/skills/<name> ../.claude/skills/<name>`
   (run from `platform-docs/skills/`, or adjust the relative path).
3. Add a row to the table above.

> The `testing` skill is **not** here — it stays at `.claude/skills/testing/`
> and is documented in `../methodology/testing.md`.
